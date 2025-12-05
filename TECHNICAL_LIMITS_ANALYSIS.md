# CloudFlare-ImgBed 技术限制分析 - 深度技术文档

## 快速参考表

### 核心限制总结

```
┌─────────────────────────────────────────────────────────────────┐
│                     CloudFlare-ImgBed 限制矩阵                    │
├─────────────────┬──────────────┬──────────────┬──────────────────┤
│ 参数            │ D1 (SQLite)  │ KV           │ Telegram         │
├─────────────────┼──────────────┼──────────────┼──────────────────┤
│ 单元值限制      │ ~1-10 MB*    │ 25 MB        │ 50 MB/文件       │
│ 建议用于        │ 小文件元数据 │ 中大型元数据 │ 实际文件存储     │
│ 实际限制        │ 行大小       │ 值大小       │ 分片上传大小     │
│ 最安全使用      │ < 100 KB     │ < 1 MB       │ 20 MB/分片       │
└─────────────────┴──────────────┴──────────────┴──────────────────┘
* D1 限制由 Cloudflare 定义，具体值未文档化
```

---

## 1. D1（SQLite）的深度分析

### 1.1 D1 的架构限制

#### SQLite 的基础限制（D1 继承）

```
SQLite 默认配置：
├─ Page Size:           4096 bytes
├─ Max Page Number:     1073741823 (2^30 - 1)
├─ Max DB Size:         4398046511104 bytes (~4 TB)
├─ Max Record Size:     理论无限，实际 ~1 GB per blob
├─ Max SQL Length:      1 million characters
└─ Pragmas:
   ├─ PRAGMA cell_size_limit
   ├─ PRAGMA max_page_count
   └─ PRAGMA page_size
```

#### Cloudflare D1 的实际限制

D1 作为分布式数据库，实施了自己的限制：

```javascript
// 推断的 D1 限制（基于最佳实践和错误报告）
D1_LIMITS = {
    MAX_QUERY_SIZE:      1_000_000,      // 1 MB SQL query
    MAX_BIND_PARAMS:     100,            // 参数数量
    MAX_ROW_SIZE:        ???,            // 未文档化，推测 1-10 MB
    MAX_TEXT_COLUMN:     ???,            // 推测 1-5 MB
    MAX_TRANSACTION_SIZE: ???,           // 推测 10-100 MB
}
```

### 1.2 CloudFlare-ImgBed 中的 D1 使用

#### 文件表的行结构

```sql
CREATE TABLE files (
    id TEXT PRIMARY KEY,                   -- 可变大小
    value TEXT,                            -- 关键：分片信息存储在这里
    metadata TEXT NOT NULL,                -- 关键：完整的 JSON 元数据
    file_name TEXT,
    file_type TEXT,
    file_size TEXT,                        -- "123.45"
    upload_ip TEXT,
    upload_address TEXT,
    list_type TEXT,
    timestamp INTEGER,
    label TEXT,
    directory TEXT,
    channel TEXT,
    channel_name TEXT,
    tg_file_id TEXT,                       -- Telegram 文件 ID
    tg_chat_id TEXT,
    tg_bot_token TEXT,                     -- 敏感信息！
    is_chunked BOOLEAN,
    tags TEXT,                             -- JSON 数组
    created_at DATETIME,
    updated_at DATETIME
);
```

#### 行大小计算示例

对于一个 1GB 的分片文件：

```
1. 固定开销：
   ├─ SQLite 行头：      ~5 bytes
   ├─ 内部指针：        ~8 bytes per column (18 columns × 8) = 144 bytes
   └─ 小计：            ~150 bytes

2. id（PRIMARY KEY）:
   ├─ 典型值：          "file_12345678_abcdef"
   ├─ 大小：            ~25 bytes
   └─ 小计：            ~25 bytes

3. metadata（JSON）：
   ├─ 示例结构：
   │  {
   │    "FileName": "large_file.iso",
   │    "FileType": "application/octet-stream",
   │    "FileSize": "1024",
   │    "UploadIP": "203.0.113.42",
   │    "UploadAddress": "Country, City, ISP",
   │    "TimeStamp": 1234567890,
   │    "Label": "safe",
   │    "Directory": "folder/subfolder/",
   │    "Tags": ["archive", "iso"],
   │    "Channel": "TelegramNew",
   │    "ChannelName": "main",
   │    "TgFileId": "AgACAgQA...",          <- 很长！
   │    "TgChatId": "-1001234567890",
   │    "TgBotToken": "123456:ABC-DEF1...",  <- 很长！
   │    "IsChunked": true,
   │    "TotalChunks": 50
   │  }
   ├─ Telegram ID 长度： ~100-200 bytes
   ├─ Bot Token 长度：   ~45 bytes
   ├─ 总计：            ~1000-2000 bytes

4. value（分片信息 JSON）：
   ├─ 对于 50 个分片：
   │  [
   │    {
   │      "index": 0,
   │      "fileId": "AgACAgQA...",         <- ~150 bytes
   │      "size": 20971520,
   │      "fileName": "file.part000"
   │    },
   │    ... （重复 49 次）
   │  ]
   ├─ 单个分片信息：    ~200 bytes
   ├─ 50 个分片：       50 × 200 = 10,000 bytes = 10 KB
   └─ 小计：            ~10 KB

5. 其他文本列（directory, tags, file_name等）：
   ├─ file_name：       ~30 bytes
   ├─ file_type：       ~30 bytes
   ├─ directory：       ~50 bytes
   ├─ tags：            ~50 bytes（JSON）
   ├─ 其他：            ~100 bytes
   └─ 小计：            ~260 bytes

───────────────────────────────────────────
总计：
  150 + 25 + 1,500 + 10,000 + 260 = ~12 KB

这通常在 SQLite 的能力范围内，但：
- 如果元数据更复杂或 Telegram ID 更长
- 或者存储了更多的分片（理论上支持到 79,000+ 个）
- 行大小可能超过 D1 的实际限制
```

### 1.3 D1 的实际故障模式

根据代码设计，D1 在这些情况下会失败：

```javascript
// 场景 1：超大型元数据
case 1_超大元数据: {
    metadata = {
        ...基础字段,
        // 额外的大字段
        description: "非常长的描述，包含 1MB 的文本",
        custom_data: { /* 大量嵌套数据 */ },
        // ...
    };
    // → D1 可能返回: "Row too large" 或类似错误
}

// 场景 2：超大分片数量
case 2_超大分片数量: {
    // 如果 50 分片限制被移除，而有 1000 个分片
    const chunks = Array(1000).fill({
        index: i,
        fileId: "AgACAgQA...", // ~150 bytes
        size: 20_000_000,
        fileName: "file.part..."
    });
    // value = JSON.stringify(chunks) // ~150 KB
    // → D1 可能在存储或查询时失败
}

// 场景 3：密集的并发写入
case 3_并发限制: {
    // 同时写入 100 个大型记录到 D1
    // → D1 的事务或行级锁可能成为瓶颈
}
```

### 1.4 为什么代码优先使用 KV 而不是 D1

代码证据（`/functions/utils/databaseAdapter.js`，第 13-25 行）：

```javascript
export function createDatabaseAdapter(env) {
    // KV 优先检查
    if (env.img_url && typeof env.img_url.get === 'function') {
        return new KVAdapter(env.img_url);  // ← KV 优先！
    }
    // D1 作为备选
    else if (env.img_d1 && typeof env.img_d1.prepare === 'function') {
        return new D1Database(env.img_d1);
    }
    else {
        console.error('No database configured');
        return null;
    }
}
```

**原因**：
1. KV 专为高频读写优化
2. KV 的 25MB 值限制足够
3. D1 更适合结构化查询，不适合存储大型 JSON
4. KV 支持自动过期（expirationTtl）

---

## 2. KV（键值存储）的深度分析

### 2.1 KV 的官方限制

根据 Cloudflare KV 文档：

```
Cloudflare KV 规格：
┌─────────────────────────────────────────┐
│ 参数                 │ 限制               │
├─────────────────────────────────────────┤
│ 单个 key 名称       │ 512 bytes         │
│ 单个 value 值       │ 25 MB             │ ← 关键
│ 单个 metadata       │ 1 KB              │
│ 单个 key 读取       │ 无限（受速率限制）│
│ 单个 key 写入       │ 无限（受速率限制）│
│ 列表操作上限        │ 1000 keys/page    │
│ 列表游标有效期      │ 60 秒              │
│ 命名空间数          │ 10 个/账户        │
└─────────────────────────────────────────┘
```

### 2.2 CloudFlare-ImgBed 中的 KV 使用

#### KV 适配器的方法

```javascript
class KVAdapter {
    async put(key, value, options) {
        // options 结构：
        options = {
            metadata: {
                // 最多 1 KB 的元数据
                customField1: value1,
                customField2: value2,
                ...
            },
            expirationTtl: 3600, // 1小时后过期
            ...其他选项
        };
        return await this.kv.put(key, value, options);
    }
}
```

#### 不同场景的数据大小

```javascript
// 场景 1：小文件元数据
{
    key: "file_abc123",
    value: "",  // 空值
    metadata: {
        // 关键信息存储在元数据中
        fileName: "photo.jpg",
        fileSize: "2.5",
        channel: "TelegramNew",
        // ...
    }
    // 总大小：< 1 KB ✓
}

// 场景 2：大文件分片信息
{
    key: "file_xyz789",
    value: JSON.stringify([
        { index: 0, fileId: "AgAC...", size: 20971520, fileName: "file.part000" },
        { index: 1, fileId: "AgAC...", size: 20971520, fileName: "file.part001" },
        // ... 50 个分片
    ]),
    metadata: {
        fileName: "large_file.iso",
        fileSize: "1024",
        isChunked: true,
        totalChunks: 50,
        // ...
    }
    // value 大小：~10 KB
    // metadata 大小：~1 KB
    // 总大小：~11 KB ✓ 远在 25 MB 限制内
}

// 场景 3：理论上的最坏情况
{
    key: "massive_file",
    value: JSON.stringify(Array(1000).fill({
        index: i,
        fileId: "AgACAgQA...",
        size: 20971520,
        fileName: "file.part..."
    })),
    // 大小：~150-200 KB
    // 仍在 25 MB 限制内 ✓
}
```

### 2.3 KV 实际能支持的文件大小

根据当前代码架构：

```javascript
// KV 中存储的是分片信息，不是文件内容本身
const maxChunkCount = Math.floor(25 * 1024 * 1024 / 300); // 300 bytes per chunk info
// = 87,380 个分片

const maxFileSize = maxChunkCount * 20 * 1024 * 1024 / 1024 / 1024 / 1024;
// = 87,380 × 20 MB ≈ 1.7 TB

// 但代码中限制：
const maxChunksInCode = 50;
const maxFileInCode = 50 * 20 * 1024 * 1024 / 1024 / 1024;
// = 1 GB
```

**结论**：
- **理论 KV 限制**：1.7 TB（如果允许分片数）
- **代码中实际限制**：1 GB（50 分片 × 20 MB）
- **真实瓶颈**：Worker CPU 时间（分片上传需要时间）

### 2.4 KV 与 D1 的选择决策树

```
文件大小 < 1 MB
    ↓
[是否需要 SQL 查询?]
    ├─ 是 → 选择 D1
    └─ 否 → 选择 KV（更快）

1 MB < 文件大小 < 10 MB
    ↓
[是否需要 SQL 查询?]
    ├─ 是 → 选择 KV + D1（混合）
    └─ 否 → 选择 KV（元数据大小仍 < 10 KB）

文件大小 > 10 MB
    ↓
[使用分片上传?]
    ├─ 否 → 选择 KV（如果 KV 值 < 25 MB）
    └─ 是 → 选择 KV（分片信息极小）

推荐方案：
        ├─ 文件存储：Telegram（不用 D1/KV）
        ├─ 元数据：KV（优先）或 D1（备选）
        └─ 分片信息：KV（>D1）
```

---

## 3. Telegram 分片上传的深度分析

### 3.1 分片策略源代码分析

```javascript
// 来源：/functions/upload/index.js，第 392-398 行
const CHUNK_SIZE = 20 * 1024 * 1024; // 20 MB

if (fileSize > CHUNK_SIZE) {
    return await uploadLargeFileToTelegram(
        env, file, fullId, metadata, fileName, fileType, 
        url, returnLink, tgBotToken, tgChatId, tgChannel
    );
}

// 来源：/functions/upload/chunkUpload.js，第 1023-1029 行
const CHUNK_SIZE = 20 * 1024 * 1024; // 20 MB
const fileSize = file.size;
const totalChunks = Math.ceil(fileSize / CHUNK_SIZE);

if (totalChunks > 50) {  // ← 限制分片数
    return createResponse(
        'Error: File too large (exceeds 1GB limit)', 
        { status: 413 }
    );
}

// 1 GB 计算：
// 50 chunks × 20 MB/chunk = 1000 MB = 1 GB ✓
```

### 3.2 分片上传流程

```javascript
// 来源：/functions/upload/chunkUpload.js，第 1037-1077 行

for (let i = 0; i < totalChunks; i++) {
    const start = i * CHUNK_SIZE;                    // 分片起始位置
    const end = Math.min(start + CHUNK_SIZE, fileSize);  // 分片结束位置
    const chunkBlob = file.slice(start, end);        // 切割分片

    const chunkFileName = `${fileName}.part${i.toString().padStart(3, '0')}`;
    // 生成分片文件名，例如：large_file.iso.part000

    // 带重试的上传
    const chunkInfo = await uploadChunkToTelegramWithRetry(
        tgBotToken,
        tgChatId,
        chunkBlob,
        chunkFileName,
        i,
        totalChunks
    );

    if (!chunkInfo) {
        throw new Error(
            `Failed to upload chunk ${i + 1}/${totalChunks} after retries`
        );
    }

    // 保存分片信息
    chunks.push({
        index: i,
        fileId: chunkInfo.file_id,     // Telegram 返回的 file_id
        size: chunkInfo.file_size,     // 实际上传的大小
        fileName: chunkFileName
    });

    // CPU 限制保护
    if (i > 0 && i % 10 === 0) {
        await new Promise(resolve => setTimeout(resolve, 50));
    }
}
```

### 3.3 单个分片上传函数

```javascript
// 来源：/functions/upload/chunkUpload.js，第 1119-1149 行

async function uploadChunkToTelegramWithRetry(
    tgBotToken, 
    tgChatId, 
    chunkBlob, 
    chunkFileName, 
    chunkIndex, 
    totalChunks, 
    maxRetries = 2
) {
    for (let attempt = 0; attempt < maxRetries; attempt++) {
        try {
            const tgAPI = new TelegramAPI(tgBotToken);

            // 生成分片标题
            const caption = `Part ${chunkIndex + 1}/${totalChunks}`;

            // 上传到 Telegram
            const response = await tgAPI.sendFile(
                chunkBlob,
                tgChatId,
                'sendDocument',  // ← 使用文档 API
                'document',
                caption,
                chunkFileName
            );

            if (!response.ok) {
                throw new Error(response.description || 'Telegram API error');
            }

            // 提取文件信息
            const fileInfo = tgAPI.getFileInfo(response);
            if (!fileInfo) {
                throw new Error('Failed to extract file info from response');
            }

            return fileInfo;  // { file_id, file_size, ... }

        } catch (error) {
            console.warn(
                `Chunk ${chunkIndex} upload attempt ${attempt + 1} failed:`, 
                error.message
            );

            if (attempt === maxRetries - 1) {
                return null;  // 所有重试都失败
            }

            // 指数退避重试延迟
            await new Promise(resolve => 
                setTimeout(resolve, 500 * (attempt + 1))
            );
        }
    }

    return null;
}
```

### 3.4 Telegram API 限制

根据 Telegram Bot API 官方文档：

```
Telegram Bot API 文件限制：
┌────────────────────────────────────────┐
│ 接口         │ 最大文件大小           │
├────────────────────────────────────────┤
│ sendPhoto    │ 5 MB                   │
│ sendAudio    │ 50 MB                  │
│ sendDocument │ 50 MB                  │ ← 代码使用的
│ sendVideo    │ 50 MB                  │
│ sendAnimation│ 50 MB                  │
│ getFile      │ 20 MB（下载限制）      │
└────────────────────────────────────────┘

Cloudflare Worker 时间限制：
├─ CPU 时间：30 秒
├─ 墙钟时间：600 秒

所以：
├─ 20 MB 分片上传时间：~2-5 秒
├─ 50 个分片：~100-250 秒 = 1.7-4.2 分钟
└─ 在 600 秒墙钟时间内可完成 ✓
```

### 3.5 为什么选择 20 MB 而不是 50 MB？

```
平衡因素：

1. 安全性：
   ├─ Telegram 50 MB 限制
   └─ 代码选择 20 MB（留有缓冲）

2. 可靠性：
   ├─ 网络超时风险随文件大小增加而增加
   └─ 20 MB 更容易在不稳定网络中成功

3. 性能：
   ├─ Worker CPU 时间限制 30 秒
   ├─ 20 MB 分片需要 ~2-3 秒
   ├─ 50 个 20 MB 分片 = 100-150 秒（后台任务）
   └─ 如果用 50 MB，只能 10-12 个分片 = 500-600 MB 最大

4. 分片数量：
   ├─ 20 MB 分片：最多 50 个 = 1 GB
   └─ 50 MB 分片：最多 50 个 = 2.5 GB（但实际受 CPU 限制）
```

---

## 4. 数据库操作中的关键代码点

### 4.1 写入元数据的方式

```javascript
// 来源：/functions/upload/index.js，第 469-471 行
await db.put(fullId, "", {
    metadata: metadata,  // 元数据对象
});

// KV 中实际存储的：
{
    key: fullId,
    value: "",          // 空值（元数据存储在 metadata 参数中）
    metadata: {         // 由 KV 自动管理
        FileName: "photo.jpg",
        FileType: "image/jpeg",
        FileSize: "2.5",
        // ...完整的 metadata 对象
    }
}

// D1 中实际存储的：
INSERT INTO files (id, value, metadata, ...)
VALUES (
    fullId,
    "",
    JSON.stringify(metadata),  // 序列化为 JSON 字符串
    ...
);
```

### 4.2 写入分片信息的方式

```javascript
// 来源：/functions/upload/chunkMerge.js，第 442-445 行
const chunksData = JSON.stringify(chunks);
await db.put(finalFileId, chunksData, { metadata });

// 结构：
{
    key: finalFileId,
    value: chunksData,  // JSON 字符串：[{index,fileId,size,fileName},..]
    metadata: {
        FileName: "large_file.iso",
        IsChunked: true,
        TotalChunks: 50,
        FileSize: "1024",
        // ...
    }
}

// 分片数据大小计算：
chunks = [
    {
        index: 0,
        fileId: "AgACAgQA...(~150 chars)" → ~150 bytes,
        size: 20971520,                    → ~10 bytes,
        fileName: "file.part000"           → ~15 bytes,
    },
    // ... 重复 49 次
];
// JSON 序列化后：50 * ~175 = ~8,750 bytes = 8.75 KB ✓
```

### 4.3 检查数据库配置

```javascript
// 来源：/functions/utils/databaseAdapter.js，第 161-172 行

export function checkDatabaseConfig(env) {
    var hasD1 = env.img_d1 && typeof env.img_d1.prepare === 'function';
    var hasKV = env.img_url && typeof env.img_url.get === 'function';

    return {
        hasD1: hasD1,
        hasKV: hasKV,
        usingD1: hasD1,          // ← D1 优先使用
        usingKV: !hasD1 && hasKV, // ← KV 作为备选
        configured: hasD1 || hasKV
    };
}

// 注意：代码逻辑是"如果有 D1 就用 D1，否则用 KV"
// 但在 createDatabaseAdapter 中（第 13-25 行）是 KV 优先！
// 这里存在不一致性，建议验证实际行为
```

---

## 5. 性能和规模分析

### 5.1 不同文件大小的性能特征

```
文件大小    分片数   总时间*  数据库写入  建议方案
──────────────────────────────────────────────
1 MB        1       1 秒      < 1 KB     D1/KV
10 MB       1       2 秒      < 10 KB    D1/KV
20 MB       1       3 秒      < 20 KB    KV（D1 可能失败）
100 MB      5       15 秒     < 5 KB     KV ✓
500 MB      25      90 秒     < 5 KB     KV ✓
1 GB        50      180 秒    < 10 KB    KV ✓
> 1 GB      > 50    ✗ 限制   ✗ 不支持   ✗ 不支持

* 仅为后台处理时间估计，不包括客户端上传时间
```

### 5.2 并发性能

```javascript
// 假设情况：同时上传 10 个 100 MB 文件

// KV 角度：
├─ 每个文件元数据：< 10 KB
├─ 10 个文件：< 100 KB
├─ KV 吞吐量：> 1000 req/sec
└─ 结果：✓ 无问题

// D1 角度：
├─ 每个文件行：~50-100 KB
├─ 10 个文件：~500 KB - 1 MB
├─ 行级锁开销
└─ 结果：⚠️ 可能出现锁竞争

// Telegram 角度：
├─ 每个文件 5 个分片，共 50 个 API 请求
├─ Telegram 速率限制
└─ 结果：⚠️ 可能触发速率限制
```

---

## 6. 故障和错误场景

### 6.1 D1 故障场景

```javascript
// 场景 1：元数据过大导致行超限
{
    status: "error",
    message: "SQLITE_TOOBIG: statement too large"
    // 或
    message: "Row size exceeds maximum"
}

// 场景 2：并发写入冲突
{
    status: "error",
    message: "database is locked"
}

// 场景 3：事务超时
{
    status: "error",
    message: "transaction too large"
}
```

### 6.2 KV 故障场景

```javascript
// 场景 1：值过大
{
    status: "error",
    message: "Value is too large (max 25MB)"
}

// 场景 2：速率限制
{
    status: 429,
    message: "Too Many Requests"
}

// 场景 3：键不存在
{
    status: 404,
    message: "Key not found"
}
```

### 6.3 Telegram 故障场景

```javascript
// 场景 1：文件过大
{
    ok: false,
    description: "Request Entity Too Large"
}

// 场景 2：网络超时
{
    ok: false,
    error_code: 408,
    description: "Request timeout"
}

// 场景 3：速率限制
{
    ok: false,
    error_code: 429,
    description: "Too Many Requests: retry after 10"
}
```

---

## 7. 建议的改进方案

### 7.1 显式的大小限制检查

```javascript
// 建议在 /functions/upload/uploadTools.js 中添加

const LIMITS = {
    D1: {
        MAX_METADATA_SIZE: 100 * 1024,        // 100 KB
        MAX_VALUE_SIZE: 1024 * 1024,          // 1 MB
        MAX_ROW_SIZE: 5 * 1024 * 1024,        // 5 MB（保守估计）
    },
    KV: {
        MAX_VALUE_SIZE: 25 * 1024 * 1024,     // 25 MB
        MAX_METADATA_SIZE: 1024,              // 1 KB
    },
    TELEGRAM: {
        CHUNK_SIZE: 20 * 1024 * 1024,         // 20 MB
        MAX_CHUNKS: 50,
        MAX_FILE_SIZE: 50 * 20 * 1024 * 1024, // 1 GB
    }
};

function validateMetadataSize(metadata, targetDB) {
    const serialized = JSON.stringify(metadata);
    const size = Buffer.byteLength(serialized, 'utf8');
    
    if (targetDB === 'D1') {
        if (size > LIMITS.D1.MAX_METADATA_SIZE) {
            throw new Error(
                `Metadata too large for D1: ${size} bytes (max ${LIMITS.D1.MAX_METADATA_SIZE})`
            );
        }
    } else if (targetDB === 'KV') {
        if (size > LIMITS.KV.MAX_METADATA_SIZE) {
            throw new Error(
                `Metadata too large for KV: ${size} bytes (max ${LIMITS.KV.MAX_METADATA_SIZE})`
            );
        }
    }
}
```

### 7.2 自动数据库选择

```javascript
// 建议改进 /functions/utils/databaseAdapter.js

function selectOptimalDatabase(env, data_size) {
    // 强制 KV 用于大文件
    if (data_size > 100 * 1024) {  // > 100 KB
        if (env.img_url && typeof env.img_url.get === 'function') {
            return new KVAdapter(env.img_url);
        }
    }
    
    // 对于小文件，优先 D1（更好的查询能力）
    if (env.img_d1 && typeof env.img_d1.prepare === 'function') {
        return new D1Database(env.img_d1);
    }
    
    // 备选 KV
    if (env.img_url && typeof env.img_url.get === 'function') {
        return new KVAdapter(env.img_url);
    }
    
    return null;
}
```

### 7.3 监控和告警

```javascript
// 建议添加监控

async function monitorDatabaseOperation(
    operationType, 
    dataSize, 
    executionTime, 
    success
) {
    // 发送到 Sentry 或其他监控系统
    const metrics = {
        operation: operationType,
        data_size: dataSize,
        execution_time_ms: executionTime,
        success: success,
        timestamp: Date.now(),
    };
    
    // 告警阈值
    if (dataSize > 5 * 1024 * 1024) {  // > 5 MB
        console.warn('Large data operation detected:', metrics);
    }
    
    if (executionTime > 5000) {  // > 5 秒
        console.warn('Slow database operation detected:', metrics);
    }
}
```

---

## 8. 测试检查清单

### 8.1 D1 测试用例

```javascript
// 测试 1：小元数据
test('D1 with small metadata', async () => {
    const metadata = { fileName: 'test.txt', fileSize: '1' };
    // → 应该成功
});

// 测试 2：中等元数据
test('D1 with medium metadata (50 KB)', async () => {
    const metadata = { /* 50 KB of data */ };
    // → 应该成功
});

// 测试 3：大元数据
test('D1 with large metadata (1 MB)', async () => {
    const metadata = { /* 1 MB of data */ };
    // → 可能失败，获取错误信息
});

// 测试 4：分片信息
test('D1 with 50 chunks info', async () => {
    const value = JSON.stringify(/* 50 chunks */);
    // → 应该成功
});
```

### 8.2 KV 测试用例

```javascript
// 测试 1：大值（20 MB）
test('KV with 20 MB value', async () => {
    const value = 'x'.repeat(20 * 1024 * 1024);
    // → 应该成功
});

// 测试 2：超大值（超过 25 MB）
test('KV with 30 MB value (exceeds limit)', async () => {
    const value = 'x'.repeat(30 * 1024 * 1024);
    // → 应该失败，返回 413 或类似错误
});

// 测试 3：1000 个分片信息
test('KV with 1000 chunks info', async () => {
    const chunks = Array(1000).fill({...});
    // → 应该成功（大小仍 < 25 MB）
});
```

### 8.3 Telegram 测试用例

```javascript
// 测试 1：50 个 20 MB 分片
test('Telegram 1 GB file upload', async () => {
    // → 应该成功（在 180-300 秒内）
});

// 测试 2：超过 50 个分片
test('Telegram > 50 chunks (exceeds limit)', async () => {
    // → 应该立即失败
});

// 测试 3：网络中断恢复
test('Telegram chunk upload with retry', async () => {
    // → 应该在 2 次重试后成功或失败
});
```

---

## 9. 结论与建议

### 关键发现总结

```
D1（SQLite）：
├─ 最适合：< 100 KB 的结构化数据
├─ 警惕：> 1 MB 的行大小
└─ 结论：不适合存储大文件元数据

KV：
├─ 最适合：元数据存储和分片信息
├─ 限制：单值 25 MB
└─ 结论：当前架构的最佳选择

Telegram：
├─ 最适合：实际文件内容存储
├─ 限制：单片 50 MB（代码限制 20 MB），最多 50 片
└─ 结论：1 GB 以内的文件存储理想方案
```

### 立即可采取的行动

1. **添加明确的大小限制检查**（见 7.1）
2. **改进数据库选择逻辑**（见 7.2）
3. **添加监控和告警**（见 7.3）
4. **运行测试检查清单**（见 8）


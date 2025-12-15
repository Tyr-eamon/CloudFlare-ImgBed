# CloudFlare-ImgBed 大文件限制 - 测试指南

## 快速开始

本指南提供可运行的测试代码，用于验证 D1、KV 和 Telegram 的大文件上传限制。

---

## 1. D1 大小限制测试

### 1.1 测试代码模板

创建文件 `/functions/upload/tests/d1-size-limits.test.js`：

```javascript
import { getDatabase } from '../../utils/databaseAdapter.js';

/**
 * D1 大小限制测试套件
 * 测试在不同的元数据和值大小下 D1 的行为
 */

export async function runD1SizeTests(env) {
    const db = getDatabase(env);
    const results = [];

    console.log('=== Starting D1 Size Limit Tests ===\n');

    // 测试 1：小元数据（应该成功）
    try {
        const smallMetadata = {
            FileName: 'test.txt',
            FileType: 'text/plain',
            FileSize: '1',
            TimeStamp: Date.now(),
        };
        
        const serialized = JSON.stringify(smallMetadata);
        console.log(`Test 1: Small metadata (${serialized.length} bytes)`);
        
        await db.put('test_small', '', { metadata: smallMetadata });
        console.log('✓ SUCCESS: D1 handles small metadata\n');
        results.push({ test: 1, status: 'PASS', size: serialized.length });
    } catch (error) {
        console.log(`✗ FAILED: ${error.message}\n`);
        results.push({ test: 1, status: 'FAIL', error: error.message });
    }

    // 测试 2：中等元数据（50 KB，应该成功）
    try {
        const mediumData = 'x'.repeat(50 * 1024);
        const mediumMetadata = {
            FileName: 'large_file.iso',
            FileType: 'application/octet-stream',
            FileSize: '1024',
            Description: mediumData,
            Tags: Array(100).fill('tag'),
            TimeStamp: Date.now(),
        };
        
        const serialized = JSON.stringify(mediumMetadata);
        console.log(`Test 2: Medium metadata (${serialized.length} bytes, ~${(serialized.length / 1024).toFixed(2)} KB)`);
        
        await db.put('test_medium', '', { metadata: mediumMetadata });
        console.log('✓ SUCCESS: D1 handles 50 KB metadata\n');
        results.push({ test: 2, status: 'PASS', size: serialized.length });
    } catch (error) {
        console.log(`✗ FAILED: ${error.message}\n`);
        results.push({ test: 2, status: 'FAIL', error: error.message });
    }

    // 测试 3：大元数据（500 KB，可能失败）
    try {
        const largeData = 'x'.repeat(500 * 1024);
        const largeMetadata = {
            FileName: 'huge_file.bin',
            FileType: 'application/octet-stream',
            FileSize: '10000',
            Description: largeData,
            TimeStamp: Date.now(),
        };
        
        const serialized = JSON.stringify(largeMetadata);
        console.log(`Test 3: Large metadata (${serialized.length} bytes, ~${(serialized.length / 1024).toFixed(2)} KB)`);
        
        await db.put('test_large', '', { metadata: largeMetadata });
        console.log('✓ SUCCESS: D1 handles 500 KB metadata\n');
        results.push({ test: 3, status: 'PASS', size: serialized.length });
    } catch (error) {
        console.log(`✗ FAILED: ${error.message}\n`);
        results.push({ test: 3, status: 'FAIL', error: error.message });
    }

    // 测试 4：分片信息（50 个，应该成功）
    try {
        const chunks = Array(50).fill(null).map((_, i) => ({
            index: i,
            fileId: 'AgACAgQA' + 'x'.repeat(150),
            size: 20 * 1024 * 1024,
            fileName: `file.part${i.toString().padStart(3, '0')}`
        }));
        
        const chunksData = JSON.stringify(chunks);
        console.log(`Test 4: 50 chunks info (${chunksData.length} bytes, ~${(chunksData.length / 1024).toFixed(2)} KB)`);
        
        const metadata = {
            FileName: 'large_file.iso',
            IsChunked: true,
            TotalChunks: 50,
        };
        
        await db.put('test_chunks_50', chunksData, { metadata });
        console.log('✓ SUCCESS: D1 handles 50 chunks info\n');
        results.push({ test: 4, status: 'PASS', size: chunksData.length });
    } catch (error) {
        console.log(`✗ FAILED: ${error.message}\n`);
        results.push({ test: 4, status: 'FAIL', error: error.message });
    }

    // 测试 5：超大分片信息（1000 个，可能失败）
    try {
        const chunks = Array(1000).fill(null).map((_, i) => ({
            index: i,
            fileId: 'AgACAgQA' + 'x'.repeat(150),
            size: 20 * 1024 * 1024,
            fileName: `file.part${i.toString().padStart(4, '0')}`
        }));
        
        const chunksData = JSON.stringify(chunks);
        console.log(`Test 5: 1000 chunks info (${chunksData.length} bytes, ~${(chunksData.length / 1024).toFixed(2)} KB)`);
        
        const metadata = {
            FileName: 'huge_file.iso',
            IsChunked: true,
            TotalChunks: 1000,
        };
        
        await db.put('test_chunks_1000', chunksData, { metadata });
        console.log('✓ SUCCESS: D1 handles 1000 chunks info\n');
        results.push({ test: 5, status: 'PASS', size: chunksData.length });
    } catch (error) {
        console.log(`✗ FAILED: ${error.message}\n`);
        results.push({ test: 5, status: 'FAIL', error: error.message });
    }

    // 测试 6：完整的大文件场景
    try {
        const largeMetadata = {
            FileName: 'movie.mkv',
            FileType: 'video/x-matroska',
            FileSize: '1024',
            UploadIP: '203.0.113.42',
            UploadAddress: 'Country, City, ISP Information with Long Text',
            Label: 'safe',
            Directory: 'folder/subfolder/deep/path/',
            Tags: ['video', 'movie', 'mkv', 'archive', 'backup'],
            Channel: 'TelegramNew',
            ChannelName: 'main_channel',
            TgFileId: 'AgACAgQA' + 'x'.repeat(100),
            TgChatId: '-1001234567890',
            TgBotToken: '123456:ABC-DEF1234567890GHIJKLMNOPQRSTUVWXYZ',
            IsChunked: true,
            TotalChunks: 50,
            TimeStamp: Date.now(),
        };
        
        const chunks = Array(50).fill(null).map((_, i) => ({
            index: i,
            fileId: 'AgACAgQA' + 'x'.repeat(100),
            size: 20 * 1024 * 1024,
            fileName: `movie.mkv.part${i.toString().padStart(3, '0')}`
        }));
        
        const serialized = JSON.stringify(largeMetadata);
        const chunksData = JSON.stringify(chunks);
        
        console.log(`Test 6: Full large file scenario`);
        console.log(`  - Metadata: ${(serialized.length / 1024).toFixed(2)} KB`);
        console.log(`  - Chunks: ${(chunksData.length / 1024).toFixed(2)} KB`);
        console.log(`  - Total: ${((serialized.length + chunksData.length) / 1024).toFixed(2)} KB`);
        
        await db.put('test_full_scenario', chunksData, { metadata: largeMetadata });
        console.log('✓ SUCCESS: D1 handles full large file scenario\n');
        results.push({ test: 6, status: 'PASS', size: serialized.length + chunksData.length });
    } catch (error) {
        console.log(`✗ FAILED: ${error.message}\n`);
        results.push({ test: 6, status: 'FAIL', error: error.message });
    }

    // 汇总结果
    console.log('=== Test Summary ===');
    const passed = results.filter(r => r.status === 'PASS').length;
    const failed = results.filter(r => r.status === 'FAIL').length;
    console.log(`Total: ${results.length}, Passed: ${passed}, Failed: ${failed}\n`);
    
    if (failed > 0) {
        console.log('Failed tests:');
        results.filter(r => r.status === 'FAIL').forEach(r => {
            console.log(`  - Test ${r.test}: ${r.error}`);
        });
    }

    return results;
}

// 导出便捷函数
export async function testD1Limits(env) {
    return await runD1SizeTests(env);
}
```

### 1.2 运行 D1 测试

在 `/functions/api/admin/test-limits.js` 中创建端点：

```javascript
import { testD1Limits } from '../tests/d1-size-limits.test.js';

export async function onRequest(context) {
    const { env, request } = context;
    
    // 仅允许 POST 请求
    if (request.method !== 'POST') {
        return new Response('Method Not Allowed', { status: 405 });
    }

    try {
        console.log('Starting D1 size limit tests...');
        const results = await testD1Limits(env);
        
        return new Response(JSON.stringify({
            success: true,
            testType: 'D1 Size Limits',
            results: results
        }, null, 2), {
            status: 200,
            headers: { 'Content-Type': 'application/json' }
        });
    } catch (error) {
        return new Response(JSON.stringify({
            success: false,
            error: error.message
        }, null, 2), {
            status: 500,
            headers: { 'Content-Type': 'application/json' }
        });
    }
}
```

---

## 2. KV 大小限制测试

### 2.1 KV 测试代码

创建文件 `/functions/upload/tests/kv-size-limits.test.js`：

```javascript
/**
 * KV 大小限制测试套件
 * 测试 KV 能否存储特定大小的值
 */

export async function testKVSizeLimit(kv, sizeInMB) {
    try {
        // 创建指定大小的数据
        const data = 'x'.repeat(sizeInMB * 1024 * 1024);
        const testKey = `kv_test_${sizeInMB}mb_${Date.now()}`;
        
        console.log(`Testing KV with ${sizeInMB} MB value...`);
        
        // 尝试存储
        await kv.put(testKey, data);
        console.log(`✓ SUCCESS: KV accepts ${sizeInMB} MB value`);
        
        // 尝试读取验证
        const retrieved = await kv.get(testKey);
        if (retrieved && retrieved.length === data.length) {
            console.log(`✓ VERIFIED: Retrieved data matches original size`);
        }
        
        // 清理
        await kv.delete(testKey);
        
        return { success: true, size: sizeInMB };
    } catch (error) {
        console.log(`✗ FAILED: ${error.message}`);
        return { success: false, size: sizeInMB, error: error.message };
    }
}

export async function runKVSizeTests(env) {
    const results = [];

    console.log('=== Starting KV Size Limit Tests ===\n');

    // KV 限制是 25 MB
    const testSizes = [1, 5, 10, 20, 24, 24.9, 25, 25.1, 30];

    for (const size of testSizes) {
        const result = await testKVSizeLimit(env.img_url, size);
        results.push(result);
    }

    console.log('\n=== KV Test Summary ===');
    const passed = results.filter(r => r.success).length;
    const failed = results.filter(r => !r.success).length;
    
    console.log(`Total: ${results.length}`);
    console.log(`Passed: ${passed}`);
    console.log(`Failed: ${failed}`);
    
    console.log('\nResults:');
    results.forEach(r => {
        const status = r.success ? '✓' : '✗';
        console.log(`${status} ${r.size} MB: ${r.success ? 'OK' : r.error}`);
    });

    return results;
}
```

### 2.2 运行 KV 测试

```javascript
// 在管理员端点中调用
import { runKVSizeTests } from '../tests/kv-size-limits.test.js';

export async function onRequest(context) {
    const { env, request } = context;
    
    if (request.method !== 'POST') {
        return new Response('Method Not Allowed', { status: 405 });
    }

    try {
        const results = await runKVSizeTests(env);
        
        return new Response(JSON.stringify({
            success: true,
            testType: 'KV Size Limits',
            results: results
        }, null, 2), {
            status: 200,
            headers: { 'Content-Type': 'application/json' }
        });
    } catch (error) {
        return new Response(JSON.stringify({
            success: false,
            error: error.message
        }, null, 2), {
            status: 500,
            headers: { 'Content-Type': 'application/json' }
        });
    }
}
```

---

## 3. 文件上传限制测试

### 3.1 不同大小文件的上传测试

创建测试文件 `/functions/upload/tests/upload-size-limits.test.js`：

```javascript
import { createResponse } from '../uploadTools.js';

/**
 * 测试不同大小的文件上传
 */

export async function generateTestFile(sizeInMB) {
    // 创建一个特定大小的虚拟文件
    const sizeInBytes = sizeInMB * 1024 * 1024;
    const chunkSize = 1024 * 1024; // 1 MB chunks
    const chunks = [];
    
    for (let i = 0; i < sizeInBytes; i += chunkSize) {
        const size = Math.min(chunkSize, sizeInBytes - i);
        chunks.push(new Uint8Array(size));
    }
    
    const blob = new Blob(chunks, { type: 'application/octet-stream' });
    return new File([blob], `test_${sizeInMB}mb.bin`, { 
        type: 'application/octet-stream',
        size: sizeInBytes
    });
}

export async function testFileUploadSize(sizeInMB, uploadFunction) {
    try {
        console.log(`Testing ${sizeInMB} MB file upload...`);
        
        const file = await generateTestFile(sizeInMB);
        
        // 需要调用者提供的 uploadFunction
        const result = await uploadFunction(file);
        
        if (result.success) {
            console.log(`✓ SUCCESS: ${sizeInMB} MB file uploaded`);
            return { success: true, size: sizeInMB };
        } else {
            console.log(`✗ FAILED: ${result.error}`);
            return { success: false, size: sizeInMB, error: result.error };
        }
    } catch (error) {
        console.log(`✗ ERROR: ${error.message}`);
        return { success: false, size: sizeInMB, error: error.message };
    }
}

export async function runUploadSizeTests(env, uploadHandler) {
    const results = [];
    
    console.log('=== Starting Upload Size Tests ===\n');
    
    // 测试不同大小的文件
    const testSizes = [
        1,      // 1 MB - 小文件
        10,     // 10 MB - 中等文件
        20,     // 20 MB - 单个分片大小
        40,     // 40 MB - 2 个分片
        100,    // 100 MB - 5 个分片
        500,    // 500 MB - 25 个分片
        1000,   // 1000 MB - 50 个分片（限制）
        1100    // 1100 MB - 超过限制
    ];
    
    for (const size of testSizes) {
        const result = await testFileUploadSize(size, uploadHandler);
        results.push(result);
    }
    
    console.log('\n=== Upload Size Test Summary ===');
    const passed = results.filter(r => r.success).length;
    const failed = results.filter(r => !r.success).length;
    
    console.log(`Total: ${results.length}`);
    console.log(`Passed: ${passed}`);
    console.log(`Failed: ${failed}`);
    
    console.log('\nResults:');
    results.forEach(r => {
        const status = r.success ? '✓' : '✗';
        const msg = r.success ? 'OK' : r.error;
        console.log(`${status} ${r.size} MB: ${msg}`);
    });
    
    return results;
}
```

---

## 4. 性能基准测试

### 4.1 写入性能测试

```javascript
/**
 * 性能基准测试
 */

export async function benchmarkDatabaseWrite(db, sizeInKB) {
    const metadata = {
        FileName: `test_${sizeInKB}kb.bin`,
        FileSize: (sizeInKB / 1024).toFixed(2),
        Description: 'x'.repeat(sizeInKB * 1024),
        TimeStamp: Date.now(),
    };
    
    const startTime = Date.now();
    try {
        await db.put(`bench_write_${sizeInKB}kb`, '', { metadata });
        const duration = Date.now() - startTime;
        
        return {
            success: true,
            size: sizeInKB,
            duration,
            throughput: (sizeInKB / duration * 1000).toFixed(2) // KB/s
        };
    } catch (error) {
        const duration = Date.now() - startTime;
        return {
            success: false,
            size: sizeInKB,
            duration,
            error: error.message
        };
    }
}

export async function benchmarkDatabaseRead(db, key) {
    const startTime = Date.now();
    try {
        const result = await db.getWithMetadata(key);
        const duration = Date.now() - startTime;
        
        const size = result ? 
            JSON.stringify(result.metadata).length + 
            (result.value ? result.value.length : 0) : 0;
        
        return {
            success: true,
            key,
            duration,
            size,
            throughput: (size / duration / 1024).toFixed(2) // KB/s
        };
    } catch (error) {
        const duration = Date.now() - startTime;
        return {
            success: false,
            key,
            duration,
            error: error.message
        };
    }
}

export async function runPerformanceBenchmarks(env) {
    const db = getDatabase(env);
    const results = {
        writes: [],
        reads: []
    };
    
    console.log('=== Performance Benchmarks ===\n');
    
    // 写入基准
    console.log('Write Benchmarks:');
    for (const size of [10, 50, 100, 500, 1000]) {
        const result = await benchmarkDatabaseWrite(db, size);
        results.writes.push(result);
        
        if (result.success) {
            console.log(
                `✓ ${size} KB in ${result.duration}ms ` +
                `(${result.throughput} KB/s)`
            );
        } else {
            console.log(`✗ ${size} KB: ${result.error}`);
        }
    }
    
    return results;
}
```

---

## 5. 端到端集成测试

### 5.1 完整的上传流程测试

```javascript
/**
 * 端到端测试
 */

export async function testCompleteUploadFlow(
    env,
    fileSizeInMB,
    uploadChannel = 'telegram'
) {
    console.log(`Testing complete upload flow for ${fileSizeInMB} MB file...`);
    
    const steps = [];
    
    try {
        // 步骤 1：生成测试文件
        console.log('Step 1: Generating test file...');
        const file = await generateTestFile(fileSizeInMB);
        steps.push({
            name: 'Generate File',
            status: 'COMPLETE',
            duration: null
        });
        
        // 步骤 2：准备元数据
        console.log('Step 2: Preparing metadata...');
        const metadata = {
            FileName: file.name,
            FileType: file.type,
            FileSize: (fileSizeInMB).toFixed(2),
            Channel: uploadChannel,
            TimeStamp: Date.now(),
        };
        steps.push({
            name: 'Prepare Metadata',
            status: 'COMPLETE',
            duration: null
        });
        
        // 步骤 3：检查分片需求
        console.log('Step 3: Checking chunk requirements...');
        const CHUNK_SIZE = 20 * 1024 * 1024;
        const totalChunks = Math.ceil((fileSizeInMB * 1024 * 1024) / CHUNK_SIZE);
        
        console.log(`  Total chunks needed: ${totalChunks}`);
        if (totalChunks > 50) {
            console.log('  ✗ Exceeds maximum chunks limit (50)');
            return {
                success: false,
                error: `File too large: needs ${totalChunks} chunks, max 50`,
                steps: [...steps, {
                    name: 'Validate Chunks',
                    status: 'FAILED',
                    error: 'Chunks exceed limit'
                }]
            };
        }
        steps.push({
            name: 'Validate Chunks',
            status: 'COMPLETE',
            chunks: totalChunks
        });
        
        // 步骤 4：测试数据库存储
        console.log('Step 4: Testing database storage...');
        const db = getDatabase(env);
        const testKey = `e2e_test_${Date.now()}`;
        
        const chunksData = JSON.stringify(
            Array(totalChunks).fill(null).map((_, i) => ({
                index: i,
                fileId: 'test_file_id_' + i,
                size: Math.min(CHUNK_SIZE, fileSizeInMB * 1024 * 1024 - i * CHUNK_SIZE),
                fileName: `${file.name}.part${i.toString().padStart(3, '0')}`
            }))
        );
        
        const startDb = Date.now();
        await db.put(testKey, chunksData, { metadata });
        const dbDuration = Date.now() - startDb;
        
        steps.push({
            name: 'Database Storage',
            status: 'COMPLETE',
            duration: dbDuration,
            dataSize: `${(chunksData.length / 1024).toFixed(2)} KB`
        });
        
        // 清理
        await db.delete(testKey);
        
        console.log('\n=== E2E Test Complete ===');
        return {
            success: true,
            fileSizeInMB,
            totalChunks,
            steps
        };
        
    } catch (error) {
        console.log(`\n✗ E2E Test Failed: ${error.message}`);
        return {
            success: false,
            fileSizeInMB,
            error: error.message,
            steps,
            failedAt: steps.length
        };
    }
}
```

---

## 6. 测试执行命令

### 6.1 运行所有测试

创建管理员端点 `/functions/api/admin/run-tests.js`：

```javascript
import { testD1Limits } from '../../upload/tests/d1-size-limits.test.js';
import { runKVSizeTests } from '../../upload/tests/kv-size-limits.test.js';
import { runPerformanceBenchmarks } from '../../upload/tests/performance-benchmarks.test.js';
import { testCompleteUploadFlow } from '../../upload/tests/e2e-tests.test.js';

export async function onRequest(context) {
    const { env, request, url } = context;
    
    if (request.method !== 'POST') {
        return new Response('Method Not Allowed', { status: 405 });
    }
    
    const testType = url.searchParams.get('type') || 'all';
    const results = {};
    
    try {
        if (testType === 'all' || testType === 'd1') {
            console.log('Running D1 tests...');
            results.d1 = await testD1Limits(env);
        }
        
        if (testType === 'all' || testType === 'kv') {
            console.log('Running KV tests...');
            results.kv = await runKVSizeTests(env);
        }
        
        if (testType === 'all' || testType === 'performance') {
            console.log('Running performance benchmarks...');
            results.performance = await runPerformanceBenchmarks(env);
        }
        
        if (testType === 'all' || testType === 'e2e') {
            console.log('Running end-to-end tests...');
            results.e2e = {
                _1mb: await testCompleteUploadFlow(env, 1),
                _100mb: await testCompleteUploadFlow(env, 100),
                _500mb: await testCompleteUploadFlow(env, 500),
                _1gb: await testCompleteUploadFlow(env, 1000),
            };
        }
        
        return new Response(JSON.stringify({
            success: true,
            testType,
            results,
            timestamp: new Date().toISOString()
        }, null, 2), {
            status: 200,
            headers: { 'Content-Type': 'application/json' }
        });
        
    } catch (error) {
        return new Response(JSON.stringify({
            success: false,
            error: error.message,
            stack: error.stack
        }, null, 2), {
            status: 500,
            headers: { 'Content-Type': 'application/json' }
        });
    }
}
```

### 6.2 curl 命令运行测试

```bash
# 运行所有测试
curl -X POST https://your-domain/api/admin/run-tests

# 仅运行 D1 测试
curl -X POST https://your-domain/api/admin/run-tests?type=d1

# 仅运行 KV 测试
curl -X POST https://your-domain/api/admin/run-tests?type=kv

# 仅运行性能基准
curl -X POST https://your-domain/api/admin/run-tests?type=performance

# 仅运行端到端测试
curl -X POST https://your-domain/api/admin/run-tests?type=e2e

# 保存测试结果到文件
curl -X POST https://your-domain/api/admin/run-tests > test_results.json
```

---

## 7. 测试结果解读

### 7.1 预期的 D1 测试结果

```
Test 1: Small metadata (< 1 KB)
✓ 应该成功

Test 2: Medium metadata (50 KB)
✓ 应该成功

Test 3: Large metadata (500 KB)
? 可能成功，取决于 D1 的行大小限制

Test 4: 50 chunks info (~10 KB)
✓ 应该成功

Test 5: 1000 chunks info (~150 KB)
✓ 应该成功

Test 6: Full large file scenario (~15 KB)
✓ 应该成功
```

### 7.2 预期的 KV 测试结果

```
1 MB: ✓ 通过
5 MB: ✓ 通过
10 MB: ✓ 通过
20 MB: ✓ 通过
24 MB: ✓ 通过
24.9 MB: ✓ 通过
25 MB: ✓ 通过（可能）
25.1 MB: ✗ 失败（超过限制）
30 MB: ✗ 失败（超过限制）
```

### 7.3 预期的上传大小测试结果

```
1 MB: ✓ 成功（单个分片）
10 MB: ✓ 成功（单个分片）
20 MB: ✓ 成功（恰好一个分片）
40 MB: ✓ 成功（2 个分片）
100 MB: ✓ 成功（5 个分片）
500 MB: ✓ 成功（25 个分片）
1000 MB: ✓ 成功（50 个分片，达到限制）
1100 MB: ✗ 失败（超过 50 分片限制）
```

---

## 8. 故障排查

### 8.1 常见问题和解决方案

| 问题 | 原因 | 解决方案 |
|-----|------|--------|
| D1 测试失败 | 数据库未配置 | 检查 wrangler.toml 中的 D1 绑定 |
| KV 测试失败 | KV 未配置 | 检查 wrangler.toml 中的 KV 绑定 |
| 文件生成超时 | 内存不足 | 减小测试文件大小 |
| Telegram 测试失败 | 网络连接问题 | 检查网络连接和 Telegram API 状态 |

### 8.2 调试技巧

```javascript
// 添加详细的日志记录
function debugLog(category, message, data = null) {
    const timestamp = new Date().toISOString();
    console.log(`[${timestamp}] [${category}] ${message}`);
    if (data) {
        console.log(JSON.stringify(data, null, 2));
    }
}

// 测试中使用
debugLog('D1', 'Writing metadata', {
    size: JSON.stringify(metadata).length,
    keys: Object.keys(metadata),
});
```

---

## 9. CI/CD 集成

### 9.1 GitHub Actions 工作流

创建 `.github/workflows/test-limits.yml`：

```yaml
name: Test File Limits

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v2
      
      - name: Setup Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm install
      
      - name: Run D1 tests
        run: npm run test:d1
      
      - name: Run KV tests
        run: npm run test:kv
      
      - name: Run Upload tests
        run: npm run test:upload
      
      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v2
        with:
          name: test-results
          path: test_results.json
```

---

## 10. 最佳实践

1. **定期运行测试**：至少每月运行一次完整测试套件
2. **监控趋势**：记录测试结果，观察性能变化
3. **在部署前测试**：在将代码部署到生产环境前运行测试
4. **记录基准**：为自己的服务器建立性能基准
5. **关注更新**：当 Cloudflare 更新服务时重新测试


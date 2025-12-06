# CloudFlare-ImgBed 完整部署指南

> 📖 本指南包含从零开始的完整部署步骤，特别包含新增的 Telegram Webhook 功能配置

## 📋 目录

1. [第一部分：前置准备](#第一部分前置准备)
2. [第二部分：环境变量配置](#第二部分环境变量配置)
3. [第三部分：部署流程](#第三部分部署流程)
4. [第四部分：Webhook 注册和配置](#第四部分webhook-注册和配置)
5. [第五部分：首次使用](#第五部分首次使用)
6. [第六部分：故障排除](#第六部分故障排除)
7. [第七部分：配置示例](#第七部分配置示例)
8. [第八部分：验证清单](#第八部分验证清单)

---

## 第一部分：前置准备

### 步骤 1：Cloudflare 账户和 Pages 项目设置

- [ ] 注册 Cloudflare 账户（如果还没有）
- [ ] 升级到 Cloudflare Pages Pro 计划（推荐，支持更多功能）
- [ ] 准备 GitHub 账户并 Fork 项目仓库
- [ ] 确保已安装 Node.js 18+ （本地测试用）

### 步骤 2：创建 Telegram Bot

1. 在 Telegram 中找到 `@BotFather`
2. 发送 `/newbot` 创建新机器人
3. 按照提示设置机器人名称和用户名
4. 保存获得的 Bot Token（格式：`1234567890:ABCdefGHIjklMNOpqrsTUVwxyz`）

> ⚠️ **重要提示：**请妥善保存 Bot Token，泄露后任何人都可以控制你的机器人。

### 步骤 3：创建 Telegram 频道并获取 Chat ID

1. 创建一个 Telegram 频道（私有或公开均可）
2. 将创建的 Bot 添加为频道管理员
3. 获取频道的 Chat ID：

#### 方法一：使用 @userinfobot
- 将 @userinfobot 添加到频道
- 它会显示频道的 Chat ID（格式：`-1001234567890`）

#### 方法二：通过 Bot API 获取
- 在频道中发送任意消息
- 访问：`https://api.telegram.org/bot<BOT_TOKEN>/getUpdates`
- 在返回的 JSON 中找到 `chat.id` 字段

### 步骤 4：生成 Webhook Secret

为提高安全性，建议生成一个随机字符串作为 Webhook Secret：

```bash
# 使用 OpenSSL 生成
openssl rand -hex 32

# 或者使用在线工具生成
# 访问 https://www.uuidgenerator.net/ 或类似网站
```

> 💡 **提示：**Webhook Secret 用于验证 Webhook 请求的真实性，防止恶意请求。

---

## 第二部分：环境变量配置

> ⚠️ **v2.0 版本重要变更：**
> 新版本所有设置项已迁移至管理端系统设置界面，原则上无需再通过环境变量方式设置。但为了保证 Telegram 渠道图片与旧版本兼容，若之前设置了 Telegram 相关环境变量，请将其保留！

### 必需的环境变量

| 变量名 | 说明 | 获取方式 | 是否必需 | 示例值 |
|--------|------|----------|----------|--------|
| `AUTH_CODE` | 上传验证码 | 自定义 | **必需** | `mysecret123` |
| `BASIC_USER` | 管理后台用户名 | 自定义 | **必需** | `admin` |
| `BASIC_PASS` | 管理后台密码 | 自定义 | **必需** | `password123` |
| `TG_BOT_TOKEN` | Telegram 上传 Bot Token | @BotFather | **必需** | `123456:ABCdefGHIjklMNOpqrsTUVwxyz` |
| `TG_CHAT_ID` | Telegram 上传频道 ID | @userinfobot 或 API | **必需** | `-1001234567890` |

### Telegram Webhook 相关变量（新增功能）

| 变量名 | 说明 | 获取方式 | 是否必需 | 示例值 |
|--------|------|----------|----------|--------|
| `TELEGRAM_LISTENER_BOT_TOKEN` | Webhook 监听 Bot Token | @BotFather（可复用上传Bot） | 可选 | `123456:ABCdefGHIjklMNOpqrsTUVwxyz` |
| `TELEGRAM_LISTENER_CHAT_ID` | 监听的频道 Chat ID | @userinfobot 或 API | 可选 | `-1001234567890` |
| `TELEGRAM_WEBHOOK_SECRET` | Webhook 验证密钥 | 自定义随机字符串 | 可选 | `abc123def456ghi789` |

### 存储配置变量

| 变量名 | 说明 | 获取方式 | 是否必需 | 示例值 |
|--------|------|----------|----------|--------|
| `R2PublicUrl` | R2 存储公开访问 URL | Cloudflare R2 控制台 | R2存储时必需 | `https://pub-xxx.r2.dev` |
| `S3_ACCESS_KEY_ID` | S3 访问密钥 ID | S3 服务商控制台 | S3存储时必需 | `AKIAIOSFODNN7EXAMPLE` |
| `S3_SECRET_ACCESS_KEY` | S3 访问密钥 | S3 服务商控制台 | S3存储时必需 | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| `S3_BUCKET_NAME` | S3 存储桶名称 | S3 服务商控制台 | S3存储时必需 | `my-image-bucket` |
| `S3_ENDPOINT` | S3 服务端点 | S3 服务商文档 | S3存储时必需 | `https://s3.amazonaws.com` |
| `S3_REGION` | S3 存储区域 | S3 服务商控制台 | 可选 | `us-east-1` |
| `S3_PATH_STYLE` | 是否使用路径样式 | 根据服务商要求 | 可选 | `true` |

### 其他可选变量

| 变量名 | 说明 | 默认值 | 是否必需 |
|--------|------|--------|----------|
| `CF_ZONE_ID` | Cloudflare Zone ID | 空 | 可选 |
| `CF_EMAIL` | Cloudflare 账户邮箱 | 空 | 可选 |
| `CF_API_KEY` | Cloudflare API Key | 空 | 可选 |
| `ALLOWED_DOMAINS` | 允许访问的域名 | 空（允许所有） | 可选 |
| `AllowRandom` | 是否启用随机图 API | `false` | 可选 |
| `disable_telemetry` | 是否禁用遥测 | `false` | 可选 |

---

## 第三部分：部署流程

### 步骤 1：Fork 仓库到 GitHub

1. 访问项目 GitHub 页面
2. 点击右上角的 "Fork" 按钮
3. 选择要 Fork 到的账户
4. 等待 Fork 完成

### 步骤 2：创建 Cloudflare Pages 项目

1. 登录 Cloudflare Dashboard
2. 进入 "Pages" 部分
3. 点击 "Create application"
4. 选择 "Connect to Git"
5. 授权 Cloudflare 访问你的 GitHub
6. 选择刚刚 Fork 的仓库

### 步骤 3：配置构建设置

> ⚠️ **v2.0 版本重要变更：**
> 构建命令已改为 `npm install`，请务必更新！

#### 基本构建设置：
- **框架预设：** None
- **构建命令：** `npm install`
- **构建输出目录：** `./`
- **根目录：** `/`

#### 环境变量设置：
1. 在 "Environment variables" 部分添加所有必需的环境变量
2. 确保敏感信息（如 Bot Token）设置正确
3. 点击 "Save and Deploy"

### 步骤 4：配置 Cloudflare 绑定

#### KV 命名空间绑定：
1. 进入 Cloudflare Dashboard → Workers & Pages
2. 选择你的 Pages 项目
3. 进入 "Settings" → "Variables"
4. 在 "KV namespace bindings" 部分：
   - 变量名：`img_url`
   - KV 命名空间：创建新的 KV 命名空间或选择现有

#### R2 存储绑定（可选）：
1. 在 "R2 bucket bindings" 部分：
   - 变量名：`img_r2`
   - R2 存储桶：创建新的 R2 存储桶或选择现有

#### D1 数据库绑定（可选）：
1. 在 "D1 database bindings" 部分：
   - 数据库名：创建新的 D1 数据库
   - 初始化数据库：执行 `database/init.sql` 脚本

### 步骤 5：部署和验证

1. 点击 "Save and Deploy" 开始部署
2. 等待部署完成（通常需要 1-3 分钟）
3. 访问生成的域名查看项目
4. 测试基本功能是否正常

---

## 第四部分：Webhook 注册和配置

> 📌 **注意：**
> Webhook 功能是新增的，用于自动监听 Telegram 频道中的文件并导入到图床。

### 步骤 1：通过管理后台配置（推荐）

1. 访问管理后台：`https://your-domain.com/manage`
2. 使用环境变量中设置的用户名和密码登录
3. 进入 "系统设置" → "其他设置"
4. 找到 "Telegram Webhook 配置" 部分
5. 填写配置信息：

| 配置项 | 说明 | 示例 |
|--------|------|------|
| Bot Token | Webhook 监听机器人的 Token | `123456:ABCdefGHIjklMNOpqrsTUVwxyz` |
| Chat ID | 要监听的频道 ID | `-1001234567890` |
| Webhook Secret | 验证密钥 | `abc123def456ghi789` |

6. 勾选 "启用 Webhook"
7. 点击 "保存配置"
8. 点击 "注册 Webhook" 按钮
9. 等待注册完成并查看状态

### 步骤 2：通过 API 配置

如果更喜欢使用 API，可以通过以下命令注册 Webhook：

```bash
curl -X POST https://your-domain.com/api/manage/webhook/telegram \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-domain.com/webhook/telegram",
    "botToken": "YOUR_BOT_TOKEN",
    "chatId": "YOUR_CHAT_ID",
    "webhookSecret": "YOUR_SECRET",
    "enabled": true
  }'
```

### 步骤 3：验证 Webhook 状态

检查 Webhook 是否注册成功：

```bash
curl -X GET https://your-domain.com/api/manage/webhook/telegram \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)"
```

成功响应应该包含：

```json
{
  "success": true,
  "configured": true,
  "webhook": {
    "url": "https://your-domain.com/webhook/telegram",
    "pendingUpdateCount": 0,
    "lastErrorDate": null,
    "lastErrorMessage": ""
  }
}
```

---

## 第五部分：首次使用

### 步骤 1：测试基本上传功能

1. 访问主页：`https://your-domain.com`
2. 输入验证码（`AUTH_CODE` 环境变量的值）
3. 选择文件并上传
4. 确认文件上传成功并获取访问链接

### 步骤 2：测试 Webhook 自动导入

1. 在配置的 Telegram 频道中发送任意图片、文件或视频
2. 稍等片刻（通常 1-5 秒）
3. 访问管理后台查看文件列表
4. 确认文件出现在 `webhook_imported/` 目录下
5. 测试文件是否可以正常访问和下载

### 步骤 3：测试文件管理功能

1. 在管理后台中尝试重命名文件
2. 测试移动文件到不同目录
3. 测试批量操作功能
4. 验证 WebDAV 访问（如果已启用）

### 步骤 4：配置系统设置

1. 进入 "系统设置" 页面
2. 配置上传渠道（Telegram、R2、S3）
3. 设置安全选项（访问控制、域名白名单等）
4. 启用或禁用随机图 API
5. 配置 WebDAV（如需要）

---

## 第六部分：故障排除

### 认证失败问题

❌ **问题：**无法访问管理后台或上传时提示验证码错误

✅ **解决方案：**
- 检查环境变量 `AUTH_CODE`、`BASIC_USER`、`BASIC_PASS` 是否正确设置
- 确保环境变量名称没有拼写错误
- 重新部署项目使环境变量生效
- 清除浏览器缓存和 Cookie

### Webhook 连接失败

❌ **问题：**Webhook 注册失败或状态显示错误

✅ **解决方案：**
- 确认 Bot Token 是否正确且有效
- 确保 Bot 已被添加为频道管理员
- 检查 Chat ID 格式是否正确（负数格式）
- 验证 Webhook Secret 是否匹配
- 查看 Cloudflare Workers 日志获取详细错误信息
- 尝试重新注册 Webhook

### 文件导入失败

❌ **问题：**Telegram 频道中的文件没有自动导入

✅ **解决方案：**
- 检查 Webhook 状态是否正常
- 确认消息类型是否支持（只监听频道消息）
- 检查文件大小是否超过 Telegram 限制（20MB 或 2GB）
- 查看 Cloudflare Workers 实时日志
- 确认 KV 命名空间绑定是否正确

### 下载错误

❌ **问题：**文件无法正常访问或下载

✅ **解决方案：**
- 检查文件存储渠道是否正常工作
- 确认 R2 或 S3 配置是否正确
- 验证 Telegram Bot Token 是否有效
- 检查 CDN 缓存设置
- 确认文件是否存在于数据库中

### 后台管理面板访问问题

❌ **问题：**管理后台无法加载或功能异常

✅ **解决方案：**
- 检查浏览器控制台是否有 JavaScript 错误
- 确认所有必需的环境变量已设置
- 验证 KV 命名空间绑定是否正确
- 尝试重新部署项目
- 检查 Cloudflare Pages 的部署日志

---

## 第七部分：配置示例

### 环境变量完整示例

```bash
# 基础认证
AUTH_CODE=mysecret123
BASIC_USER=admin
BASIC_PASS=password123

# Telegram 上传渠道
TG_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
TG_CHAT_ID=-1001234567890

# Telegram Webhook 监听（新增功能）
TELEGRAM_LISTENER_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
TELEGRAM_LISTENER_CHAT_ID=-1001234567890
TELEGRAM_WEBHOOK_SECRET=abc123def456ghi789

# Cloudflare R2 存储（可选）
R2PublicUrl=https://pub-abc123.r2.dev

# S3 兼容存储（可选）
S3_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
S3_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
S3_BUCKET_NAME=my-image-bucket
S3_ENDPOINT=https://s3.amazonaws.com
S3_REGION=us-east-1
S3_PATH_STYLE=true

# Cloudflare API（可选）
CF_ZONE_ID=abc123def456ghi789
CF_EMAIL=your-email@example.com
CF_API_KEY=your-global-api-key

# 其他配置
ALLOWED_DOMAINS=example.com,yourdomain.com
AllowRandom=true
disable_telemetry=false
```

### 不同场景的配置

#### 场景一：仅使用 Telegram 存储

```bash
# 最小配置
AUTH_CODE=mysecret123
BASIC_USER=admin
BASIC_PASS=password123
TG_BOT_TOKEN=123456:ABCdefGHIjklMNOpqrsTUVwxyz
TG_CHAT_ID=-1001234567890
```

#### 场景二：Telegram + Webhook 监听

```bash
# 基础配置
AUTH_CODE=mysecret123
BASIC_USER=admin
BASIC_PASS=password123
TG_BOT_TOKEN=123456:ABCdefGHIjklMNOpqrsTUVwxyz
TG_CHAT_ID=-1001234567890

# Webhook 监听
TELEGRAM_LISTENER_BOT_TOKEN=123456:ABCdefGHIjklMNOpqrsTUVwxyz
TELEGRAM_LISTENER_CHAT_ID=-1009876543210
TELEGRAM_WEBHOOK_SECRET=random123secret456
```

#### 场景三：Telegram + R2 存储

```bash
# 完整配置
AUTH_CODE=mysecret123
BASIC_USER=admin
BASIC_PASS=password123
TG_BOT_TOKEN=123456:ABCdefGHIjklMNOpqrsTUVwxyz
TG_CHAT_ID=-1001234567890
R2PublicUrl=https://pub-abc123.r2.dev

# Webhook 监听
TELEGRAM_LISTENER_BOT_TOKEN=123456:ABCdefGHIjklMNOpqrsTUVwxyz
TELEGRAM_LISTENER_CHAT_ID=-1001234567890
TELEGRAM_WEBHOOK_SECRET=abc123def456ghi789
```

#### 场景四：S3 存储 + Webhook

```bash
# S3 配置
AUTH_CODE=mysecret123
BASIC_USER=admin
BASIC_PASS=password123
S3_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
S3_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
S3_BUCKET_NAME=my-image-bucket
S3_ENDPOINT=https://s3.amazonaws.com
S3_REGION=us-east-1

# Webhook 监听
TELEGRAM_LISTENER_BOT_TOKEN=123456:ABCdefGHIjklMNOpqrsTUVwxyz
TELEGRAM_LISTENER_CHAT_ID=-1001234567890
TELEGRAM_WEBHOOK_SECRET=abc123def456ghi789
```

---

## 第八部分：验证清单

### 部署前检查清单

- [ ] 已 Fork 项目到自己的 GitHub 账户
- [ ] 已创建 Telegram Bot 并获取 Token
- [ ] 已创建 Telegram 频道并获取 Chat ID
- [ ] 已生成 Webhook Secret（可选但推荐）
- [ ] 已准备好所有必需的环境变量
- [ ] 已创建 Cloudflare KV 命名空间
- [ ] 已创建 R2 存储桶（如需要）
- [ ] 已创建 D1 数据库（如需要）

### 部署后验证清单

- [ ] 项目部署成功，主页可以正常访问
- [ ] 管理后台可以正常登录（/manage）
- [ ] 文件上传功能正常工作
- [ ] 文件可以正常访问和下载
- [ ] 环境变量配置正确生效
- [ ] KV 数据存储正常工作
- [ ] 存储渠道（Telegram/R2/S3）正常工作

### Webhook 功能验证清单

- [ ] Webhook 相关环境变量已设置
- [ ] Webhook 配置已在后台保存
- [ ] Webhook 注册成功，状态正常
- [ ] 频道中的文件可以自动导入
- [ ] 导入的文件可以在管理后台看到
- [ ] 导入的文件可以正常访问和下载
- [ ] 文件重命名功能正常工作
- [ ] 批量操作功能正常工作

### 常用测试命令和端点

| 功能 | 端点 | 方法 | 说明 |
|------|------|------|------|
| 主页 | `/` | GET | 上传界面 |
| 管理后台 | `/manage` | GET | 管理界面 |
| 文件访问 | `/file/[fileId]` | GET | 文件下载 |
| Webhook 端点 | `/webhook/telegram` | POST | Telegram 回调 |
| Webhook 状态 | `/api/manage/webhook/telegram` | GET | 查看 Webhook 状态 |
| Webhook 统计 | `/api/manage/webhook/stats` | GET | 导入统计信息 |
| 文件列表 | `/api/manage/list` | GET | 列出所有文件 |
| 系统配置 | `/api/manage/sysConfig/*` | GET/POST | 系统设置 |

---

## 🎉 部署完成！

恭喜！您已经成功部署了 CloudFlare-ImgBed 项目并配置了 Telegram Webhook 功能。现在您可以：

- 通过网页界面上传文件
- 通过 Telegram 频道自动导入文件
- 在管理后台管理所有文件
- 使用 WebDAV 访问文件
- 通过 API 进行批量操作

如果遇到问题，请参考本文档的故障排除部分或查看项目 Wiki。

## 📚 相关文档

- [项目主文档](./README.md)
- [Webhook 功能说明](./WEBHOOK_FEATURE.md)
- [Webhook API 文档](./WEBHOOK_API_DOCUMENTATION.md)
- [官方文档网站](https://cfbed.sanyue.de)

---

**CloudFlare-ImgBed 部署指南**

版本：v2.0 | 更新时间：2024-12

如有问题，请提交 [Issue](https://github.com/MarSeventh/CloudFlare-ImgBed/issues)

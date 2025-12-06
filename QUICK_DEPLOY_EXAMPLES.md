># CloudFlare-ImgBed 快速部署配置示例</h3>

<div class="warning">
  <strong>⚠️ 重要提示：</strong>
  这些是示例配置，请根据您的实际情况修改相应的值！
</div>

## 基础配置（仅 Telegram 存储）

```bash
# 最小必需配置
AUTH_CODE=mysecret123
BASIC_USER=admin
BASIC_PASS=password123
TG_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
TG_CHAT_ID=-1001234567890
```

## 完整配置（Telegram + Webhook 监听）

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
```

## R2 存储配置

```bash
# 基础配置
AUTH_CODE=mysecret123
BASIC_USER=admin
BASIC_PASS=password123
TG_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
TG_CHAT_ID=-1001234567890

# R2 存储
R2PublicUrl=https://pub-abc123def456.r2.dev

# Webhook 监听
TELEGRAM_LISTENER_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
TELEGRAM_LISTENER_CHAT_ID=-1001234567890
TELEGRAM_WEBHOOK_SECRET=abc123def456ghi789
```

## S3 兼容存储配置

```bash
# 基础配置
AUTH_CODE=mysecret123
BASIC_USER=admin
BASIC_PASS=password123

# S3 配置
S3_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
S3_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
S3_BUCKET_NAME=my-image-bucket
S3_ENDPOINT=https://s3.amazonaws.com
S3_REGION=us-east-1
S3_PATH_STYLE=true

# Webhook 监听
TELEGRAM_LISTENER_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
TELEGRAM_LISTENER_CHAT_ID=-1001234567890
TELEGRAM_WEBHOOK_SECRET=abc123def456ghi789
```

## 高级配置（包含所有选项）

```bash
# 基础认证
AUTH_CODE=mysecret123
BASIC_USER=admin
BASIC_PASS=password123

# Telegram 上传渠道
TG_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
TG_CHAT_ID=-1001234567890

# Telegram Webhook 监听
TELEGRAM_LISTENER_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
TELEGRAM_LISTENER_CHAT_ID=-1001234567890
TELEGRAM_WEBHOOK_SECRET=abc123def456ghi789

# Cloudflare R2 存储
R2PublicUrl=https://pub-abc123def456.r2.dev

# S3 兼容存储
S3_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
S3_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
S3_BUCKET_NAME=my-image-bucket
S3_ENDPOINT=https://s3.amazonaws.com
S3_REGION=us-east-1
S3_PATH_STYLE=true

# Cloudflare API
CF_ZONE_ID=abc123def456ghi789
CF_EMAIL=your-email@example.com
CF_API_KEY=your-global-api-key

# 其他配置
ALLOWED_DOMAINS=example.com,yourdomain.com
AllowRandom=true
disable_telemetry=false
```

## 📋 配置说明

### 必需变量

| 变量名 | 说明 | 示例值 |
|--------|------|--------|
| `AUTH_CODE` | 上传验证码 | `mysecret123` |
| `BASIC_USER` | 管理后台用户名 | `admin` |
| `BASIC_PASS` | 管理后台密码 | `password123` |
| `TG_BOT_TOKEN` | Telegram Bot Token | `123456:ABCdefGHIjklMNOpqrsTUVwxyz` |
| `TG_CHAT_ID` | Telegram 频道 ID | `-1001234567890` |

### Webhook 变量（新增功能）

| 变量名 | 说明 | 示例值 |
|--------|------|--------|
| `TELEGRAM_LISTENER_BOT_TOKEN` | Webhook 监听 Bot Token | `123456:ABCdefGHIjklMNOpqrsTUVwxyz` |
| `TELEGRAM_LISTENER_CHAT_ID` | 监听的频道 ID | `-1001234567890` |
| `TELEGRAM_WEBHOOK_SECRET` | Webhook 验证密钥 | `abc123def456ghi789` |

### 存储变量

| 变量名 | 说明 | 示例值 |
|--------|------|--------|
| `R2PublicUrl` | R2 公开访问 URL | `https://pub-abc123.r2.dev` |
| `S3_ACCESS_KEY_ID` | S3 访问密钥 ID | `AKIAIOSFODNN7EXAMPLE` |
| `S3_SECRET_ACCESS_KEY` | S3 访问密钥 | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| `S3_BUCKET_NAME` | S3 存储桶名称 | `my-image-bucket` |
| `S3_ENDPOINT` | S3 服务端点 | `https://s3.amazonaws.com` |
| `S3_REGION` | S3 存储区域 | `us-east-1` |
| `S3_PATH_STYLE` | 路径样式访问 | `true` |

## 🚀 快速部署步骤

1. **复制配置**：选择适合您需求的配置示例
2. **修改值**：将示例值替换为您的实际配置
3. **设置环境变量**：在 Cloudflare Pages 中添加环境变量
4. **配置绑定**：设置 KV、R2、D1 等绑定
5. **部署项目**：点击保存并部署
6. **测试功能**：验证上传和 Webhook 功能

## 🔧 获取配置值的方法

### Telegram Bot Token
1. 与 @BotFather 对话
2. 发送 `/newbot`
3. 按提示创建机器人
4. 保存获得的 Token

### Telegram Chat ID
1. 将 @userinfobot 添加到频道
2. 查看显示的 Chat ID
3. 或发送消息后访问：`https://api.telegram.org/bot<TOKEN>/getUpdates`

### R2 Public URL
1. 登录 Cloudflare Dashboard
2. 进入 R2 Object Storage
3. 创建存储桶后查看 Public URL

### S3 配置
1. 登录您的 S3 服务商控制台
2. 创建存储桶
3. 生成访问密钥
4. 记录相关信息

### Webhook Secret
```bash
# 生成随机密钥
openssl rand -hex 32
```

## ⚠️ 安全提醒

- **不要**将包含敏感信息的配置文件提交到公共仓库
- **定期**更换 Bot Token 和密钥
- **使用**强密码和随机字符串
- **限制**访问域名（使用 `ALLOWED_DOMAINS`）
- **启用**HTTPS（Cloudflare 自动提供）

---

如有问题，请参考 [完整部署指南](./DEPLOYMENT_GUIDE.md) 或提交 [Issue](https://github.com/MarSeventh/CloudFlare-ImgBed/issues)。

# CloudFlare-ImgBed Complete Deployment Guide

> 📖 This guide provides complete step-by-step deployment instructions, including the new Telegram Webhook monitoring feature

## 📋 Table of Contents

1. [Part 1: Prerequisites](#part-1-prerequisites)
2. [Part 2: Environment Variables Configuration](#part-2-environment-variables-configuration)
3. [Part 3: Deployment Process](#part-3-deployment-process)
4. [Part 4: Webhook Registration and Configuration](#part-4-webhook-registration-and-configuration)
5. [Part 5: First Use](#part-5-first-use)
6. [Part 6: Troubleshooting](#part-6-troubleshooting)
7. [Part 7: Configuration Examples](#part-7-configuration-examples)
8. [Part 8: Verification Checklist](#part-8-verification-checklist)

---

## Part 1: Prerequisites

### Step 1: Cloudflare Account and Pages Project Setup

- [ ] Register a Cloudflare account (if you don't have one)
- [ ] Upgrade to Cloudflare Pages Pro plan (recommended for more features)
- [ ] Prepare GitHub account and Fork project repository
- [ ] Ensure Node.js 18+ is installed (for local testing)

### Step 2: Create Telegram Bot

1. Find `@BotFather` in Telegram
2. Send `/newbot` to create a new bot
3. Follow the prompts to set bot name and username
4. Save the obtained Bot Token (format: `1234567890:ABCdefGHIjklMNOpqrsTUVwxyz`)

> ⚠️ **Important Notice:** Please keep your Bot Token secure. If leaked, anyone can control your bot.

### Step 3: Create Telegram Channel and Get Chat ID

1. Create a Telegram channel (private or public)
2. Add the created bot as a channel administrator
3. Get the channel's Chat ID:

#### Method 1: Using @userinfobot
- Add @userinfobot to the channel
- It will display the channel's Chat ID (format: `-1001234567890`)

#### Method 2: Get via Bot API
- Send any message to the channel
- Visit: `https://api.telegram.org/bot<BOT_TOKEN>/getUpdates`
- Find the `chat.id` field in the returned JSON

### Step 4: Generate Webhook Secret

For enhanced security, it's recommended to generate a random string as the Webhook Secret:

```bash
# Generate using OpenSSL
openssl rand -hex 32

# Or use online tools
# Visit https://www.uuidgenerator.net/ or similar websites
```

> 💡 **Tip:** Webhook Secret is used to verify the authenticity of Webhook requests and prevent malicious requests.

---

## Part 2: Environment Variables Configuration

> ⚠️ **v2.0 Version Important Change:**
> All settings in the new version have been migrated to the admin panel system settings interface. In principle, no longer need to set through environment variables. However, to ensure compatibility with Telegram channel images from older versions, if you previously set Telegram-related environment variables, please keep them!

### Required Environment Variables

| Variable Name | Description | How to Get | Required | Example Value |
|---------------|-------------|------------|----------|---------------|
| `AUTH_CODE` | Upload verification code | Custom | **Required** | `mysecret123` |
| `BASIC_USER` | Admin backend username | Custom | **Required** | `admin` |
| `BASIC_PASS` | Admin backend password | Custom | **Required** | `password123` |
| `TG_BOT_TOKEN` | Telegram upload Bot Token | @BotFather | **Required** | `123456:ABCdefGHIjklMNOpqrsTUVwxyz` |
| `TG_CHAT_ID` | Telegram upload channel ID | @userinfobot or API | **Required** | `-1001234567890` |

### Telegram Webhook Related Variables (New Feature)

| Variable Name | Description | How to Get | Required | Example Value |
|---------------|-------------|------------|----------|---------------|
| `TELEGRAM_LISTENER_BOT_TOKEN` | Webhook listener Bot Token | @BotFather (can reuse upload bot) | Optional | `123456:ABCdefGHIjklMNOpqrsTUVwxyz` |
| `TELEGRAM_LISTENER_CHAT_ID` | Channel Chat ID to monitor | @userinfobot or API | Optional | `-1001234567890` |
| `TELEGRAM_WEBHOOK_SECRET` | Webhook verification secret | Custom random string | Optional | `abc123def456ghi789` |

### Storage Configuration Variables

| Variable Name | Description | How to Get | Required | Example Value |
|---------------|-------------|------------|----------|---------------|
| `R2PublicUrl` | R2 storage public access URL | Cloudflare R2 console | Required for R2 storage | `https://pub-xxx.r2.dev` |
| `S3_ACCESS_KEY_ID` | S3 access key ID | S3 provider console | Required for S3 storage | `AKIAIOSFODNN7EXAMPLE` |
| `S3_SECRET_ACCESS_KEY` | S3 access key | S3 provider console | Required for S3 storage | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| `S3_BUCKET_NAME` | S3 bucket name | S3 provider console | Required for S3 storage | `my-image-bucket` |
| `S3_ENDPOINT` | S3 service endpoint | S3 provider documentation | Required for S3 storage | `https://s3.amazonaws.com` |
| `S3_REGION` | S3 storage region | S3 provider console | Optional | `us-east-1` |
| `S3_PATH_STYLE` | Whether to use path style | Based on provider requirements | Optional | `true` |

### Other Optional Variables

| Variable Name | Description | Default Value | Required |
|---------------|-------------|---------------|----------|
| `CF_ZONE_ID` | Cloudflare Zone ID | Empty | Optional |
| `CF_EMAIL` | Cloudflare account email | Empty | Optional |
| `CF_API_KEY` | Cloudflare API Key | Empty | Optional |
| `ALLOWED_DOMAINS` | Allowed access domains | Empty (allow all) | Optional |
| `AllowRandom` | Whether to enable random image API | `false` | Optional |
| `disable_telemetry` | Whether to disable telemetry | `false` | Optional |

---

## Part 3: Deployment Process

### Step 1: Fork Repository to GitHub

1. Visit the project GitHub page
2. Click the "Fork" button in the top right
3. Choose the account to fork to
4. Wait for the fork to complete

### Step 2: Create Cloudflare Pages Project

1. Log in to Cloudflare Dashboard
2. Go to the "Pages" section
3. Click "Create application"
4. Select "Connect to Git"
5. Authorize Cloudflare to access your GitHub
6. Select the just-forked repository

### Step 3: Configure Build Settings

> ⚠️ **v2.0 Version Important Change:**
> Build command has been changed to `npm install`, please update accordingly!

#### Basic Build Settings:
- **Framework preset:** None
- **Build command:** `npm install`
- **Build output directory:** `./`
- **Root directory:** `/`

#### Environment Variable Settings:
1. In the "Environment variables" section, add all required environment variables
2. Ensure sensitive information (like Bot Token) is set correctly
3. Click "Save and Deploy"

### Step 4: Configure Cloudflare Bindings

#### KV Namespace Binding:
1. Go to Cloudflare Dashboard → Workers & Pages
2. Select your Pages project
3. Go to "Settings" → "Variables"
4. In the "KV namespace bindings" section:
   - Variable name: `img_url`
   - KV namespace: Create new KV namespace or select existing

#### R2 Storage Binding (Optional):
1. In the "R2 bucket bindings" section:
   - Variable name: `img_r2`
   - R2 bucket: Create new R2 bucket or select existing

#### D1 Database Binding (Optional):
1. In the "D1 database bindings" section:
   - Database name: Create new D1 database
   - Initialize database: Execute `database/init.sql` script

### Step 5: Deployment and Verification

1. Click "Save and Deploy" to start deployment
2. Wait for deployment to complete (usually takes 1-3 minutes)
3. Visit the generated domain to view the project
4. Test basic functionality

---

## Part 4: Webhook Registration and Configuration

> 📌 **Note:**
> Webhook functionality is newly added for automatically monitoring files in Telegram channels and importing them to the image bed.

### Step 1: Configure via Admin Panel (Recommended)

1. Visit admin panel: `https://your-domain.com/manage`
2. Log in with the username and password set in environment variables
3. Go to "System Settings" → "Other Settings"
4. Find the "Telegram Webhook Configuration" section
5. Fill in configuration information:

| Configuration Item | Description | Example |
|-------------------|-------------|---------|
| Bot Token | Webhook listener bot's Token | `123456:ABCdefGHIjklMNOpqrsTUVwxyz` |
| Chat ID | Channel ID to monitor | `-1001234567890` |
| Webhook Secret | Verification secret | `abc123def456ghi789` |

6. Check "Enable Webhook"
7. Click "Save Configuration"
8. Click "Register Webhook" button
9. Wait for registration to complete and check status

### Step 2: Configure via API

If you prefer using API, you can register Webhook with the following command:

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

### Step 3: Verify Webhook Status

Check if Webhook registration was successful:

```bash
curl -X GET https://your-domain.com/api/manage/webhook/telegram \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)"
```

Successful response should include:

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

## Part 5: First Use

### Step 1: Test Basic Upload Functionality

1. Visit homepage: `https://your-domain.com`
2. Enter verification code (value of `AUTH_CODE` environment variable)
3. Select file and upload
4. Confirm file upload is successful and get access link

### Step 2: Test Webhook Auto Import

1. Send any image, file, or video to the configured Telegram channel
2. Wait a moment (usually 1-5 seconds)
3. Visit admin panel to view file list
4. Confirm files appear in the `webhook_imported/` directory
5. Test if files can be accessed and downloaded normally

### Step 3: Test File Management Functions

1. Try renaming files in the admin panel
2. Test moving files to different directories
3. Test batch operations
4. Verify WebDAV access (if enabled)

### Step 4: Configure System Settings

1. Go to "System Settings" page
2. Configure upload channels (Telegram, R2, S3)
3. Set security options (access control, domain whitelist, etc.)
4. Enable or disable random image API
5. Configure WebDAV (if needed)

---

## Part 6: Troubleshooting

### Authentication Failure Issues

❌ **Problem:** Cannot access admin backend or verification code error during upload

✅ **Solution:**
- Check if environment variables `AUTH_CODE`, `BASIC_USER`, `BASIC_PASS` are set correctly
- Ensure environment variable names have no spelling errors
- Redeploy project to make environment variables take effect
- Clear browser cache and cookies

### Webhook Connection Failure

❌ **Problem:** Webhook registration failed or status shows error

✅ **Solution:**
- Confirm Bot Token is correct and valid
- Ensure Bot has been added as channel administrator
- Check if Chat ID format is correct (negative number format)
- Verify Webhook Secret matches
- Check Cloudflare Workers logs for detailed error information
- Try re-registering Webhook

### File Import Failure

❌ **Problem:** Files in Telegram channel are not automatically imported

✅ **Solution:**
- Check if Webhook status is normal
- Confirm message type is supported (only monitors channel messages)
- Check if file size exceeds Telegram limit (20MB or 2GB)
- View Cloudflare Workers real-time logs
- Confirm KV namespace binding is correct

### Download Errors

❌ **Problem:** Files cannot be accessed or downloaded normally

✅ **Solution:**
- Check if file storage channel is working normally
- Confirm R2 or S3 configuration is correct
- Verify Telegram Bot Token is valid
- Check CDN cache settings
- Confirm files exist in database

### Admin Backend Access Issues

❌ **Problem:** Admin backend cannot load or has abnormal functionality

✅ **Solution:**
- Check browser console for JavaScript errors
- Confirm all required environment variables are set
- Verify KV namespace binding is correct
- Try redeploying project
- Check Cloudflare Pages deployment logs

---

## Part 7: Configuration Examples

### Complete Environment Variable Example

```bash
# Basic authentication
AUTH_CODE=mysecret123
BASIC_USER=admin
BASIC_PASS=password123

# Telegram upload channel
TG_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
TG_CHAT_ID=-1001234567890

# Telegram Webhook monitoring (new feature)
TELEGRAM_LISTENER_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
TELEGRAM_LISTENER_CHAT_ID=-1001234567890
TELEGRAM_WEBHOOK_SECRET=abc123def456ghi789

# Cloudflare R2 storage (optional)
R2PublicUrl=https://pub-abc123.r2.dev

# S3 compatible storage (optional)
S3_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
S3_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
S3_BUCKET_NAME=my-image-bucket
S3_ENDPOINT=https://s3.amazonaws.com
S3_REGION=us-east-1
S3_PATH_STYLE=true

# Cloudflare API (optional)
CF_ZONE_ID=abc123def456ghi789
CF_EMAIL=your-email@example.com
CF_API_KEY=your-global-api-key

# Other configurations
ALLOWED_DOMAINS=example.com,yourdomain.com
AllowRandom=true
disable_telemetry=false
```

### Different Scenario Configurations

#### Scenario 1: Telegram Storage Only

```bash
# Minimal configuration
AUTH_CODE=mysecret123
BASIC_USER=admin
BASIC_PASS=password123
TG_BOT_TOKEN=123456:ABCdefGHIjklMNOpqrsTUVwxyz
TG_CHAT_ID=-1001234567890
```

#### Scenario 2: Telegram + Webhook Monitoring

```bash
# Basic configuration
AUTH_CODE=mysecret123
BASIC_USER=admin
BASIC_PASS=password123
TG_BOT_TOKEN=123456:ABCdefGHIjklMNOpqrsTUVwxyz
TG_CHAT_ID=-1001234567890

# Webhook monitoring
TELEGRAM_LISTENER_BOT_TOKEN=123456:ABCdefGHIjklMNOpqrsTUVwxyz
TELEGRAM_LISTENER_CHAT_ID=-1009876543210
TELEGRAM_WEBHOOK_SECRET=random123secret456
```

#### Scenario 3: Telegram + R2 Storage

```bash
# Complete configuration
AUTH_CODE=mysecret123
BASIC_USER=admin
BASIC_PASS=password123
TG_BOT_TOKEN=123456:ABCdefGHIjklMNOpqrsTUVwxyz
TG_CHAT_ID=-1001234567890
R2PublicUrl=https://pub-abc123.r2.dev

# Webhook monitoring
TELEGRAM_LISTENER_BOT_TOKEN=123456:ABCdefGHIjklMNOpqrsTUVwxyz
TELEGRAM_LISTENER_CHAT_ID=-1001234567890
TELEGRAM_WEBHOOK_SECRET=abc123def456ghi789
```

#### Scenario 4: S3 Storage + Webhook

```bash
# S3 configuration
AUTH_CODE=mysecret123
BASIC_USER=admin
BASIC_PASS=password123
S3_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
S3_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
S3_BUCKET_NAME=my-image-bucket
S3_ENDPOINT=https://s3.amazonaws.com
S3_REGION=us-east-1

# Webhook monitoring
TELEGRAM_LISTENER_BOT_TOKEN=123456:ABCdefGHIjklMNOpqrsTUVwxyz
TELEGRAM_LISTENER_CHAT_ID=-1001234567890
TELEGRAM_WEBHOOK_SECRET=abc123def456ghi789
```

---

## Part 8: Verification Checklist

### Pre-deployment Checklist

- [ ] Project forked to own GitHub account
- [ ] Telegram Bot created and Token obtained
- [ ] Telegram channel created and Chat ID obtained
- [ ] Webhook Secret generated (optional but recommended)
- [ ] All required environment variables prepared
- [ ] Cloudflare KV namespace created
- [ ] R2 bucket created (if needed)
- [ ] D1 database created (if needed)

### Post-deployment Verification Checklist

- [ ] Project deployed successfully, homepage accessible
- [ ] Admin backend can be logged in (/manage)
- [ ] File upload functionality works normally
- [ ] Files can be accessed and downloaded normally
- [ ] Environment variable configuration takes effect correctly
- [ ] KV data storage works normally
- [ ] Storage channels (Telegram/R2/S3) work normally

### Webhook Functionality Verification Checklist

- [ ] Webhook related environment variables set
- [ ] Webhook configuration saved in backend
- [ ] Webhook registration successful, status normal
- [ ] Files in channel can be automatically imported
- [ ] Imported files visible in admin backend
- [ ] Imported files can be accessed and downloaded normally
- [ ] File rename functionality works normally
- [ ] Batch operations functionality works normally

### Common Test Commands and Endpoints

| Function | Endpoint | Method | Description |
|----------|----------|--------|-------------|
| Homepage | `/` | GET | Upload interface |
| Admin Backend | `/manage` | GET | Management interface |
| File Access | `/file/[fileId]` | GET | File download |
| Webhook Endpoint | `/webhook/telegram` | POST | Telegram callback |
| Webhook Status | `/api/manage/webhook/telegram` | GET | View Webhook status |
| Webhook Stats | `/api/manage/webhook/stats` | GET | Import statistics |
| File List | `/api/manage/list` | GET | List all files |
| System Config | `/api/manage/sysConfig/*` | GET/POST | System settings |

---

## 🎉 Deployment Complete!

Congratulations! You have successfully deployed the CloudFlare-ImgBed project and configured the Telegram Webhook functionality. Now you can:

- Upload files via web interface
- Automatically import files via Telegram channels
- Manage all files in the admin backend
- Access files via WebDAV
- Perform batch operations via API

If you encounter issues, please refer to the troubleshooting section of this guide or check the project Wiki.

## 📚 Related Documentation

- [Project Main Documentation](./README.md)
- [Webhook Feature Description](./WEBHOOK_FEATURE.md)
- [Webhook API Documentation](./WEBHOOK_API_DOCUMENTATION.md)
- [Official Documentation Website](https://cfbed.sanyue.de)

---

**CloudFlare-ImgBed Deployment Guide**

Version: v2.0 | Updated: 2024-12

If you have questions, please submit an [Issue](https://github.com/MarSeventh/CloudFlare-ImgBed/issues)

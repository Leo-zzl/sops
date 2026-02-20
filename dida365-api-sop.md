# 滴答清单 API 授权与任务创建 SOP

> **适用范围：** OpenClaw + 滴答清单（国内版 dida365.com）自动化任务管理  
> **创建日期：** 2026-02-20  
> **适用账号：** YOUR_EMAIL@example.com
> 
> ⚠️ **安全提示：** 本文档为公开版本，所有敏感信息（Client ID、Secret、Token、邮箱）已替换为占位符。使用时请替换为你自己的实际值！

---

## 📋 目录

1. [前置条件](#前置条件)
2. [核心概念](#核心概念)
3. [申请开发者权限](#申请开发者权限)
4. [获取授权 Code](#获取授权-code)
5. [换取 Access Token](#换取-access-token)
6. [创建任务](#创建任务)
7. [自动化脚本](#自动化脚本)
8. [常见问题](#常见问题)
9. [凭证管理](#凭证管理)

---

## 前置条件

### 必需账号
- [x] 滴答清单国内版账号（dida365.com）
- [x] 开发者权限申请入口访问权限

### 环境准备
```bash
# 安装 curl（如未安装）
apt-get install curl  # Ubuntu/Debian
brew install curl      # macOS
```

### 重要区分
| 版本 | 域名 | 说明 |
|------|------|------|
| **国内版** | `dida365.com` | ✅ 本 SOP 适用 |
| 国际版 | `ticktick.com` | ❌ 账号体系独立，不互通 |

**⚠️ 关键：** 国内版和国际版的开发者账号、Client ID、Access Token 完全独立，不能混用！

---

## 核心概念

```
┌─────────────────┐     申请      ┌─────────────────┐
│   滴答清单账号   │ ────────────→ │  开发者应用      │
│  (dida365.com)  │               │ Client ID/Secret │
└─────────────────┘               └─────────────────┘
          │                                │
          │ 登录授权                        │
          ▼                                ▼
┌─────────────────┐     换取      ┌─────────────────┐
│   授权 Code      │ ────────────→ │   Access Token   │
│  (临时，一次性)   │               │  (长期有效)      │
└─────────────────┘               └─────────────────┘
                                           │
                                           │ 调用 API
                                           ▼
                                    ┌─────────────────┐
                                    │   创建/管理任务  │
                                    │   CRUD 操作     │
                                    └─────────────────┘
```

---

## 申请开发者权限

### 步骤 1：访问开发者平台

打开浏览器，访问：
```
https://developer.dida365.com
```

使用你的滴答清单账号登录。

### 步骤 2：创建应用

1. 点击「创建应用」或「新建应用」
2. 填写应用信息：
   - **应用名称：** Aria助手（或其他）
   - **应用描述：** 自动化任务管理
   - **回调地址：** `http://localhost:3000/callback`

3. 保存后获取：
   - **Client ID：** `YOUR_CLIENT_ID_HERE`
   - **Client Secret：** `YOUR_CLIENT_SECRET_HERE`

**⚠️ 安全提示：** Client Secret 相当于密码，勿泄露！

---

## 获取授权 Code

### 步骤 3：生成授权链接

替换以下参数：
- `{CLIENT_ID}` → 你的 Client ID
- `{STATE}` → 随机字符串（防 CSRF）

```bash
CLIENT_ID="YOUR_CLIENT_ID_HERE"
REDIRECT_URI="http://localhost:3000/callback"
STATE="aria_$(date +%s)"

AUTH_URL="https://dida365.com/oauth/authorize?\
client_id=${CLIENT_ID}&\
response_type=code&\
scope=tasks:write%20tasks:read&\
state=${STATE}&\
redirect_uri=${REDIRECT_URI}"

echo "授权链接：$AUTH_URL"
```

### 步骤 4：用户授权

1. 浏览器打开生成的授权链接
2. 登录滴答清单账号（如未登录）
3. 点击「授权」按钮
4. 浏览器跳转到：
   ```
   http://localhost:3000/callback?code=AUTHORIZATION_CODE&state=aria_123456
   ```
5. **提取 `code` 参数：** `AUTHORIZATION_CODE`

**⚠️ 注意：** Code 有效期很短（通常几分钟），获取后需立即换取 Token。

---

## 换取 Access Token

### 步骤 5：调用 Token 接口

```bash
CLIENT_ID="YOUR_CLIENT_ID_HERE"
CLIENT_SECRET="YOUR_CLIENT_SECRET_HERE"
CODE="AUTHORIZATION_CODE"  # 上一步获取的 Code
REDIRECT_URI="http://localhost:3000/callback"

curl -X POST "https://dida365.com/oauth/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=${CLIENT_ID}" \
  -d "client_secret=${CLIENT_SECRET}" \
  -d "code=${CODE}" \
  -d "grant_type=authorization_code" \
  -d "redirect_uri=${REDIRECT_URI}"
```

### 步骤 6：保存返回的 Token

**成功响应：**
```json
{
  "access_token": "YOUR_ACCESS_TOKEN_HERE",
  "token_type": "bearer",
  "expires_in": 15551999,
  "scope": "tasks:read tasks:write"
}
```

**保存 Token：**
```bash
ACCESS_TOKEN="YOUR_ACCESS_TOKEN_HERE"
echo "DIDA365_ACCESS_TOKEN=${ACCESS_TOKEN}" > ~/.dida365.env
```

---

## 创建任务

### 步骤 7：调用创建任务 API

```bash
ACCESS_TOKEN="YOUR_ACCESS_TOKEN_HERE"
TODAY=$(date +%Y-%m-%d)

curl -X POST "https://api.dida365.com/open/v1/task" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"title\": \"🏃 STRYD跑步训练\",
    \"content\": \"按STRYD课表跑步，时长约1小时\",
    \"startDate\": \"${TODAY}T18:00:00+08:00\",
    \"dueDate\": \"${TODAY}T19:00:00+08:00\",
    \"isAllDay\": false,
    \"priority\": 3
  }"
```

### 任务字段说明

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `title` | string | 任务标题 | `"🏃 STRYD跑步训练"` |
| `content` | string | 任务描述 | `"按课表跑步..."` |
| `startDate` | ISO8601 | 开始时间 | `"2026-02-20T18:00:00+08:00"` |
| `dueDate` | ISO8601 | 截止时间 | `"2026-02-20T19:00:00+08:00"` |
| `isAllDay` | boolean | 是否全天 | `false` |
| `priority` | int | 优先级 0-5 | `3`（高优先级）|

---

## 自动化脚本

### 完整自动化脚本

保存为 `dida365-create-task.sh`：

```bash
#!/bin/bash
# 滴答清单自动创建任务脚本

# 配置
ACCESS_TOKEN="${DIDA365_ACCESS_TOKEN}"
TASK_TITLE="🏃 STRYD跑步训练"
TASK_CONTENT="按STRYD课表跑步，时长约1小时"
TASK_TIME="18:00"
TASK_DURATION="1"

# 检查 Token
if [ -z "$ACCESS_TOKEN" ]; then
    echo "❌ 错误：未设置 DIDA365_ACCESS_TOKEN 环境变量"
    exit 1
fi

# 计算时间
TODAY=$(date +%Y-%m-%d)
START_TIME="${TODAY}T${TASK_TIME}:00+08:00"
END_HOUR=$(date -d "$TASK_TIME + $TASK_DURATION hour" +%H)
END_TIME="${TODAY}T${END_HOUR}:00:00+08:00"

# 创建任务
echo "📝 创建任务：$TASK_TITLE"
echo "⏰ 时间：$START_TIME - $END_TIME"

RESPONSE=$(curl -s -X POST "https://api.dida365.com/open/v1/task" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"title\": \"${TASK_TITLE}\",
    \"content\": \"${TASK_CONTENT}\",
    \"startDate\": \"${START_TIME}\",
    \"dueDate\": \"${END_TIME}\",
    \"isAllDay\": false,
    \"priority\": 3
  }")

# 检查响应
if echo "$RESPONSE" | grep -q '"id"'; then
    TASK_ID=$(echo "$RESPONSE" | python3 -c "import json,sys; print(json.load(sys.stdin).get('id'))")
    echo "✅ 任务创建成功！ID: $TASK_ID"
else
    echo "❌ 创建失败：$RESPONSE"
    exit 1
fi
```

### 使用方式

```bash
# 1. 设置环境变量
export DIDA365_ACCESS_TOKEN="YOUR_ACCESS_TOKEN_HERE"

# 2. 运行脚本
chmod +x dida365-create-task.sh
./dida365-create-task.sh
```

---

## 常见问题

### Q1: 国际版和国内版 Token 能混用吗？

**不能！** 两个版本完全独立：
- 国际版：`ticktick.com`
- 国内版：`dida365.com`

错误使用会返回：
```json
{"error": "invalid_token", "error_description": "Invalid access token"}
```

### Q2: Code 过期了怎么办？

重新走授权流程（步骤 3-4），获取新的 Code。

### Q3: Access Token 有效期多久？

根据返回的 `expires_in` 字段（秒），通常约 180 天。过期后需重新授权。

### Q4: 如何设置每日重复任务？

滴答清单 API 对重复任务支持有限，建议：
1. 在 App 中手动设置重复
2. 或使用系统的 cron 定时创建

### Q5: 任务创建在哪里？

默认创建在「收件箱」(Inbox)，可在 App 中移动到具体清单。

---

## 凭证管理

### 安全存储建议

```bash
# 创建凭证文件
mkdir -p ~/.config/openclaw
cat > ~/.config/openclaw/dida365.env << 'EOF'
# 滴答清单国内版凭证
# 创建于：2026-02-20
# 账号：YOUR_EMAIL@example.com

CLIENT_ID="YOUR_CLIENT_ID_HERE"
CLIENT_SECRET="YOUR_CLIENT_SECRET_HERE"
ACCESS_TOKEN="YOUR_ACCESS_TOKEN_HERE"
EOF

# 设置权限（仅当前用户可读）
chmod 600 ~/.config/openclaw/dida365.env

# 加载凭证
source ~/.config/openclaw/dida365.env
```

### 备份建议

将凭证备份到密码管理器：
- 1Password
- Bitwarden
- 或其他安全存储

---

## API 参考

### 常用端点

| 操作 | 方法 | 端点 |
|------|------|------|
| 创建任务 | POST | `https://api.dida365.com/open/v1/task` |
| 获取任务 | GET | `https://api.dida365.com/open/v1/task/{taskId}` |
| 更新任务 | POST | `https://api.dida365.com/open/v1/task/{taskId}` |
| 删除任务 | DELETE | `https://api.dida365.com/open/v1/task/{taskId}` |
| 获取项目列表 | GET | `https://api.dida365.com/open/v1/project` |

### 官方文档

- 国内版：https://developer.dida365.com/api#/openapi
- 国际版：https://developer.ticktick.com/api#/openapi

---

## 更新记录

| 日期 | 版本 | 更新内容 |
|------|------|----------|
| 2026-02-20 | v1.0 | 初始版本，完成授权+任务创建流程 |

---

## 联系与支持

- **OpenClaw 文档：** https://docs.openclaw.ai
- **滴答清单开发者：** https://developer.dida365.com
- **本 SOP 维护：** Aria 🐔

---

*本文档基于 OpenClaw + 滴答清单 API 实际操作整理，确保可复现。*

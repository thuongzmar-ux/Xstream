# 🦞 OpenClaw Setup Playbook
> Hướng dẫn đầy đủ cài đặt OpenClaw + 9router bypass API + Telegram Bot  
> Verified: macOS · OpenClaw v2026.4.5 · 9router v0.3.82

---

## Prerequisites

| Yêu cầu | Version |
|---|---|
| Node.js | ≥ 22.12.0 |
| npm | ≥ 11.x |
| macOS | Monterey+ |

---

## PHASE 1 — Cài OpenClaw

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
openclaw --version   # → OpenClaw 2026.4.5
```

---

## PHASE 2 — Khởi động Gateway

```bash
openclaw gateway start
openclaw status
```
Gateway: `ws://127.0.0.1:18789` | Control UI: `http://127.0.0.1:18789`

---

## PHASE 3 — Kết nối Telegram Bot

### 3.1 Tạo bot
Telegram → `@BotFather` → `/newbot` → lấy **Bot Token**

### 3.2 Đấu nối bot

```bash
openclaw channels add --channel telegram --token <BOT_TOKEN>
openclaw channels status --probe
# → running, works ✅
```

### 3.3 Pair tài khoản Telegram

```bash
# 1. Nhắn /start vào bot → nhận Pairing Code
# 2. Approve:
openclaw pairing approve telegram <PAIRING_CODE>
```

---

## PHASE 4 — Cài 9router (API Bypass)

```bash
npm install -g 9router
9router --tray   # → http://localhost:20128
```

Dashboard: `http://localhost:20128/dashboard`

**Kết nối API trong dashboard:**
1. Providers → chọn provider (VD: OpenAI Codex)
2. Add account → nhập API key → Save
3. Note model IDs trong "Available Models" → VD: `cx/gpt-5.4`

> ⚠️ Model 9router dùng prefix riêng (`cx/gpt-5.4`), KHÔNG phải `gpt-4o` chuẩn.

---

## PHASE 5 — Cấu hình OpenClaw dùng 9router

### 5.1 Sửa `~/.openclaw/openclaw.json`

```json
{
  "gateway": {
    "mode": "local",
    "auth": { "mode": "token", "token": "<GATEWAY_TOKEN>" }
  },
  "channels": {
    "telegram": {
      "token": "<BOT_TOKEN>",
      "enabled": true,
      "accounts": { "default": { "token": "<BOT_TOKEN>" } },
      "botToken": "<BOT_TOKEN>"
    }
  },
  "agents": {
    "defaults": {
      "memorySearch": { "enabled": false },
      "model": { "primary": "9router/cx/gpt-5.4" }
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "9router": {
        "baseUrl": "http://localhost:20128/v1",
        "apiKey": "sk-9router",
        "api": "openai-completions",
        "models": [
          {"id": "cx/gpt-5.4", "name": "cx/gpt-5.4"},
          {"id": "cx/gpt-5.3-codex", "name": "cx/gpt-5.3-codex"},
          {"id": "cx/gpt-5.1-codex-mini", "name": "cx/gpt-5.1-codex-mini"}
        ]
      }
    }
  }
}
```

> `apiKey` có thể là bất kỳ string — 9router không check, tự lấy key từ dashboard.

### 5.2 Thêm env vào LaunchAgent plist

File: `~/Library/LaunchAgents/ai.openclaw.gateway.plist`  
Thêm vào block `<EnvironmentVariables>` trước `</dict>`:

```xml
<key>OPENAI_API_KEY</key>
<string>sk-9router-bypass</string>
<key>OPENAI_BASE_URL</key>
<string>http://localhost:20128/v1</string>
```

### 5.3 Reload daemon

```bash
launchctl bootout gui/$(id -u)/ai.openclaw.gateway
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist

# Verify
launchctl print gui/$(id -u)/ai.openclaw.gateway | grep OPENAI
# → OPENAI_API_KEY => sk-9router-bypass
# → OPENAI_BASE_URL => http://localhost:20128/v1
```

### 5.4 Verify

```bash
openclaw models status
# → Default: 9router/cx/gpt-5.4
# → 9router effective=models.json ✅
```

---

## PHASE 6 — Test

Telegram bot → `/new` → nhắn "hi" → bot trả lời ✅

---

## Troubleshooting

| Lỗi | Nguyên nhân | Fix |
|---|---|---|
| `401 Incorrect API key` | OpenClaw gọi thẳng OpenAI thật | Check `OPENAI_BASE_URL` trong plist |
| `404 No active credentials for provider: openai` | Sai model ID — 9router không có OpenAI key | Đổi sang `cx/gpt-5.4` trong config |
| `Unknown model: cx/gpt-5.4` | Chưa khai báo custom provider | Thêm `models.providers.9router` vào openclaw.json |
| `pairing required` | 2 gateway instances | `openclaw gateway stop` → `start` |
| `Something went wrong` | Session cũ lỗi | Gửi `/new` trong Telegram |
| Bot im lặng sau reboot | 9router chưa chạy | `9router --tray` |

---

## Maintenance

```bash
# Dọn session lỗi
openclaw sessions cleanup \
  --store ~/.openclaw/agents/main/sessions/sessions.json \
  --enforce --fix-missing

openclaw doctor        # Kiểm tra tổng thể
openclaw update        # Cập nhật OpenClaw
npm update -g 9router  # Cập nhật 9router
```

---

## Cấu trúc file quan trọng

```
~/.openclaw/
├── openclaw.json              ← Config chính
├── agents/main/agent/models.json  ← Auto-generated
├── workspace/                 ← AGENTS.md, SOUL.md, IDENTITY.md
└── logs/gateway.log

~/Library/LaunchAgents/
└── ai.openclaw.gateway.plist  ← Daemon (OPENAI env vars ở đây)
```

---

## Quick Start sau reboot

```bash
9router --tray          # ← BẮT BUỘC mỗi lần bật máy
openclaw status         # Gateway tự chạy qua LaunchAgent
# Bot Telegram online ngay!
```

---
*Playbook by Antigravity · OpenClaw v2026.4.5 · 9router v0.3.82 · macOS*

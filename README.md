# katabump 自动续期（GitHub Actions）

基于 GitHub Actions 的定时脚本，自动登录 [katabump](https://dashboard.katabump.com) 并续期免费服务器，避免因为忘记续期（每 4 天一次）导致服务器被停用/删除。

支持多账号（`USERS_JSON`）、Telegram 通知。

---

## 工作原理

- **Runner**：`ubuntu-latest`
- **脚本**：`app.py`（Python 3.12 + [seleniumbase](https://github.com/seleniumbase/SeleniumBase)）
- **过盾**：登录页有 Cloudflare Turnstile，脚本用 seleniumbase 的 **UC 模式**（`uc_gui_click_captcha`）在虚拟显示（xvfb）里**真实点击**验证框，而不是伪造点击事件
- **调度**：每天 UTC 00:00（北京时间 08:00）检查一次，未到续期窗口会自动跳过，不推送通知

> ⚠️ Turnstile 对出口 IP 敏感。GitHub 机房 IP 目前可以直连通过；如果哪天过不去了，配置 `HTTP_PROXY`（干净节点）即可。

---

## Secrets 配置

仓库 `Settings → Secrets and variables → Actions`：

| Secret 名称 | 是否必填 | 说明 |
|---|---|---|
| `USERS_JSON` | 必填 | 多账号 JSON 数组，格式见下 |
| `TG_BOT_TOKEN` | 可选 | Telegram Bot Token，用于推送续期结果 |
| `TG_CHAT_ID` | 可选 | 接收通知的 Chat ID |
| `HTTP_PROXY` | 可选 | 代理地址（如 `http://user:pass@host:port`），过盾失败时再加 |

### USERS_JSON 格式

```json
[
  {"username": "your_email@example.com", "password": "your_password"},
  {"username": "another@example.com", "password": "pwd"}
]
```

脚本会为每个账号独立开一个浏览器会话，依次登录并续期。

### Telegram 通知

- **续期成功** → ✅ 推送
- **登录失败** → ❌ 推送
- **未到续期时间** → 不推送（避免每天刷屏）

想要拿到 `TG_CHAT_ID`：先给 Bot 发一条 `/start`，然后访问
`https://api.telegram.org/bot<你的TOKEN>/getUpdates` 查 `chat.id`。

---

## 手动触发

`Actions → Katabump Auto Renew → Run workflow`。

---

## 文件说明

| 文件 | 说明 |
|---|---|
| `app.py` | 续期主脚本（登录、过 Turnstile、续期、TG 通知） |
| `.github/workflows/renew.yml` | 工作流：环境准备 → 运行脚本 → 上传截图 |

失败时的截图会作为 artifact `screenshots` 上传（如 `login_turnstile_fail.png`、`servers_page_fail.png`）。

---

## 重要提醒

1. **不要用 fork 跑这个**。
   GitHub 会主动禁用 fork 仓库里的定时工作流（提示 *Workflows on this fork have been disabled*），导致续期静默中断。本仓库已经通过 **Settings → Danger Zone → Leave fork network** 脱离 fork，变成独立仓库。
2. **60 天无活动会禁用定时任务**。
   独立仓库同样会因为「60 天无仓库活动」被自动禁用。工作流里的 `Keepalive` 步骤会在空闲满 30 天时自动补一个提交，规避这条规则。
3. **脚本退出码永远是 0**。
   单个账号失败不会让整个 run 变红，所以**不要只依赖邮件报警**，请留意 Telegram 通知或 run 日志。

---

## 鸣谢

过盾与续期方案参考自 [eooce/katabump-renew](https://github.com/eooce/katabump-renew) 与 [zv201413/katabump](https://github.com/zv201413/katabump)。

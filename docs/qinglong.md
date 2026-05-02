# 在青龙面板（Qinglong）中部署

本文档介绍如何在青龙面板中部署本项目，**跳过 Playwright/Chromium**，仅依赖 `httpx` 直连接口完成签到。

适用场景：
- ✅ 只签 `agentrouter`（默认就不需要 Playwright）
- ✅ 签 `anyrouter`，且当前你的青龙容器出口 IP 没有触发阿里云 WAF 挑战（多数情况下可行，建议先验证）

如果验证后发现 anyrouter 必须经过 WAF，请改回 GitHub Actions 部署，或在容器中额外安装 `playwright install chromium` 与系统依赖。

---

## 一、前置：验证 anyrouter 是否需要 WAF

**只签 agentrouter 可以跳过这步。** 如果你需要签 anyrouter，先在青龙容器里测一次：

```bash
docker exec -it qinglong sh
curl -i \
  -H "Cookie: session=你的session值" \
  -H "new-api-user: 你的api_user值" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  https://anyrouter.top/api/user/self
```

判断：

| 返回 | 含义 |
|---|---|
| `HTTP/1.1 200` + JSON `{"success": true, ...}` | ✅ 不需要 WAF，可继续本文档流程 |
| `HTTP/1.1 200` + HTML 页面 / 挑战页 | ❌ 需要 Playwright，本文档方案不适用 |
| `HTTP/1.1 401` | session 过期，重新拿 cookie 后再试 |

---

## 二、拉取仓库

在青龙面板 → **订阅管理** → 新建订阅：

- 名称：`anyrouter-check-in`
- 类型：`公开仓库`
- 链接：`https://github.com/millylee/anyrouter-check-in.git`（或你 fork 的地址）
- 定时类型：`crontab`
- 定时规则：`0 0 1 * *`（仓库本身只需偶尔拉取更新；签到调度由脚本自带的 cron 头决定）
- 白名单：`checkin\.py`
- 依赖文件：`requirements\.txt`

保存后点击运行一次，等待拉取完成。

或者用命令行方式（SSH 进青龙容器）：

```bash
ql repo https://github.com/millylee/anyrouter-check-in.git "checkin.py" "" "" "main"
```

---

## 三、安装 Python 依赖

青龙拉取仓库后，会读取 `requirements.txt` 自动安装。如果没有自动安装，可在容器内手动执行：

```bash
docker exec -it qinglong sh
pip install -r /ql/data/repo/<拉下来的目录名>/requirements.txt
```

依赖很轻：

- `httpx[http2]`
- `python-dotenv`

---

## 四、配置环境变量

在青龙面板 → **环境变量** → 新建，添加以下变量。

### 4.1 必需：账号配置

变量名：`ANYROUTER_ACCOUNTS`

值（JSON 单行）：

```json
[{"name":"我的agentrouter","provider":"agentrouter","cookies":{"session":"xxx"},"api_user":"yyy"}]
```

多账号示例：

```json
[
  {"name":"主账号","provider":"agentrouter","cookies":{"session":"sess1"},"api_user":"123"},
  {"name":"备用","provider":"anyrouter","cookies":{"session":"sess2"},"api_user":"456"}
]
```

### 4.2 关键：关闭 anyrouter 的 WAF 绕过（如果你要签 anyrouter）

变量名：`PROVIDERS`

值（JSON 单行）：

```json
{"anyrouter":{"domain":"https://anyrouter.top","bypass_method":null}}
```

这一步**至关重要**：默认 `anyrouter` 配置 `bypass_method="waf_cookies"`，会触发 Playwright；通过 `PROVIDERS` 覆盖为 `null` 后，将直接拿你 cookie 里的 session 调接口，不再启动浏览器。

> 仅签 agentrouter 的话不需要设这个变量，因为 agentrouter 默认就 `bypass_method=null`。

### 4.3 通知配置（直接走青龙）

本项目**已移除自带的通知模块**，统一使用青龙面板内置的 `notify.py`。这意味着：

- 你**只需要**在青龙面板（**系统设置 → 通知设置** 或 **环境变量**）中配置一次通知方式，就能给所有脚本使用
- 不需要在本项目里再设 `DINGDING_WEBHOOK`、`PUSHPLUS_TOKEN` 等环境变量
- 青龙支持的所有通知通道（钉钉、飞书、企业微信、PushPlus、Server 酱、Telegram、Bark、Gotify、邮件、iGot、Synology Chat 等）都可直接用

> 脚本运行时通过 `from notify import send` 引用青龙内置模块（青龙已自动把 `/ql/scripts/` 加到 `sys.path`）。如果在非青龙环境（本地、GitHub Actions）运行，会进入 ImportError 分支并跳过通知，不影响签到本身。

---

## 五、配置定时任务

脚本头部已写好青龙能识别的 cron：

```python
"""
cron: 0 */6 * * *
new Env('AnyRouter 签到');
"""
```

意思是每 6 小时跑一次。青龙面板 → **定时任务** 中找到 `AnyRouter 签到` 任务，确认 cron 表达式是 `0 */6 * * *`。如果没自动出现，可手动新建：

- 名称：`AnyRouter 签到`
- 命令：`task <你拉下来的目录名>/checkin.py`（或 `python3 /ql/data/repo/<目录名>/checkin.py`）
- 定时规则：`0 */6 * * *`

---

## 六、首次手动运行验证

定时任务列表里点击 `AnyRouter 签到` → 运行一次，查看日志。

**正常输出关键字**：

```
[SYSTEM] AnyRouter.top multi-account auto check-in script started
[INFO] Loaded 2 provider configuration(s)
[INFO] Found 1 account configurations
[PROCESSING] Starting to process xxx
[INFO] xxx: Bypass WAF not required, using user cookies directly  ← 关键，说明跳过了 Playwright
:money: Current balance: $xx.xx, Used: $xx.xx
[SUCCESS] xxx: Already checked in today  或  Check-in successful!
```

**如果出现 `playwright is not installed`**：说明你账号里有 provider 仍在用 `bypass_method=waf_cookies`。回到 4.2 设置 `PROVIDERS` 把它覆盖为 `null`。

**如果出现 401**：cookie 过期，重新去网站抓 session 更新 `ANYROUTER_ACCOUNTS`。

---

## 七、常见问题

### Q1：日志里看到 `[FAIL]` 但站点上余额确实增加了？

签到成功但接口返回的字段判断逻辑没匹配上。看下日志中 `Check-in failed - {error_msg}` 的内容，多数是"已经签到过"，脚本已识别这种情况算成功。

### Q2：每次都发通知，太吵了

脚本只在以下情况发通知：失败、首次运行、余额有变化。如果你余额每 6 小时就在变（比如有别的程序在用 quota），那是正常行为；可以拉长 cron 到 `0 0 * * *` 一天一次。

### Q3：想同时签多个 provider，但其中一个需要 WAF

本文档方案不适合你。建议：
1. 在 GitHub Actions 跑（仓库已有现成 workflow），或
2. 在青龙容器额外装 chromium：
   ```bash
   docker exec -it qinglong sh
   pip install playwright
   playwright install chromium
   playwright install-deps
   ```
   然后**不要**设置 `PROVIDERS` 来关 WAF。

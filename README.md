# Any Router 多账号自动签到（青龙面板版）

多平台多账号自动签到，理论上支持所有 NewAPI、OneAPI 平台，目前内置支持 **AnyRouter** 与 **AgentRouter**，其它可根据下方文档摸索配置。

> 本仓库 fork 自 [millylee/anyrouter-check-in](https://github.com/millylee/anyrouter-check-in)，**专为青龙面板部署改造**：移除 GitHub Actions 工作流，通知统一走青龙内置 `notify.py`，默认跳过 Playwright（直连 API）。

---

## 功能特性

- ✅ 多平台（兼容 NewAPI 与 OneAPI）
- ✅ 单/多账号自动签到
- ✅ 直接使用青龙面板的统一通知系统
- ✅ 默认无需 Chromium（通过 `PROVIDERS` 关掉 WAF 绕过即可）
- ✅ 余额变动时才发通知，避免噪音

---

## 快速开始

📘 完整部署指南见 **[docs/qinglong.md](./docs/qinglong.md)**，含：

- 验证 anyrouter 是否需要 WAF 的方法
- `ql repo` 拉库 / 订阅管理 UI 操作
- 环境变量配置（`ANYROUTER_ACCOUNTS`、`PROVIDERS`）
- 通知集成
- 故障排查

---

## 获取账号信息

每个签到账号需要两个值：

1. **Cookies**（用于身份验证）
2. **API User**（请求头 `new-api-user` 的值）

也可以借助 [在线 Secrets 配置生成器](https://millylee.github.io/anyrouter-check-in/) 一键生成。

### 获取 Cookies

1. 浏览器访问 https://anyrouter.top/ 并登录
2. F12 → Application → Cookies，复制 `session` 值（建议先重新登录确保最新，session 名义有效期 1 个月，但有时会提前失效，401 时重新获取）

![获取 cookies](./assets/request-session.png)

### 获取 API User

F12 → Network → 过滤 Fetch/XHR，找到带 `New-Api-User` 请求头的请求。该值正常是 5 位数；如果是负数或个位数，说明未登录。

![获取 api_user](./assets/request-api-user.png)

---

## 多账号配置格式

环境变量名：`ANYROUTER_ACCOUNTS`，JSON 数组，**单行**：

```json
[
  {
    "name": "AnyRouter 主账号",
    "provider": "anyrouter",
    "cookies": { "session": "abc123session" },
    "api_user": "user123"
  },
  {
    "name": "AgentRouter 备用",
    "provider": "agentrouter",
    "cookies": { "session": "xyz789session" },
    "api_user": "user456"
  }
]
```

**字段说明**

| 字段 | 必需 | 说明 |
|---|---|---|
| `cookies` | ✅ | 身份验证 cookies |
| `api_user` | ✅ | 请求头 `new-api-user` 的值 |
| `provider` | ❌ | 服务商名，默认 `anyrouter`，内置可选 `agentrouter` |
| `name` | ❌ | 自定义账号显示名（用于日志/通知） |

---

## 自定义 Provider（可选）

默认已内置 `anyrouter` 与 `agentrouter`。若需对接其它 NewAPI/OneAPI 站，或修改默认 provider 行为，设置环境变量 `PROVIDERS`：

```json
{
  "anyrouter": {
    "domain": "https://anyrouter.top",
    "bypass_method": null
  },
  "customrouter": {
    "domain": "https://custom.example.com"
  }
}
```

**字段说明**

| 字段 | 默认值 | 说明 |
|---|---|---|
| `domain` | （必需） | 服务商域名 |
| `login_path` | `/login` | 登录页路径（仅 `bypass_method=waf_cookies` 时使用） |
| `sign_in_path` | `/api/user/sign_in` | 签到接口；设 `null` 表示访问用户信息时自动完成签到 |
| `user_info_path` | `/api/user/self` | 用户信息接口 |
| `api_user_key` | `new-api-user` | 用户标识请求头名 |
| `bypass_method` | `null` | `"waf_cookies"` 启用 Playwright 绕过；`null` 直连 |
| `waf_cookie_names` | `null` | WAF cookie 名列表，`bypass_method=waf_cookies` 时必填 |

**关键提示**：内置 `anyrouter` 默认 `bypass_method="waf_cookies"`，需要 Playwright + Chromium。**青龙部署建议覆盖为 `null` 直连**：

```
PROVIDERS={"anyrouter":{"domain":"https://anyrouter.top","bypass_method":null}}
```

`agentrouter` 默认就是直连，无需配置。

---

## 通知

**已移除项目自带的通知模块**，统一通过青龙面板内置的 `notify.py` 推送。在青龙的 **系统设置 → 通知设置** 配置一次，所有脚本共享。

支持的通知渠道完全跟随青龙版本，常见包括：钉钉、飞书、企业微信、PushPlus、Server 酱、Telegram、Bark、Gotify、邮件、iGot、Synology Chat 等。

非青龙环境（本地）运行时，会自动跳过通知，签到本身不受影响。

---

## 执行频率

脚本头部 cron 默认 `0 */6 * * *`（每 6 小时一次）。可在青龙的定时任务中改为你期望的频率，比如 `0 0 * * *` 一天一次。

---

## 故障排除

| 现象 | 原因 / 解决 |
|---|---|
| HTTP 401 | session 过期，重新抓 cookie 更新 `ANYROUTER_ACCOUNTS` |
| `playwright is not installed` | 你某个 provider 仍设了 `bypass_method=waf_cookies` 但容器没装 Playwright。把它通过 `PROVIDERS` 改成 `null`，或装 `pip install playwright && playwright install chromium` |
| Error 1040 (08004) Too many connections | 站点数据库瞬时问题，等下一轮 cron 即可 |
| 没收到通知 | 检查青龙面板的「通知设置」是否配好；脚本日志里出现 `Qinglong notify module not available` 说明 `notify.py` 不在 `sys.path` 中（非青龙环境） |
| 每次都发通知 | 余额发生变化才会通知；如果你余额一直在变（其他程序消耗），属于正常 |

---

## 本地开发

```bash
# 安装依赖
uv sync

# 装 Playwright 浏览器（仅当你要测试 anyrouter 的 WAF 绕过时）
uv run playwright install chromium

# 运行
uv run checkin.py
```

`.env` 中可设：

```
ANYROUTER_ACCOUNTS=[{"cookies":{"session":"xxx"},"api_user":"yyy"}]
PROVIDERS={"anyrouter":{"domain":"https://anyrouter.top","bypass_method":null}}
PLAYWRIGHT_HEADLESS=1
```

---

## 免责声明

本脚本仅用于学习和研究目的，使用前请确保遵守相关网站的使用条款。

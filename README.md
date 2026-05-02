# Any Router 多账号自动签到（青龙面板版）

多平台多账号自动签到，理论上支持所有 NewAPI、OneAPI 平台，目前内置支持 **AnyRouter** 与 **AgentRouter**，其它可根据下方文档摸索配置。

> 本仓库 fork 自 [millylee/anyrouter-check-in](https://github.com/millylee/anyrouter-check-in)，**专为青龙面板部署改造**：移除 GitHub Actions 工作流，通知统一走青龙内置 `notify.py`，默认跳过 Playwright（直连 API）。

## 功能特性

- ✅ 多平台（兼容 NewAPI 与 OneAPI）
- ✅ 单/多账号自动签到
- ✅ 直接使用青龙面板的统一通知系统
- ✅ 默认无需 Chromium（通过 `PROVIDERS` 关掉 WAF 绕过即可）
- ✅ 余额变动时才发通知，避免噪音

---

## 第一步：获取账号信息

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

## 第二步：在青龙面板部署

### 2.1 验证 anyrouter 是否需要 WAF（仅签 anyrouter 时）

只签 `agentrouter` 可以**跳过这步**。如果你需要签 `anyrouter`，先在青龙容器里测一次：

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
| `HTTP/1.1 200` + JSON `{"success": true, ...}` | ✅ 不需要 WAF，可继续后面的部署 |
| `HTTP/1.1 200` + HTML 页面 / 挑战页 | ❌ 需要 Playwright，本仓库默认方案不适用，参考 [FAQ Q3](#q3想同时签多个-provider但其中一个需要-waf) |
| `HTTP/1.1 401` | session 过期，重新拿 cookie 后再试 |

### 2.2 拉取仓库

在青龙面板 → **订阅管理** → 新建订阅：

- 名称：`anyrouter-check-in-ql`
- 类型：`公开仓库`
- 链接：`https://github.com/yujiaxinlong/anyrouter-check-in-ql.git`（或你自己的 fork 地址）
- 定时类型：`crontab`
- 定时规则：`0 0 1 * *`（仓库本身只需偶尔拉取更新；签到调度由脚本自带的 cron 头决定）
- 白名单：`checkin\.py`
- 依赖文件：`requirements\.txt`

保存后点击运行一次，等待拉取完成。

或者用命令行方式（SSH 进青龙容器）：

```bash
ql repo https://github.com/yujiaxinlong/anyrouter-check-in-ql.git "checkin.py" "" "" "master"
```

### 2.3 安装 Python 依赖

青龙拉取仓库后会读取 `requirements.txt` 自动安装。如果没有自动安装，可在容器内手动执行：

```bash
docker exec -it qinglong sh
pip install -r /ql/data/repo/<拉下来的目录名>/requirements.txt
```

依赖很轻：

- `httpx[http2]`
- `python-dotenv`

### 2.4 配置环境变量

在青龙面板 → **环境变量** → 新建。

#### `ANYROUTER_ACCOUNTS`（必填）

JSON 数组，**单行**：

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

#### `PROVIDERS`（建议，签 anyrouter 时几乎必须）

默认 `anyrouter` 配置 `bypass_method="waf_cookies"`，会触发 Playwright；通过 `PROVIDERS` 覆盖为 `null` 后，将直接拿你 cookie 调接口，**不再启动浏览器**：

```json
{"anyrouter":{"domain":"https://anyrouter.top","bypass_method":null}}
```

仅签 `agentrouter` 的话不需要设这个变量，因为 `agentrouter` 默认就是直连。

#### 通知（直接走青龙，不在这里设）

本项目**已移除自带的通知模块**，改用青龙内置的 `notify.py`。在青龙的 **系统设置 → 通知设置** 配置一次即可，所有脚本共享。常见渠道：钉钉、飞书、企业微信、PushPlus、Server 酱、Telegram、Bark、Gotify、邮件、iGot、Synology Chat 等。

非青龙环境（本地）运行时，会自动跳过通知，签到本身不受影响。

### 2.5 配置定时任务

脚本头部已写好青龙能识别的 cron：

```python
"""
cron: 0 */6 * * *
new Env('AnyRouter 签到');
"""
```

每 6 小时跑一次。青龙拉取后会自动出现在 **定时任务** 中名为 `AnyRouter 签到` 的任务。如果没自动出现，可手动新建：

- 名称：`AnyRouter 签到`
- 命令：`task <你拉下来的目录名>/checkin.py`（或 `python3 /ql/data/repo/<目录名>/checkin.py`）
- 定时规则：`0 */6 * * *`

### 2.6 首次运行验证

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

---

## 配置详解

### `ANYROUTER_ACCOUNTS` 字段说明

| 字段 | 必需 | 说明 |
|---|---|---|
| `cookies` | ✅ | 身份验证 cookies，对象（`{"session":"xxx"}`）或字符串（`"key=val; key2=val2"`） |
| `api_user` | ✅ | 请求头 `new-api-user` 的值 |
| `provider` | ❌ | 服务商名，默认 `anyrouter`，内置可选 `agentrouter` |
| `name` | ❌ | 自定义账号显示名（用于日志/通知） |

### `PROVIDERS` 字段说明（自定义服务商）

默认已内置 `anyrouter` 与 `agentrouter`。若需对接其它 NewAPI/OneAPI 站，或修改默认 provider 行为，设置 `PROVIDERS` 环境变量：

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

| 字段 | 默认值 | 说明 |
|---|---|---|
| `domain` | （必需） | 服务商域名 |
| `login_path` | `/login` | 登录页路径（仅 `bypass_method=waf_cookies` 时使用） |
| `sign_in_path` | `/api/user/sign_in` | 签到接口；设 `null` 表示访问用户信息时自动完成签到 |
| `user_info_path` | `/api/user/self` | 用户信息接口 |
| `api_user_key` | `new-api-user` | 用户标识请求头名 |
| `bypass_method` | `null` | `"waf_cookies"` 启用 Playwright 绕过；`null` 直连 |
| `waf_cookie_names` | `null` | WAF cookie 名列表，`bypass_method=waf_cookies` 时必填 |

---

## 故障排除 / FAQ

| 现象 | 原因 / 解决 |
|---|---|
| HTTP 401 | session 过期，重新抓 cookie 更新 `ANYROUTER_ACCOUNTS` |
| `playwright is not installed` | 你某个 provider 仍设了 `bypass_method=waf_cookies` 但容器没装 Playwright。把它通过 `PROVIDERS` 改成 `null`，或参考 [Q3](#q3想同时签多个-provider但其中一个需要-waf) 装 Chromium |
| `Error 1040 (08004) Too many connections` | 站点数据库瞬时问题，等下一轮 cron 即可 |
| 没收到通知 | 检查青龙面板的「通知设置」是否配好；脚本日志里出现 `Qinglong notify module not available` 说明 `notify.py` 不在 `sys.path` 中（非青龙环境） |
| 每次都发通知 | 只在失败/首次运行/余额有变化时才发；如果余额一直在变（其他程序在用 quota），属正常，可拉长 cron 到一天一次 |

### Q1：日志里看到 `[FAIL]` 但站点上余额确实增加了？

签到成功但接口返回的字段判断逻辑没匹配上。看下日志中 `Check-in failed - {error_msg}` 的内容，多数是"已经签到过"，脚本已识别这种情况算成功。

### Q2：每次都发通知，太吵了

脚本只在以下情况发通知：失败、首次运行、余额有变化。如果你余额每 6 小时就在变（比如有别的程序在用 quota），那是正常行为；可以拉长 cron 到 `0 0 * * *` 一天一次。

### Q3：想同时签多个 provider，但其中一个需要 WAF

在青龙容器里额外装 Chromium：

```bash
docker exec -it qinglong sh
pip install playwright
playwright install chromium
playwright install-deps
```

然后**不要**通过 `PROVIDERS` 把对应 provider 的 `bypass_method` 设成 `null`，让它仍走 `waf_cookies` 流程。

注意：青龙容器一般是 Debian slim 镜像，`playwright install-deps` 拉取的系统库 + Chromium 加起来约 300MB。如果只是为了一个站点做 WAF 绕过，建议评估下 ROI。

---

## 本地开发（非青龙环境）

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

非青龙环境下通知会被自动跳过（`from notify import send` 找不到模块）；如需本地测试通知，可手动把青龙的 `notify.py` 复制一份放到 `PYTHONPATH` 中。

---

## 免责声明

本脚本仅用于学习和研究目的，使用前请确保遵守相关网站的使用条款。

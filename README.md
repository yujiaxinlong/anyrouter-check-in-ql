# Any Router 多账号自动签到（青龙面板版）

多平台多账号自动签到，理论上支持所有 NewAPI、OneAPI 平台，目前内置支持 **AnyRouter** 与 **AgentRouter**，其它可根据下方文档摸索配置。

> 本仓库 fork 自 [millylee/anyrouter-check-in](https://github.com/millylee/anyrouter-check-in)，**专为青龙面板部署改造**：移除 GitHub Actions 工作流，通知统一走青龙内置 `notify.py`，HTTP 客户端换成 `curl_cffi`（用浏览器 TLS 指纹）。

## 功能特性

- ✅ 多平台（兼容 NewAPI 与 OneAPI）
- ✅ 单/多账号自动签到
- ✅ 直接使用青龙面板的统一通知系统
- ✅ `curl_cffi` 模拟 Chrome TLS 指纹，避免被 WAF 在 TLS 层就 reset
- ✅ 余额变动时才发通知，避免噪音

## 部署前提

| Provider | 需要 Playwright？ | 说明 |
|---|---|---|
| `agentrouter` | ❌ 不需要 | 直连即可，最轻量 |
| `anyrouter` | ✅ 需要 | 阿里云 WAF 在 HTTP 层做 JS 挑战，必须真实浏览器跑完才能拿到 `acw_sc__v2` cookie |

如果你**只想签 agentrouter**，跳过下方 [2.2 节的 Playwright 安装]即可，部署只要 ~5MB 依赖。

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

### 2.1 拉取仓库

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

### 2.2 安装依赖

#### 基础依赖（无论签哪个 provider 都要装）

青龙拉取仓库后会读取 `requirements.txt` 自动装。如果没装上，进容器手动跑：

```bash
docker exec -it qinglong sh
pip install -r /ql/data/repo/<拉下来的目录名>/requirements.txt
```

依赖很轻（~5MB）：

- `curl_cffi`（浏览器 TLS 指纹模拟）
- `python-dotenv`

#### Playwright（仅签 anyrouter 必装）

anyrouter 在阿里云 WAF 后面，会用 Tengine 的 `denied by http_custom` 规则拦截一切非浏览器请求，必须真实浏览器跑完 JS 挑战拿到 `acw_tc + cdn_sec_tc + acw_sc__v2` 三个 cookie 才能放行：

```bash
docker exec -it qinglong sh
pip install playwright
playwright install chromium
playwright install-deps   # 安装 Chromium 运行时所需的系统库
```

代价：

- 镜像增大 ~300MB（Chromium + 系统库）
- 每次签到 anyrouter 多 5~10 秒（启动 headless 浏览器）

> 如果只签 agentrouter，**完全跳过这一步**。

### 2.3 配置环境变量

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

#### `PLAYWRIGHT_HEADLESS`（可选）

默认 `1`（无头）。本地调试想看见浏览器时设 `0`。青龙环境保持默认即可。

#### `PROVIDERS`（可选，高级）

仅在你需要对接其它 NewAPI/OneAPI 站点，或想覆盖默认配置时才用。`anyrouter` 和 `agentrouter` 已内置，正常使用**不需要**设。详见 [配置详解](#providers-字段说明自定义服务商)。

#### 通知（直接走青龙，不在这里设）

本项目**已移除自带的通知模块**，改用青龙内置的 `notify.py`。在青龙的 **系统设置 → 通知设置** 配置一次即可，所有脚本共享。常见渠道：钉钉、飞书、企业微信、PushPlus、Server 酱、Telegram、Bark、Gotify、邮件、iGot、Synology Chat 等。

非青龙环境（本地）运行时，会自动跳过通知，签到本身不受影响。

### 2.4 配置定时任务

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

### 2.5 首次运行验证

定时任务列表里点击 `AnyRouter 签到` → 运行一次，查看日志。

**只签 agentrouter 时正常输出**：

```
[SYSTEM] AnyRouter.top multi-account auto check-in script started
[INFO] Loaded 2 provider configuration(s)
[INFO] Found 1 account configurations
[PROCESSING] Starting to process xxx
[INFO] xxx: Bypass WAF not required, using user cookies directly
:money: Current balance: $xx.xx, Used: $xx.xx
[SUCCESS] xxx: Already checked in today  或  Check-in successful!
```

**签 anyrouter 时正常输出（多了 Playwright 那段）**：

```
[PROCESSING] xxx: Starting browser to get WAF cookies...
[PROCESSING] xxx: Access login page to get initial cookies...
[INFO] xxx: Got 3 WAF cookies
[SUCCESS] xxx: Successfully got all WAF cookies
:money: Current balance: $xx.xx, Used: $xx.xx
[SUCCESS] xxx: Check-in successful!
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

默认已内置 `anyrouter` 与 `agentrouter`。若需对接其它 NewAPI/OneAPI 站，设置 `PROVIDERS` 环境变量：

```json
{
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

> ⚠️ 之前版本的文档建议过 `PROVIDERS={"anyrouter":{"bypass_method":null}}` 来跳过 Playwright，**实测不可行**：anyrouter 的 WAF 不只看 TLS 指纹，还要 JS 挑战 cookie，纯 HTTP 客户端拿不到。如果你看到这个建议请忽略。

---

## 故障排除 / FAQ

| 现象 | 原因 / 解决 |
|---|---|
| HTTP 401 | session 过期，重新抓 cookie 更新 `ANYROUTER_ACCOUNTS` |
| `Failed to perform, curl: (35) TLS connect error` | anyrouter 的 WAF 拒了请求。装 Playwright（[2.2 节](#playwright仅签-anyrouter-必装)），不要把 `bypass_method` 改成 `null` |
| `playwright is not installed` | 你账号里有 anyrouter 但容器没装 Playwright。按 [2.2 节](#playwright仅签-anyrouter-必装) 装一下 |
| `Unknown impersonate target chrome131` | curl_cffi 版本太老，把代码里 `chrome131` 改成 `chrome124` 或 `chrome120` |
| `denied by http_custom` (curl 测试时) | WAF 在拒你，必须走 Playwright 拿完整 WAF cookies |
| `Error 1040 (08004) Too many connections` | 站点数据库瞬时问题，等下一轮 cron 即可 |
| 没收到通知 | 检查青龙面板的「通知设置」是否配好；脚本日志里出现 `Qinglong notify module not available` 说明 `notify.py` 不在 `sys.path` 中（非青龙环境） |
| 每次都发通知 | 只在失败/首次运行/余额有变化时才发；如果余额一直在变（其他程序在用 quota），属正常，可拉长 cron 到一天一次 |

### Q1：日志里看到 `[FAIL]` 但站点上余额确实增加了？

签到成功但接口返回的字段判断逻辑没匹配上。看下日志中 `Check-in failed - {error_msg}` 的内容，多数是"已经签到过"，脚本已识别这种情况算成功。

### Q2：每次都发通知，太吵了

脚本只在以下情况发通知：失败、首次运行、余额有变化。如果你余额每 6 小时就在变（比如有别的程序在用 quota），那是正常行为；可以拉长 cron 到 `0 0 * * *` 一天一次。

### Q3：能不能不装 Playwright？

只签 agentrouter 不用。如果一定要签 anyrouter 又不想装 Chromium，理论上只能：

1. 手动从浏览器拿到 `acw_tc + cdn_sec_tc + acw_sc__v2` 三个 cookie，放到 `ANYROUTER_ACCOUNTS` 的 `cookies` 字段里——但这些 WAF cookies 有效期只有 1~6 小时，需要频繁手动更新，不实用
2. 找个能跑 JS 的服务（自建 puppeteer service / browserless.io）替代本地 Playwright——超出本项目范畴

实际上每次签到多花 5~10 秒和 300MB 镜像空间换 anyrouter 的 $25/天额度，多数人会觉得划算。

---

## 本地开发（非青龙环境）

```bash
# 安装基础依赖
uv sync

# 装 Playwright 浏览器（仅当你要测试 anyrouter）
uv sync --extra playwright
uv run playwright install chromium

# 运行
uv run checkin.py
```

`.env` 中可设：

```
ANYROUTER_ACCOUNTS=[{"cookies":{"session":"xxx"},"api_user":"yyy"}]
PLAYWRIGHT_HEADLESS=1
```

非青龙环境下通知会被自动跳过（`from notify import send` 找不到模块）；如需本地测试通知，可手动把青龙的 `notify.py` 复制一份放到 `PYTHONPATH` 中。

---

## 免责声明

本脚本仅用于学习和研究目的，使用前请确保遵守相关网站的使用条款。

# Any Router 多账号自动签到（青龙面板版）

多平台多账号自动签到，理论上支持所有 NewAPI、OneAPI 平台，目前内置支持 **AnyRouter** 与 **AgentRouter**，其它平台可通过 `PROVIDERS` 环境变量自行配置。

> 本仓库 fork 自 [millylee/anyrouter-check-in](https://github.com/millylee/anyrouter-check-in)，**专为青龙面板部署改造**：移除 GitHub Actions 工作流，通知统一走青龙内置 `notify.py`，HTTP 客户端换成 `curl_cffi`（模拟浏览器 TLS 指纹）。

## 功能特性

- ✅ 多平台兼容（NewAPI / OneAPI）
- ✅ 单/多账号自动签到
- ✅ 直接使用青龙面板的统一通知系统
- ✅ `curl_cffi` 模拟 Chrome TLS 指纹，绕过 WAF 的 TLS 层检测
- ✅ 余额变动或签到失败时才发通知，避免消息轰炸
- ✅ `PROVIDERS` 覆盖内置配置时采用**合并模式**，只覆盖你显式指定的字段

## 部署前提

| Provider | 需要 Playwright？ | 说明 |
|---|---|---|
| `agentrouter` | ❌ 不需要 | `session` cookie 直连即可，最轻量 |
| `anyrouter` | ✅ 需要 | 阿里云 WAF 做 JS 挑战，必须真实浏览器跑完才能拿到 `acw_sc__v2` 等 cookie |

> 💡 如果你**只签 agentrouter**，跳过下方 Playwright 安装即可，部署只要 ~5MB 依赖。

---

## 第一步：获取账号信息

每个签到账号需要两个值：

1. **Cookies**（身份验证）
2. **API User**（请求头 `new-api-user` 的值）

也可借助 [在线 Secrets 配置生成器](https://millylee.github.io/anyrouter-check-in/) 一键生成。

### 获取 Cookies

1. 浏览器访问对应站点（如 https://anyrouter.top/ 或 https://agentrouter.org/）并登录
2. F12 → Application → Cookies，复制 `session` 值

> ⚠️ **不要登出！** 登出会让服务端销毁 session，你配置的 cookie 就会失效（表现为 401）。需要更新 cookie 时重新登录即可，不用的页面直接关闭。session 名义有效期约 1 个月，但可能提前失效。

![获取 cookies](./assets/request-session.png)

### 获取 API User

F12 → Network → 过滤 Fetch/XHR，找到带 `New-Api-User` 请求头的请求。该值正常是 5 位数；如果是负数或个位数，说明未登录。

![获取 api_user](./assets/request-api-user.png)

---

## 第二步：在青龙面板部署

### 2.1 选择青龙面板镜像

> ⚠️ **重要：如需签到 anyrouter（需要 Playwright），必须使用 Debian 版青龙面板。** 默认的 Alpine 版不支持安装 Chromium 运行所需的系统库。

```bash
# Debian 版（支持 Playwright）— 签 anyrouter 必须用这个
docker pull whyour/qinglong:debian

# 默认 Alpine 版（轻量，仅签 agentrouter 可用）
docker pull whyour/qinglong:latest
```

如果你已经在用 Alpine 版并且需要签 anyrouter，需要迁移到 Debian 版：

```bash
# 1. 备份数据
docker cp qinglong:/ql/data ./ql-backup

# 2. 停止并删除旧容器
docker stop qinglong && docker rm qinglong

# 3. 用 Debian 版重新创建容器（挂载原数据卷或恢复备份）
docker run -d --name qinglong \
  -p 5700:5700 \
  -v $(pwd)/ql-backup:/ql/data \
  whyour/qinglong:debian
```

### 2.2 拉取仓库

在青龙面板 → **订阅管理** → 新建订阅：

- 名称：`anyrouter-check-in-ql`
- 类型：`公开仓库`
- 链接：`https://github.com/yujiaxinlong/anyrouter-check-in-ql.git`（或你自己的 fork 地址）
- 定时类型：`crontab`
- 定时规则：`0 0 1 * *`（仓库只需偶尔拉取更新）
- 白名单：`checkin\.py`
- 依赖文件：`requirements\.txt`

保存后运行一次，等待拉取完成。

或者 SSH 进容器用命令行：

```bash
ql repo https://github.com/yujiaxinlong/anyrouter-check-in-ql.git "checkin.py" "" "" "master"
```

### 2.3 安装依赖

#### 基础依赖（所有 provider 都要装）

青龙拉取仓库后会读取 `requirements.txt` 自动安装。如果没装上，进容器手动跑：

```bash
docker exec -it qinglong bash
pip install curl_cffi python-dotenv
```

依赖很轻（~5MB）：

| 包 | 用途 |
|---|---|
| `curl_cffi` | 浏览器 TLS 指纹模拟，绕过 WAF 的 TLS 层检测 |
| `python-dotenv` | 读取 `.env` 文件（本地开发用） |

#### Playwright + Chromium（仅签 anyrouter 必装）

anyrouter 在阿里云 WAF 后面，会用 JS 挑战拦截非浏览器请求，必须真实浏览器跑完才能拿到 `acw_tc` + `cdn_sec_tc` + `acw_sc__v2` 三个 cookie：

```bash
# 确保你用的是 Debian 版青龙面板！
docker exec -it qinglong bash

pip install playwright
playwright install chromium
playwright install-deps   # 安装 Chromium 运行时所需的系统库（apt 包）
```

> ⚠️ `playwright install-deps` 会通过 `apt` 安装大量系统库（libglib、libnss3、libatk 等），**只有 Debian/Ubuntu 系统才支持**。Alpine 版会直接报错。这就是为什么需要 `whyour/qinglong:debian` 镜像。

代价：

- 镜像增大 ~300MB（Chromium + 系统库）
- 每次签到 anyrouter 多 5~10 秒（启动 headless 浏览器）

> 如果只签 agentrouter，**完全跳过这一步**。

### 2.4 配置环境变量

在青龙面板 → **环境变量** → 新建。

#### `ANYROUTER_ACCOUNTS`（必填）

JSON 数组，支持多账号、混合 provider：

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

| 字段 | 必填 | 说明 |
|---|---|---|
| `cookies` | ✅ | 对象 `{"session":"xxx"}` 或字符串 `"key=val; key2=val2"` |
| `api_user` | ✅ | 请求头 `new-api-user` 的值 |
| `provider` | ❌ | 默认 `anyrouter`，可选 `agentrouter` 或自定义 |
| `name` | ❌ | 账号显示名（用于日志和通知） |

#### `PLAYWRIGHT_HEADLESS`（可选）

默认 `1`（无头）。本地调试想看浏览器窗口时设 `0`。青龙环境保持默认。

#### `PROVIDERS`（可选，高级用户）

`anyrouter` 和 `agentrouter` 已内置默认配置，正常使用**不需要设**。

覆盖内置 provider 时采用**合并模式**：只覆盖你显式指定的字段，未指定的字段保留内置默认值。例如只想关闭 anyrouter 的 WAF bypass：

```json
{
  "anyrouter": { "bypass_method": null }
}
```

这只会把 `bypass_method` 改为 `null`，其他字段（`domain`、`sign_in_path`、`waf_cookie_names` 等）都保持内置值不变。

新增自定义 provider：

```json
{
  "customrouter": {
    "domain": "https://custom.example.com"
  }
}
```

完整字段列表：

| 字段 | 默认值 | 说明 |
|---|---|---|
| `domain` | （必填） | 服务商域名 |
| `login_path` | `/login` | 登录页路径（仅 `bypass_method=waf_cookies` 时用） |
| `sign_in_path` | `/api/user/sign_in` | 签到接口；设 `null` 表示查询用户信息时自动签到 |
| `user_info_path` | `/api/user/self` | 用户信息接口 |
| `api_user_key` | `new-api-user` | 用户标识请求头名 |
| `bypass_method` | `null` | `"waf_cookies"` 启用 Playwright 绕过；`null` 直连 |
| `waf_cookie_names` | `null` | WAF cookie 名列表，仅 `bypass_method=waf_cookies` 时需要 |

#### 通知配置

本项目**已移除自带的通知模块**，改用青龙内置的 `notify.py`。在青龙的 **系统设置 → 通知设置** 中统一配置即可，所有脚本共享。

非青龙环境运行时会自动跳过通知，签到本身不受影响。

### 2.5 配置定时任务

脚本头部已写好青龙可识别的 cron：

```python
# cron: 0 */6 * * *
# new Env('AnyRouter 签到');
```

每 6 小时跑一次。青龙拉取后会自动出现在 **定时任务** 中。如果没自动出现，手动新建：

- 名称：`AnyRouter 签到`
- 命令：`task <目录名>/checkin.py`
- 定时规则：`0 */6 * * *`

### 2.6 首次运行验证

在定时任务列表里点击 `AnyRouter 签到` → 运行一次，查看日志。

**只签 agentrouter 时的正常输出**：

```
[SYSTEM] AnyRouter.top multi-account auto check-in script started
[INFO] Found 1 account configurations
[PROCESSING] Starting to process AgentRouter 备用
[INFO] AgentRouter 备用: Bypass WAF not required, using user cookies directly
:money: Current balance: $325.0, Used: $0.0
[INFO] AgentRouter 备用: Check-in completed automatically
```

**签 anyrouter 时的正常输出**（多了 Playwright 部分）：

```
[PROCESSING] AnyRouter 主账号: Starting browser to get WAF cookies...
[PROCESSING] AnyRouter 主账号: Access login page to get initial cookies...
[INFO] AnyRouter 主账号: Got 3 WAF cookies
[SUCCESS] AnyRouter 主账号: Successfully got all WAF cookies
:money: Current balance: $xx.xx, Used: $xx.xx
[SUCCESS] AnyRouter 主账号: Check-in successful!
```

---

## 故障排除

| 现象 | 原因 / 解决 |
|---|---|
| HTTP 401 | session 过期。重新登录（**不要登出旧 session**）抓新 cookie，更新 `ANYROUTER_ACCOUNTS` |
| HTTP 404 (`Invalid URL POST /api/user/sign_in`) | agentrouter 没有 `/api/user/sign_in` 接口。确保 `PROVIDERS` 覆盖配置中没有错误地设置 `sign_in_path` |
| `curl: (35) TLS connect error` | WAF 拒绝了请求。安装 Playwright（[2.3 节](#playwright--chromium仅签-anyrouter-必装)） |
| `playwright is not installed` | 账号中有 anyrouter 但容器没装 Playwright。按 [2.3 节](#playwright--chromium仅签-anyrouter-必装) 安装 |
| `playwright install-deps` 报错 | 你用的是 Alpine 版青龙面板，需要换成 Debian 版：`docker pull whyour/qinglong:debian` |
| `Unknown impersonate target chrome131` | curl_cffi 版本过低，`pip install --upgrade curl_cffi` |
| `denied by http_custom` | WAF 正在拒绝你，必须走 Playwright 拿完整 WAF cookies |
| 没收到通知 | 检查青龙「通知设置」；日志中 `Qinglong notify module not available` 说明不在青龙环境 |
| 每次都发通知 | 只在失败/首次运行/余额变化时才发。如果余额持续变化（有程序在用 quota），可改 cron 为 `0 0 * * *` 一天一次 |

### Q：能不能不装 Playwright 签 anyrouter？

不实用。替代方案：

1. 手动从浏览器抓 `acw_tc` + `cdn_sec_tc` + `acw_sc__v2` 三个 cookie 放到 `cookies` 字段 — 但有效期只有 1~6 小时，需要频繁手动更新
2. 自建 puppeteer/browserless 服务 — 超出本项目范畴

每次签到多花 5~10 秒和 300MB 空间换 anyrouter 的签到额度，通常是值得的。

---

## 本地开发

```bash
# 安装基础依赖
uv sync

# 装 Playwright（仅测试 anyrouter 时需要）
uv sync --extra playwright
uv run playwright install chromium

# 配置环境变量
cp .env.example .env
# 编辑 .env 填入你的账号信息

# 运行
uv run checkin.py
```

非青龙环境下通知会自动跳过。如需本地测试通知，可手动把青龙的 `notify.py` 复制到 `PYTHONPATH` 中。

---

## 免责声明

本脚本仅用于学习和研究目的，使用前请确保遵守相关网站的使用条款。

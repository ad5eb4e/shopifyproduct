# Shopify 产品数据同步系统 — 技术文档

## 1. 系统架构

```
┌─────────────────────────────────────────────────┐
│              Mac mini（本地部署）                  │
│                                                  │
│  ┌──────────────┐     ┌───────────────────────┐ │
│  │  Web 管理后台 │     │  定时调度器             │ │
│  │  (gunicorn)   │     │  (APScheduler)         │ │
│  │  - 店铺配置   │     │  - 按店铺设定频率触发    │ │
│  │  - 状态查看   │     │  - 通过代理IP访问       │ │
│  └──────┬───────┘     └──────────┬────────────┘ │
│         │                        │               │
│         ▼                        ▼               │
│  ┌──────────────┐     ┌───────────────────────┐ │
│  │  SQLite(WAL) │     │  同步引擎              │ │
│  │  (店铺配置)   │     │  - Shopify Admin API   │ │
│  │              │     │  - updated_at 变更检测  │ │
│  └──────────────┘     │  - lark-cli 写多维表    │ │
│                       │  - 回写 Shopify         │ │
│                       └───────────┬─────────────┘ │
│         Nginx (TLS)               │               │
│         ↕ :443                    │               │
│  ┌──────────────┐        ┌────────┼────────┐      │
│  │ gunicorn:8080│        ▼        ▼        ▼      │
│  │  (内网监听)   │     代理IP-A  代理IP-B  代理IP-C │
│  └──────────────┘        │        │        │      │
└──────────────────────────┼────────┼────────┼──────┘
                           ▼        ▼        ▼
                      Shopify-A Shopify-B Shopify-C
```

## 2. 技术栈

| 组件 | 技术 | 说明 |
|------|------|------|
| Web 后台 | Flask + SQLite | 轻量，单文件部署 |
| WSGI 服务器 | gunicorn (gthread) | `gunicorn -w 1 --threads 4`，**禁止使用 Flask 开发服务器** |
| 反向代理 | Nginx + Let's Encrypt | TLS 终止，**必须部署** |
| 前端页面 | 原生 HTML + JS | 不用框架，内联表格编辑 |
| 定时调度 | APScheduler | 按店铺配置频率运行 |
| HTTP 请求 | requests | 支持代理设置 |
| 数据存储 | SQLite (WAL 模式) | 店铺配置，token 加密存储 |
| 飞书写入 | lark-cli (subprocess) | 已安装的命令行工具 |
| Token 加密 | cryptography (Fernet) | 对称加密，密钥从环境变量读取 |
| 密码哈希 | bcrypt | 管理员密码哈希存储和验证 |
| HTML 消毒 | bleach | 回写前清理 HTML，防止 XSS |
| HTML 转文本 | html2text | 去标签保留结构 |
| 代理支持 | PySocks + requests | SOCKS5 / HTTP 代理 |
| 限速 | flask-limiter | 防暴力破解 |
| 安全头 | flask-talisman | CSP、X-Frame-Options 等 |
| CSRF 保护 | flask-wtf (CSRFProtect) | 所有非 GET 端点强制校验 CSRF token |
| 进程管理 | launchd (macOS) | 开机自启、崩溃重启、日志管理 |

## 3. 项目结构

```
shopifyproduct/
  PRD.md                       # 需求文档（运营看）
  TECHNICAL.md                 # 技术文档（本文件）
  COMPLIANCE_PROMPT.md         # 合规改写规则（不提交 Git，.gitignore 排除）
  requirements.txt             # Python 依赖
  .gitignore

  app/
    __init__.py
    web.py                     # Flask 路由和后台页面
    db.py                      # SQLite 数据库操作（所有 SQL 必须参数化查询）
    crypto.py                  # Token 加密解密
    validators.py              # 输入校验（store_name、shop、proxy 等）
    templates/
      index.html               # 管理后台页面（内联表格编辑）

  sync/
    __init__.py
    shopify_auth.py            # Shopify 认证（Token/OAuth 双模式）
    shopify_admin.py           # Shopify Admin API 封装
    shopify_sync.py            # 同步逻辑（变更检测、数据对比）
    lark_writer.py             # lark-cli 命令封装
    models.py                  # 数据结构定义

  scripts/
    run_web.py                 # 启动 Web 后台
    scheduler.py               # 启动定时调度器（内嵌 Flask 进程，单进程架构）
    writeback.py               # 回写 Shopify
    start.sh                   # launchd 启动入口（加载 .env + 启动 gunicorn）

  nginx/
    shopifyproduct.conf        # Nginx 配置模板

  deploy/
    com.shopifyproduct.plist   # macOS launchd 服务文件
    .env.example               # 环境变量模板

  logs/                        # 运行日志
  data/                        # SQLite 数据库文件（chmod 600）
```

## 4. 依赖列表

```
flask
flask-limiter
flask-talisman
flask-wtf
flask-login
gunicorn
requests
html2text
bleach
cryptography
bcrypt
apscheduler
pysocks
```

## 5. 数据库设计（SQLite，WAL 模式）

初始化时执行 `PRAGMA journal_mode=WAL;` 以支持并发读写。

### stores 表

```sql
CREATE TABLE stores (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    store_name     TEXT NOT NULL UNIQUE,          -- 店铺名称，只允许 [a-zA-Z0-9_-]
    shop           TEXT NOT NULL,                 -- myshopify.com 前缀，只允许 [a-z0-9-]
    storefront     TEXT NOT NULL,                 -- 前端域名
    auth_mode      TEXT NOT NULL DEFAULT 'token', -- 认证方式: 'token' 或 'oauth'
    api_token      TEXT,                          -- 加密后的 Admin API Token
    client_id      TEXT,                          -- OAuth Client ID（不加密，非敏感）
    client_secret  TEXT,                          -- 加密后的 Client Secret
    oauth_token    TEXT,                          -- 加密后的当前 access token（自动获取）
    oauth_expires  TEXT,                          -- token 过期时间 ISO8601
    proxy          TEXT NOT NULL,                 -- 代理地址，只允许 socks5://或http(s)://开头
    light_cron     TEXT NOT NULL DEFAULT '0 3 * * *',   -- 轻量扫描 cron
    deep_cycle     INTEGER NOT NULL DEFAULT 3,          -- 深度扫描周期（天），如 3 = 3天扫完所有产品
    deep_hour      INTEGER NOT NULL DEFAULT 4,          -- 深度扫描每天执行时间（小时，0-23）
    deep_offset    INTEGER NOT NULL DEFAULT 0,          -- 当前深度扫描偏移量（内部使用，自动更新）
    table_id       TEXT NOT NULL,
    enabled        INTEGER NOT NULL DEFAULT 1,
    created_at     TEXT DEFAULT CURRENT_TIMESTAMP,
    updated_at     TEXT DEFAULT CURRENT_TIMESTAMP
);
```

> **注意：** 相比上一版，移除了 `oauth_refresh` 字段。Shopify Client Credentials 流程不返回 refresh token，详见 7.1 节。

### settings 表

```sql
CREATE TABLE settings (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);
-- 预设: lark_base_token（Fernet 加密存储，与 Shopify token 同等对待）
-- 注意: encrypt_key 不存数据库，从环境变量 FERNET_KEY 读取
```

> **安全要求：** `lark_base_token` 授予飞书多维表的完整读写权限，必须使用 Fernet 加密存储，不得明文写入数据库。

### sync_logs 表

```sql
CREATE TABLE sync_logs (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    store_id   INTEGER NOT NULL,
    action     TEXT NOT NULL,     -- sync / writeback
    status     TEXT NOT NULL,     -- success / failed
    summary    TEXT,              -- 新增N条，更新N条，失败N条
    detail     TEXT,              -- 完整日志（已脱敏，不含 token）
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (store_id) REFERENCES stores(id)
);
```

### SQL 安全规范

**所有查询必须使用参数化：**

```python
# 正确
cursor.execute("SELECT * FROM stores WHERE store_name = ?", (name,))

# 错误 — 禁止
cursor.execute(f"SELECT * FROM stores WHERE store_name = '{name}'")
```

## 6. Web 管理后台

### 6.1 页面

单页面，内联表格编辑。所有店铺配置在一张表格中直接编辑保存。

**字段：**

| 字段 | 必填 | 说明 |
|------|------|------|
| 店铺名称 | 是 | 自定义标识，只允许字母、数字、连字符、下划线 |
| 店铺标识 | 是 | myshopify.com 前缀，只允许小写字母、数字、连字符 |
| 前端域名 | 是 | 客户访问的自定义域名 |
| 认证方式 | 是 | 下拉选择：Token / OAuth |
| Access Token | Token时必填 | 永久 token（`shpat_xxx`），加密存储，页面星号遮蔽 |
| Client ID | OAuth时必填 | Developer Dashboard 应用 Client ID |
| Client Secret | OAuth时必填 | 加密存储，页面星号遮蔽 |
| 代理IP | 是 | 格式 `socks5://ip:port` 或 `http://ip:port` |
| 轻量扫描频率 | 是 | 检测新产品+基础字段变化。可选：每小时/每6小时/每天+时间 |
| 深度扫描周期 | 是 | 几天内循环扫完所有产品。可选：1/2/3/5/7 天 |
| 深度扫描时间 | 是 | 每天执行深度批次的小时，0-23 |
| 飞书 Table ID | 是 | 该店铺对应的多维表 table ID |

前端根据认证方式切换显示：选 Token 时隐藏 Client ID/Secret 输入框，选 OAuth 时隐藏 Access Token 输入框。

**操作：**
- 保存 / 删除 / 测试连接 / 立即同步 / 查看日志

### 6.2 输入校验（validators.py）

所有用户输入在保存数据库前必须校验：

```python
import re
import socket
import ipaddress

STORE_NAME_RE = re.compile(r'^[a-zA-Z0-9_-]+$')
SHOP_RE = re.compile(r'^[a-z0-9-]+$')
PROXY_RE = re.compile(r'^(socks5|https?)://[a-zA-Z0-9._-]+:\d+$')
STOREFRONT_RE = re.compile(r'^[a-zA-Z0-9]([a-zA-Z0-9.-]*[a-zA-Z0-9])?$')
HANDLE_RE = re.compile(r'^[a-z0-9-]+$')
PRODUCT_ID_RE = re.compile(r'^\d+$')

def _is_private_ip(host: str) -> bool:
    """检查 IP 或主机名是否解析到私有/保留地址（防 SSRF）"""
    try:
        # 尝试直接解析为 IP
        addr = ipaddress.ip_address(host)
        return addr.is_private or addr.is_loopback or addr.is_reserved or addr.is_link_local
    except ValueError:
        pass
    # 主机名：DNS 解析后检查
    try:
        resolved = socket.getaddrinfo(host, None)
        for _, _, _, _, sockaddr in resolved:
            addr = ipaddress.ip_address(sockaddr[0])
            if addr.is_private or addr.is_loopback or addr.is_reserved or addr.is_link_local:
                return True
    except socket.gaierror:
        return True  # DNS 解析失败，拒绝
    return False

def validate_store_name(name: str) -> bool:
    return bool(STORE_NAME_RE.match(name)) and len(name) <= 50

def validate_shop(shop: str) -> bool:
    return bool(SHOP_RE.match(shop)) and len(shop) <= 100

def validate_proxy(proxy: str) -> bool:
    if not PROXY_RE.match(proxy):
        return False
    host = proxy.split('://')[1].rsplit(':', 1)[0]
    return not _is_private_ip(host)

def validate_storefront(domain: str) -> bool:
    """校验前端域名：合法主机名格式 + 不解析到私有 IP"""
    if not STOREFRONT_RE.match(domain) or len(domain) > 253:
        return False
    # 不允许纯 IP 地址作为 storefront
    try:
        ipaddress.ip_address(domain)
        return False  # 拒绝 IP 地址
    except ValueError:
        pass
    return not _is_private_ip(domain)

def validate_product_id(pid: str) -> bool:
    """校验 Shopify 产品 ID：纯数字字符串"""
    return bool(PRODUCT_ID_RE.match(pid)) and len(pid) <= 20

def validate_handle(handle: str) -> bool:
    """校验 Handle：小写字母、数字、连字符"""
    return bool(HANDLE_RE.match(handle)) and len(handle) <= 255

def validate_hour(hour) -> bool:
    """校验扫描频率小时：整数 0-23"""
    return isinstance(hour, int) and 0 <= hour <= 23
```

> **SSRF 防御要点：**
> - 使用 `ipaddress` 标准库，覆盖 IPv4/IPv6 所有私有、回环、保留、链路本地地址
> - 主机名先 DNS 解析再检查 IP，防止 DNS 重绑定
> - `storefront` 字段必须是合法域名，**禁止 IP 地址**
> - 从飞书读取的 `product_id` 必须校验为纯数字后才可拼入 URL

### 6.3 认证与安全

**Session 登录 + bcrypt + CSRF + TLS（强制）：**

使用 Flask-Login 实现 session 登录，替代 Basic Auth：

```python
import bcrypt
from flask_login import LoginManager, login_required
from flask_wtf.csrf import CSRFProtect

csrf = CSRFProtect(app)
login_manager = LoginManager(app)

# 管理员密码以 bcrypt 哈希存储在环境变量中
# 首次部署生成哈希：python -c "import bcrypt; print(bcrypt.hashpw(b'your_password', bcrypt.gensalt()).decode())"
ADMIN_USER = os.environ["ADMIN_USER"]
ADMIN_PASS_HASH = os.environ["ADMIN_PASS_HASH"]  # bcrypt 哈希，非明文

# 启动时检测：哈希不能是 "changeme" 的哈希
if bcrypt.checkpw(b"changeme", ADMIN_PASS_HASH.encode()):
    sys.exit("ERROR: 请修改默认密码")

# 登录验证
def verify_login(username, password):
    return (username == ADMIN_USER and
            bcrypt.checkpw(password.encode(), ADMIN_PASS_HASH.encode()))
```

- Session 超时 30 分钟无操作自动退出
- 所有非 GET 端点强制 CSRF token 校验（前端通过 `<meta name="csrf-token">` 获取）
- gunicorn 监听 `0.0.0.0:8080`，局域网内通过 `http://mac-mini-ip:8080` 访问
- 如需公网访问，**必须**在前面部署 Nginx/Caddy + TLS

**登录限速：**
```python
from flask_limiter import Limiter
limiter = Limiter(app, default_limits=["60 per minute"])

# 登录端点严格限速：防暴力破解
@app.route("/login", methods=["POST"])
@limiter.limit("5 per 15 minutes")
def login(): ...

# token 明文端点严格限速
@app.route("/api/stores/<id>/token")
@limiter.limit("5 per minute")
@login_required
def get_token(id): ...
```

**安全响应头：**
```python
from flask_talisman import Talisman
Talisman(app, content_security_policy={"default-src": "'self'"})
```

**审计日志：** 每次调用 `/api/stores/<id>/token` 记录到 sync_logs（action="token_view"）。登录失败也记录（action="login_failed"）。

### 6.4 扫描频率前端转 cron

前端提供下拉选择，后端转换为 cron 表达式分别存入 `light_cron` 和 `deep_cron` 字段。

**轻量扫描选项（light_cron）：**

| 前端选项 | cron 表达式 |
|---------|------------|
| 每小时 | `0 * * * *` |
| 每6小时 | `0 */6 * * *` |
| 每天 + 小时(0-23) | `0 {hour} * * *` |

**深度扫描：** 不使用 cron 表达式，而是存 `deep_cycle`（周期天数）和 `deep_hour`（每天执行小时）。调度器每天在 `deep_hour` 时触发一次 `deep_sync()`，函数内部根据 `deep_cycle` 计算当天要扫的产品范围。

前端传 `{"deep_cycle": 3, "deep_hour": 4}`，后端直接存入数据库。

**后端必须校验：`deep_cycle` 为 1-7 的整数，`deep_hour` 为 0-23 的整数。**

### 6.5 API 路由

```
GET    /                    → 管理后台页面（需登录）
GET    /login               → 登录页面
POST   /login               → 登录验证（限速 5次/15分钟）
GET    /healthz             → 健康检查（无需登录）
GET    /api/stores          → 获取所有店铺配置（敏感字段星号化）
POST   /api/stores          → 新增店铺
PUT    /api/stores/<id>     → 更新店铺配置
DELETE /api/stores/<id>     → 删除店铺
POST   /api/stores/<id>/test    → 测试 Shopify 连接
POST   /api/stores/<id>/sync    → 手动触发同步
GET    /api/stores/<id>/logs    → 获取同步日志
GET    /api/stores/<id>/token   → 获取明文 token（限速 5次/分钟，有审计日志）
GET    /api/settings        → 获取全局设置
PUT    /api/settings        → 更新全局设置
```

### 6.6 凭证安全

- 使用 `cryptography.fernet.Fernet` 对称加密
- **加密密钥从环境变量 `FERNET_KEY` 读取，不存数据库**
- 首次部署时生成密钥：`python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`
- 以下字段加密存储：`api_token`, `client_secret`, `oauth_token`, `lark_base_token`（settings 表）
- API 返回店铺列表时，所有敏感字段替换为星号
- `client_id` 不加密（非敏感）
- 日志记录前对敏感字段脱敏（过滤 `X-Shopify-Access-Token`、`client_secret` 等）

## 7. Shopify API 对接

### 7.1 认证

系统支持两种认证方式，通过 `store.auth_mode` 字段区分：

#### 方式一：Legacy Token（`auth_mode = "token"`）

直接使用永久 Access Token，适用于 2026.1.1 前创建的 Custom App：

```python
headers = {
    "X-Shopify-Access-Token": decrypt(store.api_token),
    "Content-Type": "application/json"
}
```

#### 方式二：OAuth Client Credentials（`auth_mode = "oauth"`）

使用 Client ID + Client Secret 获取短期 token。**Shopify Client Credentials 流程不返回 refresh token**，每次 token 过期后直接用 client_id + client_secret 重新获取。

```python
def get_oauth_token(store):
    """获取 OAuth access token，过期则重新申请"""
    # 1. 检查现有 token 是否还有效（提前 5 分钟刷新）
    if store.oauth_token and store.oauth_expires:
        expires = datetime.fromisoformat(store.oauth_expires)
        if datetime.utcnow() < expires - timedelta(minutes=5):
            return decrypt(store.oauth_token)

    # 2. 用 client credentials 获取新 token
    resp = requests.post(
        f"https://{store.shop}.myshopify.com/admin/oauth/access_token",
        json={
            "client_id": store.client_id,
            "client_secret": decrypt(store.client_secret),
            "grant_type": "client_credentials"
        },
        proxies=get_proxies(store),
        timeout=10
    )
    resp.raise_for_status()  # 401/403 等异常向上抛
    data = resp.json()

    new_token = data["access_token"]
    update_store_oauth(store.id,
        oauth_token=encrypt(new_token),
        oauth_expires=(datetime.utcnow() + timedelta(hours=24)).isoformat()
    )
    return new_token
```

**统一获取请求头：**

```python
def get_shopify_headers(store):
    if store.auth_mode == "token":
        token = decrypt(store.api_token)
    else:
        token = get_oauth_token(store)
    return {
        "X-Shopify-Access-Token": token,
        "Content-Type": "application/json"
    }
```

#### OAuth Token 生命周期

| 凭证 | 有效期 | 处理方式 |
|------|-------|---------|
| Access Token | 24 小时 | 过期自动用 client_id + secret 重新获取 |
| Client ID / Secret | 永久 | 加密存储在数据库中 |

**401 处理：** OAuth 模式收到 401 时，清除缓存的 oauth_token，用 client credentials 重新获取一次。若仍然 401，说明 client_secret 可能已失效，标记店铺为"凭证过期"。

### 7.2 接口列表

#### 前端端点（日常同步，无需认证）

**产品列表（检测新产品）：**
```
GET https://{storefront_domain}/products.json?limit=250&page=1
GET https://{storefront_domain}/products.json?limit=250&page=2
...
```
返回所有已发布产品的 id、handle、title、body_html、product_type、tags、images、variants。分页直到返回空数组。

**单个产品详情（已有产品更新检测）：**
```
GET https://{storefront_domain}/products/{handle}.json
```
返回完整产品 JSON（不含 SEO metafields）。

**SEO 信息（从前端 HTML 解析）：**
```
GET https://{storefront_domain}/products/{handle}
```
解析 HTML 提取：
- `<title>` 标签 → SEO标题
- `<meta name="description" content="...">` → SEO描述

#### Admin API 端点（新产品入库 + 回写时使用）

**获取单个产品完整数据（新产品入库时需要 status、完整 variants 等前端没有的字段）：**
```
GET https://{shop}.myshopify.com/admin/api/2024-10/products/{id}.json
```

**获取产品所属集合（只读）：**
```
GET https://{shop}.myshopify.com/admin/api/2024-10/collects.json?product_id={id}
GET https://{shop}.myshopify.com/admin/api/2024-10/collections/{collection_id}.json
```
> 集合名称只做展示，不支持回写。如需修改需在 Shopify 后台手动操作。

**更新产品（回写）：**
```
PUT https://{shop}.myshopify.com/admin/api/2024-10/products/{id}.json
Body: {
  "product": {
    "id": {id},
    "title": "...",
    "body_html": "...",
    "product_type": "...",
    "tags": "tag1, tag2, tag3",
    "handle": "new-handle",
    "variants": [{"id": {variant_id}, "option1": "..."}],
    "images": [{"id": {image_id}, "alt": "..."}]
  }
}
```

**更新 SEO 元字段（回写）：**
```
PUT https://{shop}.myshopify.com/admin/api/2024-10/products/{id}/metafields.json
Body: {"metafield": {"namespace": "global", "key": "title_tag", "value": "...", "type": "single_line_text_field"}}
```
SEO标题和SEO描述各需一次 PUT 请求。

#### 数据来源总结

| 数据 | 来源 | 需要 Token |
|------|------|-----------|
| 产品列表（检测新品） | 前端 `/products.json` | 否 |
| 产品字段（title、描述、tags等） | 前端 `/products/{handle}.json` | 否 |
| SEO 标题和描述 | 前端 HTML `<title>` + `<meta>` | 否 |
| 产品图片 | 前端 `/products/{handle}.json` → images + variants | 否 |
| 产品状态、所属集合 | Admin API | 是（仅新产品入库时） |
| 回写（所有字段） | Admin API | 是 |

### 7.3 速率限制处理

- Admin API：2 请求/秒，响应头 `X-Shopify-Shop-Api-Call-Limit` 格式如 `32/40`
- 当剩余额度 < 5 时 sleep 1 秒
- 收到 429 时指数退避重试（1s → 2s → 4s），最多 3 次
- 前端 JSON 请求间隔 1-2 秒，加 0-0.5 秒随机抖动

### 7.4 代理使用

所有对 Shopify 的请求（Admin API + 前端）都走该店铺配置的代理：

```python
def get_proxies(store):
    return {"http": store.proxy, "https": store.proxy}
```

飞书 API 请求（lark-cli）不走代理。

### 7.5 API Version

代码中硬编码 `2024-10`，定义为常量。Shopify 每个版本支持约 12 个月，届时更新常量即可。

> **注意：** Shopify 从 2024.10 起将 REST Admin API 标记为 legacy，新的公共应用要求使用 GraphQL。本项目为私有应用，REST API 目前仍可正常使用。如后续 Shopify 强制迁移，需将产品查询和更新改为 GraphQL 实现。

## 8. 同步逻辑

### 8.1 同步流程

系统为每个店铺提供**两档扫描频率**，在管理后台分别设置：

| 扫描类型 | 默认频率 | 做什么 | 数据来源 |
|---------|---------|-------|---------|
| **轻量扫描** | 每天 1-3 次 | 检测新产品、基础字段变化（标题、描述、标签、类型等） | 前端 `/products.json` |
| **深度扫描** | 每 2-3 天 1 次 | 逐个产品检测 SEO、图片 Alt、所属集合 | 前端 HTML + Admin API |

轻量扫描不访问单个产品页面，只请求 `/products.json` 列表；深度扫描才会逐个访问 `/products/{handle}` 拿 SEO 信息。

#### 轻量扫描流程

```
light_sync(store_id)
    │
    ▼
获取文件锁
    │
    ▼
通过代理请求前端 /products.json（分页拿完所有产品）
GET https://{storefront}/products.json?limit=250&page=1
（无需 Token，返回 id、handle、title、body_html、tags、product_type、images、variants）
    │
    ▼
从飞书多维表读取该店铺的现有记录
    │
    ▼
在内存中构建映射：{shopify_product_id: record_id}
    │
    ▼
对比产品列表
    │
    ├─── 新产品（前端有，多维表没有）
    │    │
    │    ▼
    │    从前端 JSON 已有数据提取：title、body_html、tags、product_type、handle、images、variants
    │    额外调 Admin API 获取 status（前端只有已发布产品）
    │    额外请求前端 HTML 获取 SEO 标题和描述
    │    额外调 Admin API collects 获取所属集合
    │    │
    │    ▼
    │    写入多维表（含产品图片和 Variant 图片）
    │    合规状态 = "待检查"，回写状态 = "未回写"
    │
    ├─── 已有产品
    │    │
    │    ▼
    │    跳过回写保护期内的记录（回写状态=已回写 且 24小时内）
    │    │
    │    ▼
    │    对比前端 JSON 中的字段与多维表（title、body_html、tags、product_type、images alt、variants option）
    │    ├── 任一字段有变化 → 更新记录 + 合规状态重置为"待检查"
    │    └── 均无变化 → 跳过
    │
    └─── 已删除产品（多维表有，前端没有）
         → 标记产品状态为"已删除"

释放文件锁
```

#### 深度扫描流程（分批循环）

```
deep_sync(store_id)
    │
    ▼
获取文件锁
    │
    ▼
从飞书多维表读取该店铺所有产品记录（按 Shopify产品ID 排序）
total = 总产品数
batch_size = ceil(total / store.deep_cycle)  # 如 1000 产品 / 3 天 = 每天 334 个
offset = store.deep_offset  # 上次扫到哪里了
    │
    ▼
判断是否首次（deep_offset == 0 且多维表中无深度扫描记录）
    ├── 首次 → 全量扫描所有产品（不分批）
    └── 非首次 → 取本批次产品：products[offset : offset + batch_size]
    │
    ▼
对本批次每个产品（按 handle 逐个，间隔 1-2 秒）：
    │
    ├── 请求前端 HTML
    │   GET https://{storefront}/products/{handle}
    │   解析 <title> → SEO标题
    │   解析 <meta name="description"> → SEO描述
    │   │
    │   ├── SEO 字段有变化 → 更新多维表，合规状态重置为"待检查"
    │   └── 无变化 → 跳过
    │
    ├── 请求前端 JSON
    │   GET https://{storefront}/products/{handle}.json
    │   检查图片 alt、variant option 变化
    │   更新产品图片和 Variant 图片字段
    │
    └── （新产品入库时已拉过集合，后续不再重复拉）
    │
    ▼
更新 deep_offset：
    new_offset = offset + batch_size
    如果 new_offset >= total → 重置为 0（一轮扫完，下轮从头开始）
    否则 → 保存 new_offset
    │
    ▼
释放文件锁
```

**示例：1000 个产品，deep_cycle = 3**
- 第 1 天：扫 #1 - #334，deep_offset 更新为 334
- 第 2 天：扫 #335 - #667，deep_offset 更新为 667
- 第 3 天：扫 #668 - #1000，deep_offset 重置为 0
- 第 4 天：从头开始，扫 #1 - #334

### 8.2 数据映射

**Admin API → 多维表字段映射：**

| Admin API 字段 | 多维表字段 | 备注 |
|---------------|-----------|------|
| `product.id` | Shopify产品ID | 大整数，存为文本类型防溢出 |
| `product.status` | 产品状态 | |
| `product.variants[0].price` | 价格 | variants 可能为空，需防御 |
| `product.variants[0].sku` | SKU | 同上 |
| 拼接生成 | 前端链接 | |
| `product.images[*].src` | 产品图片 | 所有主图 URL，换行分隔 |
| `product.variants[*].featured_image.src` | Variant图片 | 每行：`option1 \| URL`，无图则跳过 |

**风控可扫描字段映射（原始值）：**

| Admin API 字段 | 多维表字段 | 备注 |
|---------------|-----------|------|
| `product.title` | 产品标题 | |
| `product.body_html` | 原始描述HTML | 从 `/products.json` 或 `/products/{handle}.json` |
| html2text(body_html) | 原始描述纯文本 | 辅助阅读 |
| `product.product_type` | 产品类型 | 从 `/products.json` |
| `product.tags` | 产品标签 | 逗号分隔字符串，从 `/products.json` |
| HTML `<title>` | SEO标题 | 从前端 HTML `GET /products/{handle}` 解析 `<title>` 标签 |
| HTML `<meta name="description">` | SEO描述 | 从前端 HTML 解析 meta description |
| `product.handle` | Handle | 从 `/products.json` |
| `product.images[*].alt` | 图片Alt文本 | 所有图片 alt，换行分隔 |
| `product.variants[*].option1` | 规格名称 | 所有 variant 的 option 值，换行分隔 |
| collects → collection.title | 所属集合 | 需 Admin API collects 端点，只读展示不回写 |

### 8.3 lark-cli 命令（实际可用语法）

> **重要：** lark-cli `+record-list` 没有 `--filter` 参数。过滤通过 `--jq` 客户端过滤或 `+data-query` DSL 查询实现。

```bash
# 读取记录（分页，客户端 jq 过滤）
# ⚠️ store_name 必须先经过 validate_store_name() 校验（只允许 [a-zA-Z0-9_-]）
# 再用 json.dumps() 安全转义后拼入 jq 表达式，禁止直接 f-string 插值
lark-cli base +record-list \
  --base-token {base_token} \
  --table-id {table_id} \
  --limit 100 \
  --offset 0 \
  --jq '.items[] | select(.fields.店铺名称 == "{store_name}")'

# 批量创建记录
lark-cli base +record-batch-create \
  --base-token {base_token} \
  --table-id {table_id} \
  --json '{"fields":["店铺名称","Shopify产品ID","Handle",...],"rows":[["US-01","12345","handle-1",...]]}'

# 批量更新记录
lark-cli base +record-batch-update \
  --base-token {base_token} \
  --table-id {table_id} \
  --json '{"record_id_list":["recXXX"],"patch":{"原始描述HTML":"...","合规状态":"待检查"}}'

# 高级查询（用 DSL 过滤，回写时用）
lark-cli base +data-query \
  --base-token {base_token} \
  --dsl '{"table_id":"{table_id}","filter":{"conjunction":"and","conditions":[{"field_name":"合规状态","operator":"is","value":["已改写"]},{"field_name":"回写状态","operator":"is_not","value":["已回写"]}]}}'
```

### 8.4 lark-cli 调用规范

**分页处理：**

```python
records = []
offset = 0
limit = 100
while True:
    cmd = [
        "lark-cli", "base", "+record-list",
        "--base-token", base, "--table-id", table,
        "--limit", str(limit), "--offset", str(offset)
    ]
    result = subprocess.run(cmd, capture_output=True, text=True, timeout=30)
    if result.returncode != 0:
        raise RuntimeError(f"lark-cli error: {result.stderr}")
    data = json.loads(result.stdout)
    items = data.get("items", [])
    records.extend(items)
    if len(items) < limit:
        break
    offset += limit
```

**JSON 构建安全规范：**

所有传给 lark-cli 的 JSON 参数必须使用 `json.dumps()` 构建，禁止字符串拼接：

```python
# 正确
fields_json = json.dumps({
    "店铺名称": store_name,
    "Shopify产品ID": str(product_id),  # 文本类型
    "原始描述HTML": body_html
}, ensure_ascii=False)
cmd = ["lark-cli", "base", "+record-batch-create", "--json", fields_json, ...]

# 错误 — 禁止
cmd = [..., "--json", f'{{"店铺名称":"{store_name}"}}']
```

**subprocess 调用规范：**
- 永远使用列表形式，**禁止 `shell=True`**
- 检查 `returncode != 0` 时记录 stderr（脱敏后）
- 超时设为 30 秒

**记录 ID 映射：** lark-cli 返回的每条记录包含 `record_id` 字段。同步时在内存中构建 `{shopify_product_id: record_id}` 映射。如果出现重复记录（网络重试导致），以第一条为准，记录告警日志。

### 8.5 已删除产品处理

如果 Shopify Admin API 返回的产品列表中不包含多维表中已有的某个产品 ID：
- 不删除多维表记录
- 将该记录的"产品状态"更新为"已删除"
- 跳过前端描述抓取

### 8.6 Handle 变更处理

Shopify 允许修改产品 handle，导致前端 URL 变化。同步时从 Admin API 轻量列表中获取最新 handle（已包含在 fields 中），如果与多维表中的 handle 不同，同时更新 handle 和前端链接字段。

## 9. 回写逻辑

```
writeback()
    │
    ▼
遍历所有启用的店铺，对每个店铺的 table_id：
    │
    ▼
从飞书多维表查找（使用 +data-query DSL）：
  合规状态 == "已改写" AND 回写状态 != "已回写"
    │
    ▼
对每条记录：
    ├── 检查：至少有一个改写字段不为空，否则跳过
    ├── 从记录中读取：Shopify产品ID、所有改写字段、店铺名称
    ├── **校验 Shopify产品ID 为纯数字（validate_product_id()），不合法则跳过并告警**
    ├── **校验所有改写字段：长度限制、格式检查（Handle 必须匹配 [a-z0-9-]）**
    ├── 根据店铺名称从 SQLite 获取认证凭证和 proxy
    ├── 获取文件锁 /tmp/shopify_sync_{store_id}.lock
    ├── 构建回写请求（可能需要多个 API 调用）：
    │
    │   ── 产品主体更新（一次 PUT）──
    │   body = {"product": {"id": product_id}}
    │   如果改写标题不为空     → body["product"]["title"] = 改写标题（限 255 字符）
    │   如果改写描述不为空     → body["product"]["body_html"] = bleach.clean(改写描述, tags=ALLOWED_TAGS)
    │   如果改写产品类型不为空 → body["product"]["product_type"] = 改写产品类型
    │   如果改写标签不为空     → body["product"]["tags"] = 改写标签
    │   如果改写Handle不为空   → body["product"]["handle"] = 改写Handle
    │   如果改写图片Alt不为空  → body["product"]["images"] = [{id, alt}, ...]
    │   如果改写规格名称不为空 → body["product"]["variants"] = [{id, option1}, ...]
    │   PUT /admin/api/2024-10/products/{id}.json
    │
    │   ── SEO 元字段更新（最多两次 PUT）──
    │   如果改写SEO标题不为空  → PUT metafield global.title_tag
    │   如果改写SEO描述不为空  → PUT metafield global.description_tag
    │
    ├── 全部成功 → 更新多维表（单次 batch-update）：
    │         回写状态 = "已回写"
    │         将每个有值的改写字段同步到对应的原始字段
    │         （防止下次同步误判为变化）
    │         最后更新时间 = now
    └── 任一失败 → 更新多维表：回写状态 = "回写失败"
              写入 sync_logs（脱敏）
    │
    释放文件锁
```

回写由调度器每小时执行一次。

## 10. 调度器设计（单进程架构）

Web 后台和调度器运行在**同一个进程**中，避免跨进程锁的复杂性：

```python
# scripts/scheduler.py
from app.web import create_app
from apscheduler.schedulers.background import BackgroundScheduler

app = create_app()
scheduler = BackgroundScheduler()

with app.app_context():
    # 加载店铺配置，创建同步任务
    for store in get_enabled_stores():
        # 轻量扫描（检测新产品+基础字段）
        scheduler.add_job(
            light_sync,
            CronTrigger.from_crontab(store.light_cron),
            args=[store.id],
            id=f"light_{store.id}"
        )
        # 深度扫描（SEO + 图片Alt，每天执行一批）
        scheduler.add_job(
            deep_sync,
            CronTrigger(hour=store.deep_hour, minute=0),
            args=[store.id],
            id=f"deep_{store.id}"
        )

    # 全局回写任务
    scheduler.add_job(writeback, CronTrigger.from_crontab("0 * * * *"), id="writeback")

    scheduler.start()
    # 由 gunicorn 启动：gunicorn -w 1 --threads 4 -b 0.0.0.0:8080 "scripts.scheduler:create_app()"
    # 禁止使用 app.run()（Flask 开发服务器不适合生产环境）
    return app
```

**热更新机制：** Web 后台修改店铺配置后，通过 `scheduler.reschedule_job()` 或 `remove_job()` + `add_job()` 直接操作同进程的 scheduler 实例，无需进程间通信。

**并发保护：** 使用文件锁（`fcntl.flock`）按店铺粒度加锁，同一店铺的同步和回写不会并行。文件锁天然支持同进程内多线程场景。

**手动同步防阻塞：** "立即同步"按钮的 API 端点使用 `fcntl.flock(fd, LOCK_EX | LOCK_NB)` 非阻塞锁。如果同步已在进行，立即返回 HTTP 409 Conflict 和"同步进行中，请稍后"提示，避免 HTTP 请求长时间挂起。

**OAuth Token 刷新锁：** `get_oauth_token()` 内部使用独立的文件锁 `/tmp/shopify_oauth_{store_id}.lock`，防止同步和回写同时刷新 token 导致竞态覆盖。

## 11. 飞书多维表字段设计

默认所有店铺共用一张表，通过"店铺名称"字段区分。如需分表，可为每个店铺配置不同的 table_id。唯一键为"店铺名称 + Shopify产品ID"组合（代码层面去重，多维表不支持唯一约束）。

**基础信息字段：**

| 序号 | 字段名 | 字段类型 | 写入方 | 说明 |
|-----|-------|---------|-------|------|
| 1 | 店铺名称 | 单选 | 同步脚本 | 区分店铺来源 |
| 2 | Shopify产品ID | 文本 | 同步脚本 | 大整数存为文本防溢出 |
| 3 | 产品状态 | 单选 | 同步脚本 | active / draft / archived / 已删除 |
| 4 | 价格 | 数字 | 同步脚本 | 首个 variant 的价格 |
| 5 | SKU | 文本 | 同步脚本 | |
| 6 | 前端链接 | 链接 | 同步脚本 | `https://{domain}/products/{handle}` |
| 7 | 产品图片 | 文本 | 同步脚本 | 所有主图 URL，换行分隔 |
| 8 | Variant图片 | 文本 | 同步脚本 | 每行：`规格名 \| 图片URL` |

**风控可扫描字段（原始 + 改写成对）：**

| 序号 | 原始字段 | 改写字段 | 类型 | 说明 |
|-----|---------|---------|------|------|
| 9/10 | 产品标题 | 改写标题 | 文本 | |
| 11/12 | 原始描述HTML | 改写描述 | 文本 | 改写为 HTML 格式 |
| 13 | 原始描述纯文本 | — | 文本 | 辅助阅读，无需改写 |
| 14/15 | 产品类型 | 改写产品类型 | 文本 | 如 Supplement → Personal Care |
| 16/17 | 产品标签 | 改写标签 | 文本 | 逗号分隔 |
| 18/19 | SEO标题 | 改写SEO标题 | 文本 | 从前端 HTML meta 标签提取 |
| 20/21 | SEO描述 | 改写SEO描述 | 文本 | 同上 |
| 22/23 | Handle | 改写Handle | 文本 | URL slug |
| 24/25 | 图片Alt文本 | 改写图片Alt | 文本 | 所有图片 alt，换行分隔 |
| 26/27 | 规格名称 | 改写规格名称 | 文本 | 所有 variant option，换行分隔 |
| 28 | 所属集合 | — | 文本 | 只读展示，需在 Shopify 后台手动修改 |

**状态字段：**

| 序号 | 字段名 | 字段类型 | 写入方 | 说明 |
|-----|-------|---------|-------|------|
| 29 | 合规状态 | 单选 | 同步脚本/外部 | 待检查 / 合规 / 已改写 |
| 30 | 回写状态 | 单选 | 同步脚本 | 未回写 / 已回写 / 回写失败 |
| 31 | 最后更新时间 | 日期时间 | 同步脚本 | |

**状态流转：**

```
新产品入库 → 合规状态: 待检查, 回写状态: 未回写
任一可扫描字段变化 → 合规状态: 待检查（重置）
外部改合规 → 合规状态: 合规（无需回写）
外部改已改写 + 写入改写字段 → 合规状态: 已改写 → 触发回写
回写成功 → 回写状态: 已回写, 所有有改写值的原始字段同步更新
回写失败 → 回写状态: 回写失败
需重新回写 → 手动将回写状态改回"未回写"即可再次触发
回写24小时后前端描述仍不同 → 解除跳过保护，重新检测变化
```

## 12. 多维表数据交换接口

本程序通过飞书多维表与外部程序交换数据：

```
本程序                    飞书多维表                  外部程序
  │                          │                         │
  │── 写入产品数据 ──────────▶│                         │
  │   (字段1-12,15,16,17)    │                         │
  │                          │                         │
  │                          │◀── 读取标题+描述 ────────│
  │                          │    (字段4,12)            │
  │                          │                         │
  │                          │◀── 写入改写结果 ─────────│
  │                          │    (字段13,14,15)        │
  │                          │                         │
  │◀── 读取改写结果 ──────────│                         │
  │   (字段13,14,15,16)      │                         │
  │                          │                         │
  │── 回写Shopify后更新 ────▶│                         │
  │   (字段4,11,12,16,17)    │                         │
```

**改写字段格式要求：**
- 改写描述：HTML 格式，回写前经过 `bleach.clean()` 消毒（只允许安全标签：`p, br, strong, em, ul, ol, li, h1-h6, a, img, span, div, table, tr, td, th`），再写入 Shopify 的 `body_html`
- 改写标签：逗号分隔，如 `tag1, tag2, tag3`
- 图片Alt、规格名称：换行分隔，每行对应一个图片/规格
- 其他改写字段：纯文本
- 不需要改写的字段留空即可，系统只回写有内容的改写字段

## 13. 错误处理

| 场景 | 处理方式 |
|------|---------|
| Shopify API 429 | 指数退避重试（1s → 2s → 4s），最多 3 次 |
| Shopify API 401（Token模式） | 记录日志，标记该店铺 token 失效，跳过 |
| Shopify API 401（OAuth模式） | 清除 oauth_token 缓存，用 client credentials 重新获取；若仍 401，标记"凭证过期" |
| 前端 JSON 404 | 产品可能已下架，跳过该产品，不删除多维表记录 |
| 前端 JSON 超时 | 重试 2 次，仍失败则跳过 |
| lark-cli 执行失败 | 重试 2 次（间隔 2s → 5s），仍失败则记录日志（脱敏）并终止该店铺同步 |
| 代理连接失败 | 记录日志，标记该店铺代理异常，跳过 |
| 单条产品处理异常 | try-except 包裹，记录失败产品 ID，继续处理下一条 |
| 回写失败 | 更新回写状态为"回写失败"，记录原因到日志（脱敏） |
| product.variants 为空 | 价格和 SKU 字段填空，不报错 |
| product.image 为 null | 主图URL 填空，不报错 |
| 重复记录（创建时网络重试） | 创建前检查 ID 映射，已存在则改为 update |

## 14. 部署（macOS / Mac mini）

### 14.1 环境要求

- Mac mini（本地部署）
- Python 3.8+（macOS 自带或 Homebrew 安装）
- 如需外网访问管理后台：Nginx + 自签证书 或 Caddy（自动 TLS）
- 如仅局域网访问：可直接访问 `http://localhost:8080`，跳过 Nginx
- lark-cli 已安装并登录（`lark-cli auth login`）
- 每个店铺一个独立代理 IP

### 14.2 部署步骤

```bash
# 1. 拉取代码
git clone <repo-url> ~/shopifyproduct
cd ~/shopifyproduct

# 2. 安装依赖
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 3. 创建 .env（一次性写入，非追加）
FERNET_KEY=$(python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())")
ADMIN_PASS_HASH=$(python -c "import bcrypt; print(bcrypt.hashpw(b'你的强密码', bcrypt.gensalt()).decode())")

cat > ~/shopifyproduct/.env << EOF
FERNET_KEY=${FERNET_KEY}
ADMIN_USER=admin
ADMIN_PASS_HASH=${ADMIN_PASS_HASH}
SECRET_KEY=$(python -c "import secrets; print(secrets.token_hex(32))")
EOF

chmod 600 ~/shopifyproduct/.env

# ⚠️ 备份 FERNET_KEY！丢失后所有加密凭证不可恢复
echo "请将 FERNET_KEY 备份到安全位置：${FERNET_KEY}"

# 4. 设置数据库目录权限
mkdir -p ~/shopifyproduct/data
chmod 700 ~/shopifyproduct/data

# 5. 部署 launchd 服务（开机自启 + 崩溃自动重启）
cp deploy/com.shopifyproduct.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.shopifyproduct.plist

# 查看状态
launchctl list | grep shopifyproduct

# 查看日志
tail -f ~/shopifyproduct/logs/app.log
tail -f ~/shopifyproduct/logs/app_error.log
```

### 14.3 launchd 服务文件（deploy/com.shopifyproduct.plist）

macOS 使用 launchd 替代 systemd，功能等价：开机自启、崩溃重启、日志管理。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.shopifyproduct</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Users/adao/shopifyproduct/venv/bin/gunicorn</string>
        <string>-w</string>
        <string>1</string>
        <string>--threads</string>
        <string>4</string>
        <string>-b</string>
        <string>0.0.0.0:8080</string>
        <string>--timeout</string>
        <string>120</string>
        <string>scripts.scheduler:create_app()</string>
    </array>

    <key>WorkingDirectory</key>
    <string>/Users/adao/shopifyproduct</string>

    <!-- 从 .env 文件加载环境变量（launchd 不支持 EnvironmentFile，需逐条列出或用 wrapper） -->
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/Users/adao/shopifyproduct/venv/bin:/usr/local/bin:/usr/bin:/bin</string>
    </dict>

    <!-- 开机自启 -->
    <key>RunAtLoad</key>
    <true/>

    <!-- 崩溃自动重启，间隔 5 秒 -->
    <key>KeepAlive</key>
    <true/>
    <key>ThrottleInterval</key>
    <integer>5</integer>

    <!-- 日志输出 -->
    <key>StandardOutPath</key>
    <string>/Users/adao/shopifyproduct/logs/app.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/adao/shopifyproduct/logs/app_error.log</string>
</dict>
</plist>
```

> **注意：** launchd 不支持 `EnvironmentFile`。实际实现中用一个 wrapper 脚本（`scripts/start.sh`）加载 `.env` 后启动 gunicorn：

```bash
#!/bin/bash
# scripts/start.sh — launchd 启动入口
set -a
source /Users/adao/shopifyproduct/.env
set +a

cd /Users/adao/shopifyproduct
exec /Users/adao/shopifyproduct/venv/bin/gunicorn \
    -w 1 --threads 4 \
    -b 0.0.0.0:8080 \
    --timeout 120 \
    "scripts.scheduler:create_app()"
```

对应 plist 中 `ProgramArguments` 改为：

```xml
<key>ProgramArguments</key>
<array>
    <string>/bin/bash</string>
    <string>/Users/adao/shopifyproduct/scripts/start.sh</string>
</array>
```

**常用管理命令：**

```bash
# 启动
launchctl load ~/Library/LaunchAgents/com.shopifyproduct.plist

# 停止
launchctl unload ~/Library/LaunchAgents/com.shopifyproduct.plist

# 重启（先停后启）
launchctl unload ~/Library/LaunchAgents/com.shopifyproduct.plist
launchctl load ~/Library/LaunchAgents/com.shopifyproduct.plist

# 查看状态（0 = 正常运行，非 0 = 异常）
launchctl list | grep shopifyproduct
```

### 14.4 环境变量模板（deploy/.env.example）

```bash
# 加密密钥（首次部署时生成，丢失不可恢复）
FERNET_KEY=

# 管理员登录（ADMIN_PASS_HASH 为 bcrypt 哈希，非明文密码）
ADMIN_USER=admin
ADMIN_PASS_HASH=

# Flask session 密钥
SECRET_KEY=
```

### 14.5 Nginx 配置（可选）

gunicorn 默认绑定 `0.0.0.0:8080`，局域网内通过 `http://mac-mini-ip:8080` 直接访问，无需 Nginx。

如果需要**公网访问**，建议用 Caddy（自动 HTTPS）或 Nginx + 自签证书：

```nginx
server {
    listen 443 ssl http2;
    server_name your-admin-domain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # TLS 加固
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # 安全头
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    proxy_hide_header X-Powered-By;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name your-admin-domain.com;
    return 301 https://$host$request_uri;
}
```

### 14.6 日志

- 同步日志存入 SQLite `sync_logs` 表，后台面板可查看
- 所有日志记录前脱敏（过滤 token、secret 等敏感字段）
- 应用日志输出到 `~/shopifyproduct/logs/app.log` 和 `app_error.log`
- 建议配合 macOS 自带的 `newsyslog` 或定期手动清理日志文件

### 14.7 健康检查

```
GET /healthz  → 无需登录
```

检查项：
- SQLite 可读写
- APScheduler 正在运行
- 每个启用店铺的最后同步时间是否超过预设频率的 2 倍

返回 200（健康）或 503（异常）+ JSON 详情。可配合 UptimeRobot 等外部监控。

### 14.8 SQLite 备份

macOS 使用 launchd 定时任务替代 cron：

```bash
# 简单方案：添加到用户 crontab
crontab -e
# 添加以下行（每天凌晨 4 点备份）
0 4 * * * sqlite3 ~/shopifyproduct/data/stores.db ".backup ~/shopifyproduct/data/stores_backup.db"
```

> **关键提醒：** `FERNET_KEY` 丢失后所有加密的 API 凭证不可恢复。部署后务必将 FERNET_KEY 备份到独立的安全位置（如密码管理器）。

### 14.9 Shopify API 版本监控

代码启动时检查：如果当前日期距 API 版本（如 `2024-10`）发布已超过 10 个月，在日志中输出 WARNING 提醒升级。

### 14.10 Mac mini 注意事项

- **防休眠：** 系统设置 → 节能 → 勾选"防止自动进入睡眠"，否则休眠后定时任务停止
- **开机自启：** launchd plist 放在 `~/Library/LaunchAgents/` 下 + `RunAtLoad=true`，用户登录后自动启动
- **网络稳定性：** Mac mini 建议使用有线网络连接，Wi-Fi 休眠后可能断开
- **macOS 更新：** 系统更新重启后 launchd 会自动重新加载服务，无需手动操作

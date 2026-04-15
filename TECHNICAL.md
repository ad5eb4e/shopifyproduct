# Shopify 代理中台 — 技术文档

## 1. 系统架构

系统分为两层：**中台核心层**（店铺管理 + Shopify API 抽象）和**业务应用层**（同步、回写等具体任务）。

```
┌─────────────────────────────────────────────────────────────┐
│                    Mac mini（本地部署）                        │
│                                                              │
│  ┌─────────────────── 业务应用层 ──────────────────────────┐ │
│  │                                                          │ │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐ │ │
│  │  │ 产品同步    │  │ 合规回写    │  │ 未来应用            │ │ │
│  │  │ light_sync │  │ writeback  │  │ product_copy       │ │ │
│  │  │ deep_sync  │  │            │  │ report_export      │ │ │
│  │  └─────┬──────┘  └─────┬──────┘  └────────┬───────────┘ │ │
│  │        │               │                   │             │ │
│  │        │    ┌───────────┴──────┐            │             │ │
│  │        │    │  lark_writer     │            │             │ │
│  │        │    │  (飞书多维表读写) │            │             │ │
│  │        │    └──────────────────┘            │             │ │
│  └────────┼───────────────────────────────────┼─────────────┘ │
│           ▼                                   ▼               │
│  ┌─────────────────── 中台核心层 ──────────────────────────┐  │
│  │                                                          │  │
│  │  ┌──────────────┐     ┌───────────────────────────────┐ │  │
│  │  │  Web 管理后台 │     │  shopify_client               │ │  │
│  │  │  (gunicorn)   │     │  ● get_client(store) → Client │ │  │
│  │  │  - 店铺 CRUD  │     │  ● 自动认证（Token/OAuth）     │ │  │
│  │  │  - 状态查看   │     │  ● 自动代理路由                │ │  │
│  │  └──────┬───────┘     │  ● 限速 + 重试                │ │  │
│  │         │              └──────────────┬────────────────┘ │  │
│  │         ▼                             │                   │  │
│  │  ┌──────────────┐     ┌───────────────┴───────────────┐  │  │
│  │  │  SQLite(WAL) │     │  定时调度器 (APScheduler)      │  │  │
│  │  │  - stores    │     │  - 按店铺/应用配置触发          │  │  │
│  │  │  - settings  │     │  - 注册/注销任务               │  │  │
│  │  │  - sync_logs │     └───────────────────────────────┘  │  │
│  │  └──────────────┘                                         │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│         ┌────────────────────────────────────────┐               │
│         │          代理路由（每店独立）             │               │
│         │    代理IP-A    代理IP-B    代理IP-C      │               │
│         └────────┬──────────┬──────────┬─────────┘               │
└──────────────────┼──────────┼──────────┼─────────────────────────┘
                   ▼          ▼          ▼
              Shopify-A  Shopify-B  Shopify-C
```

## 2. 技术栈

| 组件 | 技术 | 层级 |
|------|------|------|
| Web 后台 | Flask + SQLite | 中台 |
| WSGI 服务器 | gunicorn (gthread) | 中台 |
| 反向代理 | Nginx + Let's Encrypt | 中台（可选） |
| 前端页面 | 原生 HTML + JS | 中台 |
| 定时调度 | APScheduler | 中台 |
| HTTP 请求 | requests | 中台 |
| 数据存储 | SQLite (WAL 模式) | 中台 |
| Token 加密 | cryptography (Fernet) | 中台 |
| 密码哈希 | bcrypt | 中台 |
| 代理支持 | PySocks + requests | 中台 |
| 限速 | flask-limiter | 中台 |
| 安全头 | flask-talisman | 中台 |
| CSRF 保护 | flask-wtf (CSRFProtect) | 中台 |
| 进程管理 | launchd (macOS) | 中台 |
| 飞书写入 | lark-cli (subprocess) | 应用层 |
| HTML 消毒 | bleach | 应用层 |
| HTML 转文本 | html2text | 应用层 |

## 3. 项目结构

```
shopifyproduct/
  PRD.md                       # 需求文档（运营看）
  TECHNICAL.md                 # 技术文档（本文件）
  COMPLIANCE_PROMPT.md         # 合规改写规则（.gitignore 排除）
  requirements.txt             # Python 依赖
  .gitignore

  ── 中台核心层 ──

  core/
    __init__.py
    shopify_client.py          # Shopify API 统一客户端（认证、代理、限速）
    shopify_auth.py            # 认证逻辑（Token / OAuth 双模式）
    store_manager.py           # 店铺 CRUD 操作
    db.py                      # SQLite 数据库操作
    crypto.py                  # Token 加密解密
    validators.py              # 输入校验
    scheduler.py               # APScheduler 调度器管理

  web/
    __init__.py
    routes.py                  # Flask 路由（店铺管理 + 应用入口）
    templates/
      index.html               # 管理后台页面

  ── 业务应用层 ──

  apps/
    __init__.py
    sync/
      __init__.py
      light_sync.py            # 轻量扫描（新产品 + 基础字段变化）
      deep_sync.py             # 深度扫描（SEO + 图片Alt，分批循环）
      writeback.py             # 合规回写
      lark_writer.py           # lark-cli 命令封装
      models.py                # 同步相关数据结构
    # 未来应用在此添加
    # product_copy/
    # report_export/

  ── 启动与部署 ──

  scripts/
    start.sh                   # launchd 启动入口
    main.py                    # 应用入口（Flask + Scheduler）

  nginx/
    shopifyproduct.conf        # Nginx 配置模板

  deploy/
    com.shopifyproduct.plist   # macOS launchd 服务文件
    .env.example               # 环境变量模板

  logs/                        # 运行日志
  data/                        # SQLite 数据库文件（chmod 600）
```

## 4. 中台核心层

### 4.1 Shopify Client（`core/shopify_client.py`）

这是中台的核心——一个统一的 Shopify API 客户端。上层应用通过它访问 Shopify，无需关心认证、代理、限速等细节。

```python
class ShopifyClient:
    """每个 store 实例对应一个 client，自动处理认证和代理"""

    def __init__(self, store):
        self.store = store
        self._proxies = {"http": store.proxy, "https": store.proxy}

    # ── 前端操作（不需要 Token） ──

    def list_products(self) -> list:
        """分页获取所有产品列表（前端 /products.json）"""
        ...

    def get_product(self, handle: str) -> dict:
        """获取单个产品详情（前端 /products/{handle}.json）"""
        ...

    def get_product_seo(self, handle: str) -> dict:
        """从前端 HTML 解析 SEO 标题和描述"""
        ...

    # ── Admin API 操作（需要 Token） ──

    def get_product_admin(self, product_id: str) -> dict:
        """获取产品完整数据（含 status 等前端没有的字段）"""
        ...

    def get_product_collections(self, product_id: str) -> list:
        """获取产品所属集合"""
        ...

    def update_product(self, product_id: str, data: dict) -> bool:
        """更新产品（标题、描述、标签、Handle、图片Alt、规格等）"""
        ...

    def update_product_seo(self, product_id: str, title: str = None, description: str = None) -> bool:
        """更新 SEO 元字段"""
        ...

    # ── 内部方法 ──

    def _request_frontend(self, path: str) -> requests.Response:
        """前端请求（带代理，1-2秒间隔 + 随机抖动）"""
        ...

    def _request_admin(self, method: str, path: str, json=None) -> requests.Response:
        """Admin API 请求（带代理、认证头、限速处理）"""
        ...

    def _get_auth_headers(self) -> dict:
        """统一获取认证头（Token 直接用，OAuth 自动续期）"""
        ...


def get_client(store) -> ShopifyClient:
    """工厂方法：根据 store 配置创建客户端"""
    return ShopifyClient(store)
```

**关键设计：**
- 上层应用只需 `client = get_client(store)` 然后调用方法
- 认证、代理、限速全部在 client 内部处理
- 未来新应用（产品复制、报表导出）使用同一个 client

### 4.2 认证模块（`core/shopify_auth.py`）

#### 方式一：Legacy Token（`auth_mode = "token"`）

直接使用永久 Access Token：

```python
headers = {
    "X-Shopify-Access-Token": decrypt(store.api_token),
    "Content-Type": "application/json"
}
```

#### 方式二：OAuth Client Credentials（`auth_mode = "oauth"`）

```python
def get_oauth_token(store):
    """获取 OAuth access token，过期则重新申请"""
    # 1. 检查现有 token 是否还有效（提前 5 分钟刷新）
    if store.oauth_token and store.oauth_expires:
        expires = datetime.fromisoformat(store.oauth_expires)
        if datetime.utcnow() < expires - timedelta(minutes=5):
            return decrypt(store.oauth_token)

    # 2. 用 client credentials 获取新 token（Shopify 不返回 refresh token）
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
    resp.raise_for_status()
    data = resp.json()

    new_token = data["access_token"]
    update_store_oauth(store.id,
        oauth_token=encrypt(new_token),
        oauth_expires=(datetime.utcnow() + timedelta(hours=24)).isoformat()
    )
    return new_token
```

| 凭证 | 有效期 | 处理方式 |
|------|-------|---------|
| Access Token | 永久 | 直接使用 |
| OAuth Access Token | 24 小时 | 过期自动重新获取 |
| Client ID / Secret | 永久 | 加密存储 |

**401 处理：** OAuth 模式收到 401 时，清除 oauth_token，重新获取。若仍 401，标记"凭证过期"。

### 4.3 数据库设计（SQLite，WAL 模式）

初始化时执行 `PRAGMA journal_mode=WAL;`。

#### stores 表

```sql
CREATE TABLE stores (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    store_name     TEXT NOT NULL UNIQUE,          -- 只允许 [a-zA-Z0-9_-]
    shop           TEXT NOT NULL,                 -- myshopify.com 前缀
    storefront     TEXT NOT NULL,                 -- 前端域名
    auth_mode      TEXT NOT NULL DEFAULT 'token', -- 'token' 或 'oauth'
    api_token      TEXT,                          -- 加密后的 Admin API Token
    client_id      TEXT,                          -- OAuth Client ID（不加密）
    client_secret  TEXT,                          -- 加密后的 Client Secret
    oauth_token    TEXT,                          -- 加密后的当前 access token
    oauth_expires  TEXT,                          -- token 过期时间 ISO8601
    proxy          TEXT NOT NULL,                 -- 代理地址
    table_id       TEXT NOT NULL,                 -- 飞书 Table ID
    enabled        INTEGER NOT NULL DEFAULT 1,
    created_at     TEXT DEFAULT CURRENT_TIMESTAMP,
    updated_at     TEXT DEFAULT CURRENT_TIMESTAMP
);
```

#### app_configs 表（应用专属配置）

```sql
CREATE TABLE app_configs (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    store_id   INTEGER NOT NULL,
    app_name   TEXT NOT NULL,                    -- 应用标识，如 'sync'
    config     TEXT NOT NULL DEFAULT '{}',       -- JSON 格式的应用配置
    enabled    INTEGER NOT NULL DEFAULT 1,
    FOREIGN KEY (store_id) REFERENCES stores(id),
    UNIQUE(store_id, app_name)
);

-- 产品同步应用的 config 示例：
-- {
--   "light_cron": "0 3 * * *",
--   "deep_cycle": 3,
--   "deep_hour": 4,
--   "deep_offset": 0
-- }
```

> **设计思路：** 店铺基础信息（凭证、代理）属于中台，应用专属配置（扫描频率等）属于应用层。`app_configs` 表通过 `app_name` 区分不同应用，未来新应用只需插入新行，无需改表结构。

#### settings 表

```sql
CREATE TABLE settings (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);
-- 预设: lark_base_token（Fernet 加密存储）
```

#### sync_logs 表

```sql
CREATE TABLE sync_logs (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    store_id   INTEGER NOT NULL,
    app_name   TEXT NOT NULL DEFAULT 'sync',     -- 哪个应用产生的日志
    action     TEXT NOT NULL,                    -- sync / writeback / product_copy / ...
    status     TEXT NOT NULL,                    -- success / failed
    summary    TEXT,
    detail     TEXT,                             -- 完整日志（已脱敏）
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (store_id) REFERENCES stores(id)
);
```

#### SQL 安全规范

所有查询必须使用参数化：

```python
# 正确
cursor.execute("SELECT * FROM stores WHERE store_name = ?", (name,))

# 错误 — 禁止
cursor.execute(f"SELECT * FROM stores WHERE store_name = '{name}'")
```

### 4.4 输入校验（`core/validators.py`）

```python
import re, socket, ipaddress

STORE_NAME_RE = re.compile(r'^[a-zA-Z0-9_-]+$')
SHOP_RE = re.compile(r'^[a-z0-9-]+$')
PROXY_RE = re.compile(r'^(socks5|https?)://[a-zA-Z0-9._-]+:\d+$')
STOREFRONT_RE = re.compile(r'^[a-zA-Z0-9]([a-zA-Z0-9.-]*[a-zA-Z0-9])?$')
HANDLE_RE = re.compile(r'^[a-z0-9-]+$')
PRODUCT_ID_RE = re.compile(r'^\d+$')

def _is_private_ip(host: str) -> bool:
    """检查 IP 或主机名是否解析到私有/保留地址（防 SSRF）"""
    try:
        addr = ipaddress.ip_address(host)
        return addr.is_private or addr.is_loopback or addr.is_reserved or addr.is_link_local
    except ValueError:
        pass
    try:
        resolved = socket.getaddrinfo(host, None)
        for _, _, _, _, sockaddr in resolved:
            addr = ipaddress.ip_address(sockaddr[0])
            if addr.is_private or addr.is_loopback or addr.is_reserved or addr.is_link_local:
                return True
    except socket.gaierror:
        return True
    return False

def validate_store_name(name): return bool(STORE_NAME_RE.match(name)) and len(name) <= 50
def validate_shop(shop):       return bool(SHOP_RE.match(shop)) and len(shop) <= 100
def validate_proxy(proxy):
    if not PROXY_RE.match(proxy): return False
    host = proxy.split('://')[1].rsplit(':', 1)[0]
    return not _is_private_ip(host)
def validate_storefront(domain):
    if not STOREFRONT_RE.match(domain) or len(domain) > 253: return False
    try:
        ipaddress.ip_address(domain); return False
    except ValueError: pass
    return not _is_private_ip(domain)
def validate_product_id(pid):  return bool(PRODUCT_ID_RE.match(pid)) and len(pid) <= 20
def validate_handle(handle):   return bool(HANDLE_RE.match(handle)) and len(handle) <= 255
def validate_hour(hour):       return isinstance(hour, int) and 0 <= hour <= 23
```

### 4.5 Web 管理后台

#### 页面

单页面，内联表格编辑。分为两个区域：
1. **中台配置区：** 店铺基础信息（名称、标识、域名、认证、代理）
2. **应用配置区：** 根据启用的应用显示对应配置（如同步应用的扫描频率）

**店铺字段（中台）：**

| 字段 | 必填 | 说明 |
|------|------|------|
| 店铺名称 | 是 | 只允许字母、数字、连字符、下划线 |
| 店铺标识 | 是 | myshopify.com 前缀 |
| 前端域名 | 是 | 客户访问的自定义域名 |
| 认证方式 | 是 | Token / OAuth |
| Access Token | Token时必填 | 加密存储，页面星号遮蔽 |
| Client ID | OAuth时必填 | |
| Client Secret | OAuth时必填 | 加密存储，页面星号遮蔽 |
| 代理IP | 是 | `socks5://ip:port` 或 `http://ip:port` |
| 飞书 Table ID | 是 | 该店铺对应的多维表 table ID |

**同步应用字段（应用层）：**

| 字段 | 必填 | 说明 |
|------|------|------|
| 轻量扫描频率 | 是 | 每小时/每6小时/每天+时间 |
| 深度扫描周期 | 是 | 1/2/3/5/7 天 |
| 深度扫描时间 | 是 | 0-23 |

**操作：**
- 通用：保存 / 删除 / 测试连接 / 查看日志
- 同步应用：立即同步

#### 认证与安全

**Session 登录 + bcrypt + CSRF + TLS：**

```python
import bcrypt
from flask_login import LoginManager, login_required
from flask_wtf.csrf import CSRFProtect

csrf = CSRFProtect(app)
login_manager = LoginManager(app)

ADMIN_USER = os.environ["ADMIN_USER"]
ADMIN_PASS_HASH = os.environ["ADMIN_PASS_HASH"]  # bcrypt 哈希

# 启动时检测默认密码
if bcrypt.checkpw(b"changeme", ADMIN_PASS_HASH.encode()):
    sys.exit("ERROR: 请修改默认密码")

def verify_login(username, password):
    return (username == ADMIN_USER and
            bcrypt.checkpw(password.encode(), ADMIN_PASS_HASH.encode()))
```

- Session 超时 30 分钟
- 所有非 GET 端点强制 CSRF
- 登录限速：5次/15分钟
- Token 查看限速：5次/分钟
- 安全头：`Talisman(app, content_security_policy={"default-src": "'self'"})`

#### API 路由

```
── 中台路由 ──
GET    /                        → 管理后台页面（需登录）
GET    /login                   → 登录页面
POST   /login                   → 登录验证（限速 5次/15分钟）
GET    /healthz                 → 健康检查（无需登录）
GET    /api/stores              → 获取所有店铺（敏感字段星号化）
POST   /api/stores              → 新增店铺
PUT    /api/stores/<id>         → 更新店铺
DELETE /api/stores/<id>         → 删除店铺
POST   /api/stores/<id>/test   → 测试连接
GET    /api/stores/<id>/token  → 获取明文 token（限速，有审计日志）
GET    /api/stores/<id>/logs   → 获取日志
GET    /api/settings            → 获取全局设置
PUT    /api/settings            → 更新全局设置

── 同步应用路由 ──
GET    /api/stores/<id>/sync-config   → 获取同步配置
PUT    /api/stores/<id>/sync-config   → 更新同步配置
POST   /api/stores/<id>/sync          → 手动触发同步
```

#### 凭证安全

- `cryptography.fernet.Fernet` 对称加密
- 密钥从环境变量 `FERNET_KEY` 读取
- 加密字段：`api_token`, `client_secret`, `oauth_token`, `lark_base_token`
- `client_id` 不加密（非敏感）
- API 返回时敏感字段替换为星号
- 日志脱敏

### 4.6 Shopify API 对接（中台内部）

#### 接口列表

**前端端点（无需认证）：**

```
GET https://{storefront}/products.json?limit=250&page=1    # 产品列表
GET https://{storefront}/products/{handle}.json             # 产品详情
GET https://{storefront}/products/{handle}                  # HTML → SEO
```

**Admin API 端点（需认证）：**

```
GET  /admin/api/2024-10/products/{id}.json                  # 产品完整数据
GET  /admin/api/2024-10/collects.json?product_id={id}       # 所属集合
GET  /admin/api/2024-10/collections/{cid}.json              # 集合详情
PUT  /admin/api/2024-10/products/{id}.json                  # 更新产品
PUT  /admin/api/2024-10/products/{id}/metafields.json       # 更新 SEO
```

#### 速率限制

- Admin API：2 请求/秒，监控 `X-Shopify-Shop-Api-Call-Limit` 响应头
- 剩余额度 < 5 时 sleep 1 秒
- 429 指数退避重试（1s → 2s → 4s），最多 3 次
- 前端请求间隔 1-2 秒 + 0-0.5 秒随机抖动

#### 代理

所有 Shopify 请求走该店铺的代理：

```python
def get_proxies(store):
    return {"http": store.proxy, "https": store.proxy}
```

飞书 API（lark-cli）不走代理。

#### API Version

硬编码 `2024-10`，启动时检查是否即将过期（> 10 个月输出 WARNING）。

### 4.7 调度器（`core/scheduler.py`）

Web 后台和调度器运行在**同一个进程**中：

```python
from apscheduler.schedulers.background import BackgroundScheduler

class TaskScheduler:
    """中台调度器，管理所有应用的定时任务"""

    def __init__(self):
        self.scheduler = BackgroundScheduler()

    def register_app_tasks(self, store, app_name, tasks):
        """为某个店铺注册应用的定时任务
        tasks: [{"func": callable, "trigger": CronTrigger, "id_suffix": str}]
        """
        for task in tasks:
            job_id = f"{app_name}_{store.id}_{task['id_suffix']}"
            self.scheduler.add_job(
                task["func"], task["trigger"],
                args=[store.id], id=job_id
            )

    def unregister_app_tasks(self, store_id, app_name):
        """注销某个店铺某个应用的所有任务"""
        for job in self.scheduler.get_jobs():
            if job.id.startswith(f"{app_name}_{store_id}_"):
                job.remove()

    def start(self):
        self.scheduler.start()
```

同步应用注册任务示例：

```python
# apps/sync/ 注册到调度器
scheduler.register_app_tasks(store, "sync", [
    {
        "func": light_sync,
        "trigger": CronTrigger.from_crontab(config["light_cron"]),
        "id_suffix": "light"
    },
    {
        "func": deep_sync,
        "trigger": CronTrigger(hour=config["deep_hour"], minute=0),
        "id_suffix": "deep"
    }
])

# 全局回写（每小时）
scheduler.scheduler.add_job(writeback, CronTrigger.from_crontab("0 * * * *"), id="writeback")
```

**并发保护：** 文件锁（`fcntl.flock`）按店铺粒度加锁。手动同步用非阻塞锁 `LOCK_NB`。OAuth 刷新用独立锁。

---

## 5. 业务应用层：产品同步

### 5.1 轻量扫描（`apps/sync/light_sync.py`）

```
light_sync(store_id)
    │
    ▼
获取文件锁 → client = get_client(store)
    │
    ▼
client.list_products() — 前端 /products.json 分页拿完
    │
    ▼
从飞书多维表读取该店铺现有记录 → 构建 {product_id: record_id} 映射
    │
    ▼
对比产品列表
    │
    ├─── 新产品
    │    ├── 从前端 JSON 提取：title、body_html、tags、product_type、handle、images、variants
    │    ├── client.get_product_admin(id) → 获取 status
    │    ├── client.get_product_seo(handle) → 获取 SEO
    │    ├── client.get_product_collections(id) → 获取集合
    │    └── 写入多维表，合规状态 = "待检查"
    │
    ├─── 已有产品
    │    ├── 跳过回写保护期（已回写 且 24小时内）
    │    ├── 对比字段，有变化 → 更新 + 重置"待检查"
    │    └── 无变化 → 跳过
    │
    └─── 已删除产品 → 标记"已删除"

释放文件锁
```

### 5.2 深度扫描（`apps/sync/deep_sync.py`）

```
deep_sync(store_id)
    │
    ▼
获取文件锁 → client = get_client(store)
    │
    ▼
从飞书读取所有产品（按 ID 排序）
batch_size = ceil(total / deep_cycle)
    │
    ▼
首次（deep_offset == 0 且无记录）→ 全量扫描
非首次 → 取 products[offset : offset + batch_size]
    │
    ▼
对每个产品（间隔 1-2 秒）：
    ├── client.get_product_seo(handle) → SEO 变化检测
    ├── client.get_product(handle) → 图片 alt / variant option 变化
    └── 更新产品图片字段
    │
    ▼
更新 deep_offset（循环到头则重置为 0）

释放文件锁
```

### 5.3 合规回写（`apps/sync/writeback.py`）

```
writeback()
    │
    ▼
遍历所有启用的店铺：
    │
    ▼
从飞书查找：合规状态 == "已改写" AND 回写状态 != "已回写"
    │
    ▼
对每条记录：
    ├── 校验 product_id + 改写字段
    ├── client = get_client(store)
    ├── client.update_product(id, data) — 主体更新
    ├── client.update_product_seo(id, title, desc) — SEO 更新
    ├── 全部成功 → 回写状态 = "已回写" + 原始字段同步更新
    └── 任一失败 → 回写状态 = "回写失败"
```

### 5.4 数据映射

**Admin API → 多维表字段：**

| API 字段 | 多维表字段 | 备注 |
|----------|-----------|------|
| `product.id` | Shopify产品ID | 大整数存文本 |
| `product.title` | 产品标题 | |
| `product.body_html` | 原始描述HTML | |
| html2text(body_html) | 原始描述纯文本 | |
| `product.product_type` | 产品类型 | |
| `product.tags` | 产品标签 | 逗号分隔 |
| `product.handle` | Handle | |
| `product.status` | 产品状态 | |
| `product.variants[0].price` | 价格 | |
| `product.variants[0].sku` | SKU | |
| `product.images[*].src` | 产品图片 | 换行分隔 |
| `product.images[*].alt` | 图片Alt文本 | 换行分隔 |
| `product.variants[*].featured_image.src` | Variant图片 | `option1 \| URL` |
| `product.variants[*].option1` | 规格名称 | 换行分隔 |
| HTML `<title>` | SEO标题 | 前端 HTML 解析 |
| HTML `<meta description>` | SEO描述 | 前端 HTML 解析 |
| collects → collection.title | 所属集合 | 只读 |

### 5.5 lark-cli 调用规范

```bash
# 读取记录
lark-cli base +record-list \
  --base-token {base_token} --table-id {table_id} \
  --limit 100 --offset 0 \
  --jq '.items[] | select(.fields.店铺名称 == "{store_name}")'

# 批量创建
lark-cli base +record-batch-create \
  --base-token {base_token} --table-id {table_id} \
  --json '{...}'

# 批量更新
lark-cli base +record-batch-update \
  --base-token {base_token} --table-id {table_id} \
  --json '{...}'

# DSL 查询（回写时用）
lark-cli base +data-query \
  --base-token {base_token} \
  --dsl '{...}'
```

**安全规范：**
- JSON 参数用 `json.dumps()` 构建，禁止字符串拼接
- subprocess 用列表形式，禁止 `shell=True`
- 超时 30 秒
- 脱敏后记录 stderr

### 5.6 飞书多维表字段设计

默认所有店铺共用一张表。唯一键为"店铺名称 + Shopify产品ID"（代码层面去重）。

| 序号 | 字段名 | 类型 | 写入方 | 说明 |
|-----|-------|------|-------|------|
| 1 | 店铺名称 | 单选 | 同步 | |
| 2 | Shopify产品ID | 文本 | 同步 | |
| 3 | 产品状态 | 单选 | 同步 | active/draft/archived/已删除 |
| 4 | 价格 | 数字 | 同步 | |
| 5 | SKU | 文本 | 同步 | |
| 6 | 前端链接 | 链接 | 同步 | |
| 7 | 产品图片 | 文本 | 同步 | |
| 8 | Variant图片 | 文本 | 同步 | |
| 9/10 | 产品标题 / 改写标题 | 文本 | 同步/外部 | |
| 11/12 | 原始描述HTML / 改写描述 | 文本 | 同步/外部 | |
| 13 | 原始描述纯文本 | 文本 | 同步 | |
| 14/15 | 产品类型 / 改写产品类型 | 文本 | 同步/外部 | |
| 16/17 | 产品标签 / 改写标签 | 文本 | 同步/外部 | |
| 18/19 | SEO标题 / 改写SEO标题 | 文本 | 同步/外部 | |
| 20/21 | SEO描述 / 改写SEO描述 | 文本 | 同步/外部 | |
| 22/23 | Handle / 改写Handle | 文本 | 同步/外部 | |
| 24/25 | 图片Alt文本 / 改写图片Alt | 文本 | 同步/外部 | |
| 26/27 | 规格名称 / 改写规格名称 | 文本 | 同步/外部 | |
| 28 | 所属集合 | 文本 | 同步 | 只读 |
| 29 | 合规状态 | 单选 | 同步/外部 | 待检查/合规/已改写 |
| 30 | 回写状态 | 单选 | 同步 | 未回写/已回写/回写失败 |
| 31 | 最后更新时间 | 日期时间 | 同步 | |

---

## 6. 错误处理

| 场景 | 处理方式 | 层级 |
|------|---------|------|
| Shopify API 429 | 指数退避重试（1s → 2s → 4s），最多 3 次 | 中台 |
| Shopify API 401（Token） | 标记 token 失效，跳过 | 中台 |
| Shopify API 401（OAuth） | 清除缓存重新获取，仍 401 则标记"凭证过期" | 中台 |
| 代理连接失败 | 标记代理异常，跳过 | 中台 |
| 前端 JSON 404 | 跳过该产品 | 应用 |
| 前端 JSON 超时 | 重试 2 次 | 应用 |
| lark-cli 失败 | 重试 2 次（2s → 5s），终止该店铺同步 | 应用 |
| 单条产品异常 | try-except，记录失败 ID，继续下一条 | 应用 |
| 回写失败 | 标记"回写失败" | 应用 |
| variants 为空 | 价格/SKU 填空 | 应用 |
| 重复记录 | 已存在则改为 update | 应用 |

## 7. 部署（macOS / Mac mini）

### 7.1 环境要求

- Mac mini（本地部署）
- Python 3.8+
- lark-cli 已安装并登录
- 每个店铺一个独立代理 IP
- 如需公网访问：Nginx/Caddy + TLS

### 7.2 部署步骤

```bash
# 1. 拉取代码
git clone <repo-url> ~/shopifyproduct
cd ~/shopifyproduct

# 2. 安装依赖
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 3. 创建 .env
FERNET_KEY=$(python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())")
ADMIN_PASS_HASH=$(python -c "import bcrypt; print(bcrypt.hashpw(b'你的强密码', bcrypt.gensalt()).decode())")

cat > ~/shopifyproduct/.env << EOF
FERNET_KEY=${FERNET_KEY}
ADMIN_USER=admin
ADMIN_PASS_HASH=${ADMIN_PASS_HASH}
SECRET_KEY=$(python -c "import secrets; print(secrets.token_hex(32))")
EOF

chmod 600 ~/shopifyproduct/.env

# ⚠️ 备份 FERNET_KEY！

# 4. 数据库目录
mkdir -p ~/shopifyproduct/data
chmod 700 ~/shopifyproduct/data

# 5. launchd 服务
cp deploy/com.shopifyproduct.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.shopifyproduct.plist
```

### 7.3 launchd 服务

通过 `scripts/start.sh` 启动（加载 .env + gunicorn）：

```bash
#!/bin/bash
set -a
source /Users/adao/shopifyproduct/.env
set +a
cd /Users/adao/shopifyproduct
exec /Users/adao/shopifyproduct/venv/bin/gunicorn \
    -w 1 --threads 4 -b 0.0.0.0:8080 --timeout 120 \
    "scripts.main:create_app()"
```

plist 配置：RunAtLoad + KeepAlive + ThrottleInterval=5。

### 7.4 环境变量（deploy/.env.example）

```bash
FERNET_KEY=
ADMIN_USER=admin
ADMIN_PASS_HASH=
SECRET_KEY=
```

### 7.5 Nginx（可选，公网访问时需要）

gunicorn 绑定 `0.0.0.0:8080`，局域网直接访问。公网需 Nginx + TLS。

### 7.6 健康检查

```
GET /healthz → 200（健康）/ 503（异常）
```

检查：SQLite 可读写、APScheduler 运行中、各店铺最后同步时间正常。

### 7.7 Mac mini 注意事项

- **防休眠：** 系统设置 → 节能 → 勾选"防止自动进入睡眠"
- **开机自启：** launchd + RunAtLoad
- **网络：** 建议有线连接
- **备份：** FERNET_KEY 必须备份到安全位置

## 8. 依赖列表

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

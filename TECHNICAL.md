# Shopify 代理中台 — 技术文档

## 1. 系统架构

系统分为**中台核心层**和**业务应用层**。应用之间不互相调用，只通过飞书多维表的字段状态协作。

```
┌─────────────────────────────────────────────────────────────────┐
│                      Mac mini（本地部署）                          │
│                                                                  │
│  ┌──────────────────── 业务应用层 ─────────────────────────────┐ │
│  │                                                              │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │ │
│  │  │ sync     │  │compliance│  │writeback │  │ 未来应用    │  │ │
│  │  │ Shopify→ │  │ 飞书→    │  │ 飞书→    │  │            │  │ │
│  │  │ 飞书     │  │ 检查→飞书│  │ Shopify  │  │            │  │ │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └─────┬──────┘  │ │
│  │       │             │             │               │         │ │
│  │  ─ ─ ─│─ ─ ─ ─ ─ ─ │─ ─ 飞书多维表 ─ ─ ─ ─ ─ ─ ─│─ ─ ─   │ │
│  │       │             │             │               │         │ │
│  └───────┼─────────────┼─────────────┼───────────────┼─────────┘ │
│          ▼             ▼             ▼               ▼           │
│  ┌──────────────────── 中台核心层 ──────────────────────────────┐ │
│  │                                                              │ │
│  │  ┌──────────────┐  ┌───────────────┐  ┌───────────────────┐ │ │
│  │  │ Nginx (TLS)  │  │ ShopifyClient │  │ LarkClient        │ │ │
│  │  │   ↕ :443     │  │ ● 认证        │  │ ● lark-cli 封装   │ │ │
│  │  │ gunicorn     │  │ ● 代理路由    │  │ ● 分页 / DSL 查询 │ │ │
│  │  │ 127.0.0.1    │  │ ● 限速+重试   │  │ ● 批量读写        │ │ │
│  │  │  :8080       │  └───────┬───────┘  └────────┬──────────┘ │ │
│  │  └──────┬────────┘         │                    │            │ │
│  │         ▼                  │                    │            │ │
│  │  ┌──────────────┐  ┌──────┴─────────┐          │            │ │
│  │  │ SQLite(WAL)  │  │ Scheduler      │          │            │ │
│  │  │ - stores     │  │ (APScheduler)  │          │            │ │
│  │  │ - app_configs│  │ 热更新支持     │          │            │ │
│  │  │ - app_state  │  └────────────────┘          │            │ │
│  │  │ - sync_logs  │                               │            │ │
│  │  └──────────────┘                               │            │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                    │
│         代理IP-A          代理IP-B          代理IP-C               │
│            │                 │                 │                   │
└────────────┼─────────────────┼─────────────────┼───────────────────┘
             ▼                 ▼                 ▼
        Shopify-A         Shopify-B         Shopify-C
```

## 2. 技术栈

| 组件 | 技术 | 层级 |
|------|------|------|
| Web 后台 | Flask + SQLite | 中台 |
| WSGI 服务器 | gunicorn (gthread)，**仅绑 127.0.0.1:8080** | 中台 |
| 反向代理 | **Nginx + TLS（必须部署）** | 中台 |
| 前端页面 | 原生 HTML + JS | 中台 |
| 定时调度 | APScheduler | 中台 |
| HTTP 请求 | requests + PySocks | 中台 |
| Token 加密 | cryptography (Fernet) | 中台 |
| 密码哈希 | bcrypt | 中台 |
| 安全 | flask-limiter / flask-talisman / flask-wtf | 中台 |
| 进程管理 | launchd (macOS) | 中台 |
| 飞书读写 | lark-cli (subprocess) | 中台 |
| HTML 消毒 | bleach | 应用 |
| HTML 转文本 | html2text | 应用 |

## 3. 项目结构

```
shopifyproduct/
  PRD.md                       # 需求文档
  TECHNICAL.md                 # 技术文档（本文件）
  COMPLIANCE_PROMPT.md         # 合规规则（.gitignore 排除，禁止提交）
  requirements.txt
  .gitignore

  ── 中台核心层 ──

  core/
    __init__.py
    shopify_client.py          # Shopify API 统一客户端
    shopify_auth.py            # 认证（Token / OAuth）
    lark_client.py             # lark-cli 封装（分页、批量、DSL）
    store_manager.py           # 店铺 CRUD（含 table_id fallback 逻辑）
    app_registry.py            # 应用发现与注册
    db.py                      # SQLite 操作（WAL + busy_timeout）
    crypto.py                  # Fernet 加密解密
    validators.py              # 输入校验（含 cron 校验）
    scheduler.py               # APScheduler 管理 + 热更新

  web/
    __init__.py
    routes.py                  # Flask 路由（中台 + 应用入口）
    templates/
      index.html               # 管理后台页面

  ── 业务应用层 ──

  apps/
    __init__.py
    base.py                    # 应用基类 / 接口定义

    sync/                      # 产品同步（Shopify → 飞书）
      __init__.py
      light_sync.py
      deep_sync.py
      models.py

    compliance/                # 合规检查（飞书 → 检查 → 飞书）
      __init__.py
      checker.py               # 9 字段风控检测 + 所属集合告警
      rewriter.py

    writeback/                 # 合规回写（飞书 → Shopify，自动执行）
      __init__.py
      writer.py

  ── 启动与部署 ──

  scripts/
    start.sh
    main.py

  nginx/
    shopifyproduct.conf        # Nginx 配置（必须部署）

  deploy/
    com.shopifyproduct.plist
    .env.example

  logs/
  data/
```

## 4. 应用接口规范（`apps/base.py`）

```python
from abc import ABC, abstractmethod

class BaseApp(ABC):

    @property
    @abstractmethod
    def app_name(self) -> str: ...

    @property
    @abstractmethod
    def description(self) -> str: ...

    @property
    def config_schema(self) -> dict:
        """配置字段及默认值，存入 app_configs 表"""
        return {}

    def validate_config(self, config: dict) -> list[str]:
        """校验配置合法性，返回错误列表（空 = 合法）
        必须校验 cron 格式、数值范围等，防止注入"""
        return []

    def get_scheduled_tasks(self, store, config) -> list:
        """返回 [{"func": callable, "trigger": CronTrigger, "id_suffix": str}]"""
        return []

    def on_config_changed(self, store, old_config, new_config):
        """配置变更时调用，用于热更新 scheduler 任务
        中台在 PUT /api/stores/<id>/apps/<app> 后自动调用"""
        pass

    def on_store_deleted(self, store_id):
        """店铺删除时的清理钩子"""
        pass

    def get_routes(self):
        """返回 Flask Blueprint（可选）"""
        return None
```

**sync 应用注册示例：**

```python
class SyncApp(BaseApp):
    app_name = "sync"
    description = "产品数据同步（Shopify → 飞书）"
    config_schema = {
        "light_cron": "0 3 * * *",
        "deep_cycle": 3,
        "deep_hour": 4,
    }

    def validate_config(self, config):
        errors = []
        if not validate_cron(config.get("light_cron", "")):
            errors.append("light_cron 格式不合法")
        dc = config.get("deep_cycle")
        if not isinstance(dc, int) or dc < 1 or dc > 7:
            errors.append("deep_cycle 必须为 1-7")
        if not validate_hour(config.get("deep_hour")):
            errors.append("deep_hour 必须为 0-23")
        return errors

    def on_config_changed(self, store, old_config, new_config):
        """配置变更 → 重新注册调度任务（立即生效）"""
        scheduler.unregister_app_tasks(store.id, self.app_name)
        tasks = self.get_scheduled_tasks(store, new_config)
        scheduler.register_app_tasks(store, self.app_name, tasks)
```

**compliance 应用：**

```python
class ComplianceApp(BaseApp):
    app_name = "compliance"
    description = "合规检查与自动改写（飞书 → 检查 → 飞书）"
    config_schema = {"check_cron": "0 * * * *"}

    def validate_config(self, config):
        if not validate_cron(config.get("check_cron", "")):
            return ["check_cron 格式不合法"]
        return []
```

**writeback 应用：**

```python
class WritebackApp(BaseApp):
    app_name = "writeback"
    description = "合规回写（飞书 → Shopify，自动执行）"
    config_schema = {"writeback_cron": "0 * * * *"}

    def validate_config(self, config):
        if not validate_cron(config.get("writeback_cron", "")):
            return ["writeback_cron 格式不合法"]
        return []
```

### 4.1 应用发现

```python
# core/app_registry.py
def discover_apps() -> dict[str, BaseApp]:
    """扫描 apps/ 目录，自动发现 BaseApp 子类"""
    apps = {}
    package = importlib.import_module("apps")
    for _, module_name, is_pkg in pkgutil.iter_modules(package.__path__):
        if not is_pkg: continue
        module = importlib.import_module(f"apps.{module_name}")
        for attr in dir(module):
            cls = getattr(module, attr)
            if isinstance(cls, type) and issubclass(cls, BaseApp) and cls is not BaseApp:
                instance = cls()
                apps[instance.app_name] = instance
    return apps
```

### 4.2 新增应用步骤

1. 在 `apps/` 下创建新目录（如 `apps/product_copy/`）
2. 实现 `BaseApp` 子类，声明 `app_name`、`config_schema`、`validate_config`
3. 实现 `get_scheduled_tasks()` 返回定时任务（可选）
4. 实现 `get_routes()` 返回 Flask Blueprint（可选）
5. 重启服务 → 中台自动发现并加载

**无需修改 `core/`、`web/` 或其他应用的任何代码。**

### 4.3 应用间解耦规则

- 每个应用是独立目录，有自己的代码和配置
- 应用之间**不互相 import**，只通过飞书多维表的字段状态交换数据
- 应用只依赖中台提供的能力（`ShopifyClient`、`LarkClient`、`TaskScheduler`）
- 并发保护：所有应用共享 store 级文件锁（见 5.6 节）

---

## 5. 中台核心层

### 5.1 ShopifyClient

```python
class ShopifyClient:
    """统一客户端，自动处理认证、代理、限速"""

    def __init__(self, store):
        self.store = store
        self._proxies = {"http": store.proxy, "https": store.proxy}

    # 前端（不需要 Token）
    def list_products(self) -> list: ...
    def get_product(self, handle) -> dict: ...
    def get_product_seo(self, handle) -> dict: ...

    # Admin API（需要 Token）
    def get_product_admin(self, product_id) -> dict: ...
    def get_product_collections(self, product_id) -> list: ...
    def update_product(self, product_id, data) -> bool: ...
    def update_product_seo(self, product_id, title=None, description=None) -> bool: ...

    # 内部
    def _request_frontend(self, path) -> requests.Response: ...
    def _request_admin(self, method, path, json=None) -> requests.Response: ...
    def _get_auth_headers(self) -> dict: ...

def get_client(store) -> ShopifyClient:
    return ShopifyClient(store)
```

### 5.2 LarkClient

```python
class LarkClient:
    def __init__(self, base_token):
        self.base_token = base_token

    def list_records(self, table_id, jq_filter=None) -> list: ...
    def batch_create(self, table_id, records) -> list: ...
    def batch_update(self, table_id, updates) -> bool: ...
    def query(self, table_id, dsl) -> list: ...

    def _run(self, args: list) -> dict:
        """lark-cli 调用，列表形式，禁止 shell=True，超时 30s"""
        ...
```

**安全规范：**
- JSON 用 `json.dumps()` 构建，禁止拼接
- subprocess 列表形式，禁止 `shell=True`
- **从飞书读回的数据（改写字段）写入 Shopify 前必须经过 bleach.clean() 消毒 + 长度/格式校验**

### 5.3 认证

```python
def get_oauth_token(store):
    if store.oauth_token and store.oauth_expires:
        expires = datetime.fromisoformat(store.oauth_expires)
        if datetime.utcnow() < expires - timedelta(minutes=5):
            return decrypt(store.oauth_token)

    resp = requests.post(
        f"https://{store.shop}.myshopify.com/admin/oauth/access_token",
        json={
            "client_id": store.client_id,
            "client_secret": decrypt(store.client_secret),
            "grant_type": "client_credentials"
        },
        proxies=get_proxies(store), timeout=10
    )
    resp.raise_for_status()
    data = resp.json()

    # 从响应读取过期时间，而非硬编码
    expires_in = data.get("expires_in", 86400)  # 默认 24h
    update_store_oauth(store.id,
        oauth_token=encrypt(data["access_token"]),
        oauth_expires=(datetime.utcnow() + timedelta(seconds=expires_in)).isoformat()
    )
    return data["access_token"]
```

| 凭证 | 有效期 | 处理 |
|------|-------|------|
| Access Token | 永久 | 直接用 |
| OAuth Token | 从响应 `expires_in` 读取 | 自动续期 |
| Client ID/Secret | 永久 | 加密存储 |

401 处理：OAuth 收到 401 → 清缓存重新获取 → 仍 401 → 标记"凭证过期"。

**OAuth 刷新锁：** 使用独立文件锁 `data/locks/oauth_{store_id}.lock`（per-store 粒度），防止同步和回写同时刷新竞态。

### 5.4 数据库设计（SQLite WAL + busy_timeout）

初始化：
```sql
PRAGMA journal_mode=WAL;
PRAGMA busy_timeout=5000;  -- 写入冲突时等待最多 5 秒
```

#### stores 表

```sql
CREATE TABLE stores (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    store_name     TEXT NOT NULL UNIQUE,
    shop           TEXT NOT NULL,
    storefront     TEXT NOT NULL,
    auth_mode      TEXT NOT NULL DEFAULT 'token',
    api_token      TEXT,                          -- Fernet 加密
    client_id      TEXT,
    client_secret  TEXT,                          -- Fernet 加密
    oauth_token    TEXT,                          -- Fernet 加密
    oauth_expires  TEXT,
    proxy          TEXT NOT NULL,
    table_id       TEXT,                          -- 可选覆盖，NULL 则 fallback 到全局
    enabled        INTEGER NOT NULL DEFAULT 1,
    created_at     TEXT DEFAULT CURRENT_TIMESTAMP,
    updated_at     TEXT DEFAULT CURRENT_TIMESTAMP
);
```

> **table_id 策略：** 默认为 NULL，使用 settings 表中的全局 `default_table_id`。如果某店铺需要单独的表，可在此字段覆盖。代码层面：`store.table_id or get_setting("default_table_id")`。

#### app_configs 表

```sql
CREATE TABLE app_configs (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    store_id   INTEGER NOT NULL,
    app_name   TEXT NOT NULL,
    config     TEXT NOT NULL DEFAULT '{}',
    enabled    INTEGER NOT NULL DEFAULT 1,
    FOREIGN KEY (store_id) REFERENCES stores(id),
    UNIQUE(store_id, app_name)
);
```

#### app_state 表（应用运行时状态，与 config 分离）

```sql
CREATE TABLE app_state (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    store_id   INTEGER NOT NULL,
    app_name   TEXT NOT NULL,
    state_key  TEXT NOT NULL,               -- 如 'deep_offset'
    state_val  TEXT NOT NULL DEFAULT '0',
    updated_at TEXT DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (store_id) REFERENCES stores(id),
    UNIQUE(store_id, app_name, state_key)
);
```

> **设计说明：** `deep_offset` 等运行时游标从 `app_configs.config` JSON 中分离出来，放在 `app_state` 表。避免 config 编辑与运行时状态互相覆盖的竞态问题。

#### settings 表

```sql
CREATE TABLE settings (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);
-- default_table_id: 全局飞书 Table ID
-- lark_base_token: Fernet 加密
```

#### sync_logs 表

```sql
CREATE TABLE sync_logs (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    store_id   INTEGER NOT NULL,
    app_name   TEXT NOT NULL,
    action     TEXT NOT NULL,
    status     TEXT NOT NULL,
    summary    TEXT,
    detail     TEXT,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (store_id) REFERENCES stores(id)
);
```

#### SQL 安全规范

```python
# 正确：参数化查询
cursor.execute("SELECT * FROM stores WHERE store_name = ?", (name,))
# 错误：禁止
cursor.execute(f"SELECT * FROM stores WHERE store_name = '{name}'")
```

### 5.5 输入校验（`core/validators.py`）

```python
import re, socket, ipaddress

STORE_NAME_RE  = re.compile(r'^[a-zA-Z0-9_-]+$')
SHOP_RE        = re.compile(r'^[a-z0-9-]+$')
PROXY_RE       = re.compile(r'^(socks5|https?)://[a-zA-Z0-9._-]+:\d+$')
STOREFRONT_RE  = re.compile(r'^[a-zA-Z0-9]([a-zA-Z0-9.-]*[a-zA-Z0-9])?$')
HANDLE_RE      = re.compile(r'^[a-z0-9-]+$')
PRODUCT_ID_RE  = re.compile(r'^\d+$')
CRON_FIELD_RE  = re.compile(r'^(\*|\d+(-\d+)?(,\d+(-\d+)?)*)(/\d+)?$')

def validate_cron(expr: str) -> bool:
    """校验 5 字段 cron 表达式，防止注入"""
    parts = expr.strip().split()
    if len(parts) != 5:
        return False
    ranges = [(0,59), (0,23), (1,31), (1,12), (0,7)]
    for part, (lo, hi) in zip(parts, ranges):
        if not CRON_FIELD_RE.match(part):
            return False
        # 检查数值范围
        for num in re.findall(r'\d+', part):
            if not (lo <= int(num) <= hi):
                return False
    return True

def _is_private_ip(host):
    try:
        addr = ipaddress.ip_address(host)
        return addr.is_private or addr.is_loopback or addr.is_reserved or addr.is_link_local
    except ValueError: pass
    try:
        for _, _, _, _, sockaddr in socket.getaddrinfo(host, None):
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
    return not _is_private_ip(proxy.split('://')[1].rsplit(':', 1)[0])
def validate_storefront(domain):
    if not STOREFRONT_RE.match(domain) or len(domain) > 253: return False
    try: ipaddress.ip_address(domain); return False
    except ValueError: pass
    return not _is_private_ip(domain)
def validate_product_id(pid):  return bool(PRODUCT_ID_RE.match(pid)) and len(pid) <= 20
def validate_handle(handle):   return bool(HANDLE_RE.match(handle)) and len(handle) <= 255
def validate_hour(hour):       return isinstance(hour, int) and 0 <= hour <= 23
```

### 5.6 调度器（热更新）

```python
class TaskScheduler:
    def __init__(self):
        self.scheduler = BackgroundScheduler()

    def register_app_tasks(self, store, app_name, tasks):
        for task in tasks:
            job_id = f"{app_name}_{store.id}_{task['id_suffix']}"
            self.scheduler.add_job(
                task["func"], task["trigger"],
                args=[store.id], id=job_id, replace_existing=True
            )

    def unregister_app_tasks(self, store_id, app_name):
        for job in self.scheduler.get_jobs():
            if job.id.startswith(f"{app_name}_{store_id}_"):
                job.remove()

    def start(self):
        self.scheduler.start()
```

**热更新流程：** `PUT /api/stores/<id>/apps/<app>` → `validate_config()` → 保存 → `app.on_config_changed()` → unregister + register → **立即生效**。

**并发保护：** 文件锁 `fcntl.flock`，锁粒度为 **store_id 级别**（`data/locks/store_{store_id}.lock`），**所有应用共享同一把锁**，防止不同应用同时写同一行飞书记录。手动触发用 `LOCK_NB`。锁文件放在项目 `data/locks/` 目录下（`chmod 700`），不使用 `/tmp`（防止其他进程抢占）。

### 5.7 Web 管理后台

#### API 路由

```
── 中台路由 ──
GET    /                            → 管理后台页面
GET    /login                       → 登录页
POST   /login                       → 登录（限速 5次/15分钟）
GET    /healthz                     → 健康检查（未认证: 仅 {"status":"ok"}）
GET    /healthz?detail=1            → 详细健康检查（需认证）
GET    /api/stores                  → 所有店铺（限速 30次/分钟）
POST   /api/stores                  → 新增店铺（限速 10次/分钟）
PUT    /api/stores/<id>             → 更新店铺（限速 10次/分钟）
DELETE /api/stores/<id>             → 删除店铺（限速 5次/分钟）
POST   /api/stores/<id>/test       → 测试连接（限速 5次/分钟）
POST   /api/stores/<id>/token      → 明文 token（限速 5次/分钟，需二次密码确认，POST 提交密码）
GET    /api/stores/<id>/logs       → 日志（可按 app_name 筛选）
GET    /api/settings                → 全局设置（限速 30次/分钟）
PUT    /api/settings                → 更新全局设置（限速 3次/分钟，需二次密码确认）

── 应用配置路由 ──
GET    /api/apps                    → 已发现的所有应用列表
GET    /api/stores/<id>/apps        → 该店铺已启用的应用及配置
PUT    /api/stores/<id>/apps/<app>  → 更新应用配置（限速 10次/分钟，校验后热更新调度）

── 应用专属路由（手动触发，限速 3次/分钟/店铺） ──
POST   /api/stores/<id>/sync       → [sync] 手动触发同步
POST   /api/stores/<id>/check      → [compliance] 手动触发检查
POST   /api/stores/<id>/writeback  → [writeback] 手动触发回写
```

#### 认证与安全

- **Nginx 必须部署**：gunicorn 仅绑 `127.0.0.1:8080`，外部通过 Nginx :443 访问
- Session 登录 + bcrypt + CSRF + flask-talisman
- 超时 30 分钟
- 登录限速 5次/15分钟
- **Token 查看需二次密码确认**（POST 提交密码 → 返回 token）
- **全局设置修改需二次密码确认**（防止 table_id / lark_base_token 被篡改）
- **所有端点均有限速**（CRUD、手动触发、设置修改）
- 凭证 Fernet 加密，密钥从 `FERNET_KEY` 环境变量读取
- `/healthz` 未认证时只返回 `{"status":"ok"}`，认证后返回详细信息（调度器状态、各应用最后运行时间、代理连通性）
- **CORS 锁死**：无 `Access-Control-Allow-Origin` 头

### 5.8 Shopify API 对接

#### 接口列表

| 端点 | 用途 | Token |
|------|------|-------|
| `GET {storefront}/products.json` | 产品列表 | 否 |
| `GET {storefront}/products/{handle}.json` | 产品详情 | 否 |
| `GET {storefront}/products/{handle}` | HTML → SEO | 否 |
| `GET /admin/api/{version}/products/{id}.json` | 完整数据 | 是 |
| `GET /admin/api/{version}/collects.json` | 所属集合 | 是 |
| `PUT /admin/api/{version}/products/{id}.json` | 更新产品 | 是 |
| `PUT /admin/api/{version}/products/{id}/metafields.json` | 更新 SEO | 是 |

#### API Version

默认 `2025-04`，存为 settings 表中的全局配置项（可在管理后台修改，无需改代码）。启动时检查距发布超过 10 个月则 WARNING。

#### 限速

- Admin API：2 req/s，监控 `X-Shopify-Shop-Api-Call-Limit`
- 额度 < 5 → sleep 1s
- 429 → 指数退避（1s → 2s → 4s），最多 3 次
- 前端：1-2s 间隔 + 0-0.5s 随机抖动

---

## 6. 业务应用层

### 6.1 产品同步（`apps/sync/`）

**依赖：** `core.shopify_client` + `core.lark_client`
**不依赖：** compliance / writeback

#### 轻量扫描

```
light_sync(store_id)
  → 获取 store 级文件锁
  → client.list_products()
  → lark.list_records(table_id, filter=store_name)
  → 对比（基于字段值对比，不用 updated_at）：
      新产品 → 写入飞书，合规状态="待检查"
      字段变化 → 更新飞书，合规状态="待检查"，清空所有改写字段
      已删除 → 产品状态="已删除"
  → 释放锁
```

**关键规则：** 重置"待检查"时**必须同时清空所有改写列**（改写标题、改写描述等），防止旧改写数据与新内容不匹配。

#### 深度扫描

```
deep_sync(store_id)
  → 获取 store 级文件锁
  → 从飞书读取所有产品（按 ID 排序）
  → 从 app_state 读取 deep_offset
  → batch_size = ceil(total / deep_cycle)
  → 首次全量，后续分批
  → 对每个产品：
      client.get_product_seo(handle) → SEO 变化
      client.get_product(handle) → 图片 alt / variant 变化
  → 更新 app_state 中的 deep_offset
  → 释放锁
```

### 6.2 合规检查（`apps/compliance/`）

**依赖：** `core.lark_client`
**不依赖：** shopify_client / sync / writeback

```
compliance_check(store_id)
  → 获取 store 级文件锁
  → lark.query(dsl={合规状态=="待检查" AND 产品状态≠"已删除"})
  → 对每条记录读取 9 个可扫描字段
  → 额外检查"所属集合"，命中风险 → 写 sync_logs 告警，不自动改写
  → 全部合规 → batch_update(合规状态="合规")
  → 需改写 → rewriter 生成改写内容
           → batch_update(改写字段 + 合规状态="已改写", 回写状态="未回写")
  → 释放锁
```

### 6.3 合规回写（`apps/writeback/`）

**依赖：** `core.shopify_client` + `core.lark_client`
**不依赖：** sync / compliance

```
writeback(store_id)
  → 获取 store 级文件锁
  → lark.query(dsl={合规状态=="已改写" AND 回写状态≠"已回写" AND 产品状态≠"已删除"})
  → 对每条记录：
      validate_product_id()
      所有改写字段：长度限制 + 格式校验（Handle 须 [a-z0-9-]，标签须逗号分隔等）
      改写描述 → bleach.clean(ALLOWED_TAGS) — HTML 消毒
      改写标题/类型/标签/SEO/Handle/Alt/规格 → 纯文本，strip HTML 标签 + 长度截断
      client.update_product(id, data)
      client.update_product_seo(id, title, desc)
      成功 → batch_update(回写状态="已回写", 合规状态="合规", 原始字段同步更新)
      失败 → batch_update(回写状态="回写失败")
  → 释放锁
```

**全自动执行：** compliance 设置 `合规状态="已改写"` 后，writeback 下次运行时自动回写，无需人工干预。如需重新回写（如修改了改写内容），将"回写状态"改回"未回写"即可。

### 6.4 应用依赖关系

```
              core.shopify_client    core.lark_client
                     │                      │
       ┌─────────────┼──────────────────────┼──────────┐
       │             │                      │          │
       ▼             ▼                      ▼          ▼
  ┌─────────┐   ┌─────────┐          ┌───────────┐
  │  sync   │   │writeback│          │compliance │
  │ 用 both │   │ 用 both │          │ 只用 lark │
  └────┬────┘   └────┬────┘          └─────┬─────┘
       │             │                     │
       └─────── 飞书多维表（数据总线） ──────┘
                字段状态驱动协作
                store 级文件锁保护
                无代码级依赖
```

### 6.5 数据映射（sync）

| API 字段 | 多维表字段 |
|----------|-----------|
| `product.id` | Shopify产品ID |
| `product.title` | 产品标题 |
| `product.body_html` | 原始描述HTML |
| html2text(body_html) | 原始描述纯文本 |
| `product.product_type` | 产品类型 |
| `product.tags` | 产品标签 |
| `product.handle` | Handle |
| `product.status` | 产品状态 |
| `product.variants[*].price` | 价格（每行：`option1 \| price`） |
| `product.variants[*].sku` | SKU（每行：`option1 \| sku`） |
| `product.images[*].src` | 产品图片 |
| `product.images[*].alt` | 图片Alt文本 |
| `product.variants[*].featured_image.src` | Variant图片 |
| `product.variants[*].option1` | 规格名称 |
| HTML `<title>` | SEO标题 |
| HTML `<meta description>` | SEO描述 |
| collects → collection.title | 所属集合 |

### 6.6 飞书多维表字段

> 字段的业务含义见 PRD.md 第 5.2 节。以下为技术实现规格。

| # | 字段名 | 类型 | 写入方 |
|---|-------|------|-------|
| 1 | 店铺名称 | 单选 | sync |
| 2 | Shopify产品ID | 文本 | sync |
| 3 | 产品状态 | 单选 | sync（含"已删除"） |
| 4 | 价格 | 文本 | sync（每行：`规格名 \| 价格`） |
| 5 | SKU | 文本 | sync（每行：`规格名 \| SKU`） |
| 6 | 前端链接 | 链接 | sync |
| 7 | 产品图片 | 文本 | sync |
| 8 | Variant图片 | 文本 | sync |
| 9/10 | 产品标题 / 改写标题 | 文本 | sync / compliance |
| 11/12 | 原始描述HTML / 改写描述 | 文本 | sync / compliance |
| 13 | 原始描述纯文本 | 文本 | sync |
| 14/15 | 产品类型 / 改写产品类型 | 文本 | sync / compliance |
| 16/17 | 产品标签 / 改写标签 | 文本 | sync / compliance |
| 18/19 | SEO标题 / 改写SEO标题 | 文本 | sync / compliance |
| 20/21 | SEO描述 / 改写SEO描述 | 文本 | sync / compliance |
| 22/23 | Handle / 改写Handle | 文本 | sync / compliance |
| 24/25 | 图片Alt文本 / 改写图片Alt | 文本 | sync / compliance |
| 26/27 | 规格名称 / 改写规格名称 | 文本 | sync / compliance |
| 28 | 所属集合 | 文本 | sync（只读，风险时日志告警） |
| 29 | 合规状态 | 单选 | sync / compliance / writeback |
| 30 | 回写状态 | 单选 | writeback |
| 31 | 最后更新时间 | 日期时间 | 各应用 |

**回写状态三态：** 未回写 → 已回写 / 回写失败（如需重新回写，改回"未回写"即可）

---

## 7. 错误处理

| 场景 | 处理 | 层级 |
|------|------|------|
| Shopify 429 | 指数退避（1→2→4s），最多 3 次 | 中台 |
| Shopify 401（Token） | 标记失效 | 中台 |
| Shopify 401（OAuth） | 清缓存（per-store 锁）重获，仍 401 标记过期 | 中台 |
| 代理连接失败 | 标记异常，跳过 | 中台 |
| lark-cli 失败 | 重试 2 次（2s → 5s） | 中台 |
| 前端 404 | 跳过该产品 | sync |
| 前端超时 | 重试 2 次 | sync |
| 单条产品异常 | catch，记录，继续 | sync |
| 合规检查异常 | catch，记录，继续 | compliance |
| 回写失败 | 标记"回写失败" | writeback |

## 8. 部署

### 8.1 环境要求

- Mac mini + Python 3.8+ + lark-cli
- 每店独立代理
- **Nginx + TLS 必须部署**（gunicorn 仅绑 127.0.0.1）

### 8.2 部署步骤

```bash
git clone <repo> ~/shopifyproduct && cd ~/shopifyproduct
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# 生成 .env（仅首次，后续不覆盖）
FERNET_KEY=$(python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())")
ADMIN_PASS_HASH=$(python -c "import bcrypt; print(bcrypt.hashpw(b'你的密码', bcrypt.gensalt()).decode())")
cat > .env << EOF
FERNET_KEY=${FERNET_KEY}
ADMIN_USER=admin
ADMIN_PASS_HASH=${ADMIN_PASS_HASH}
SECRET_KEY=$(python -c "import secrets; print(secrets.token_hex(32))")
EOF
chmod 600 .env

# ⚠️ .env 仅生成一次，后续重启/升级不覆盖，否则所有 session 和加密数据失效
# ⚠️ FERNET_KEY 必须备份到独立安全位置（如密码管理器），丢失不可恢复

mkdir -p data && chmod 700 data

# 部署 Nginx（必须）
sudo cp nginx/shopifyproduct.conf /etc/nginx/servers/
sudo nginx -t && sudo nginx -s reload

# launchd 服务
cp deploy/com.shopifyproduct.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.shopifyproduct.plist
```

### 8.3 启动脚本

```bash
#!/bin/bash
# scripts/start.sh — gunicorn 仅绑 127.0.0.1
set -a && source /Users/adao/shopifyproduct/.env && set +a
cd /Users/adao/shopifyproduct
exec venv/bin/gunicorn -w 1 --threads 4 -b 127.0.0.1:8080 --timeout 120 \
    "scripts.main:create_app()"
```

### 8.4 Nginx 配置（必须）

```nginx
server {
    listen 443 ssl http2;
    server_name your-domain.com;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    add_header Strict-Transport-Security "max-age=63072000" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Content-Security-Policy "default-src 'self'" always;
    proxy_hide_header X-Powered-By;

    # 不设置 Access-Control-Allow-Origin

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
    server_name your-domain.com;
    return 301 https://$host$request_uri;
}
```

### 8.5 Mac mini 注意事项

- 防休眠：系统设置 → 节能 → 勾选"防止自动进入睡眠"
- 网络：建议有线
- 备份：FERNET_KEY 备份到独立安全位置（非 Time Machine，非同目录）
- .env 保护：数据库文件和 .env 不要备份到同一位置

### 8.6 健康检查

```
GET /healthz                → {"status": "ok"}（无需认证）
GET /healthz?detail=1       → 详细状态（需认证）：
    - SQLite 可读写
    - APScheduler 运行中
    - 各店铺各应用最后成功时间
    - 代理连通性
    - lark-cli 可用性
```

## 9. 依赖

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

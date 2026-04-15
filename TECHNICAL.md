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
│  │  │ Web 管理后台  │  │ ShopifyClient │  │ LarkClient        │ │ │
│  │  │ (gunicorn)    │  │ ● 认证        │  │ ● lark-cli 封装   │ │ │
│  │  │ - 店铺 CRUD   │  │ ● 代理路由    │  │ ● 分页 / DSL 查询 │ │ │
│  │  │ - 应用管理    │  │ ● 限速+重试   │  │ ● 批量读写        │ │ │
│  │  └──────┬────────┘  └───────┬───────┘  └────────┬──────────┘ │ │
│  │         ▼                   │                    │            │ │
│  │  ┌──────────────┐  ┌───────┴────────┐           │            │ │
│  │  │ SQLite(WAL)  │  │ Scheduler      │           │            │ │
│  │  │ - stores     │  │ (APScheduler)  │           │            │ │
│  │  │ - app_configs│  │ 应用注册/注销   │           │            │ │
│  │  │ - sync_logs  │  └────────────────┘           │            │ │
│  │  └──────────────┘                                │            │ │
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
| WSGI 服务器 | gunicorn (gthread) | 中台 |
| 反向代理 | Nginx + Let's Encrypt | 中台（可选） |
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
  COMPLIANCE_PROMPT.md         # 合规规则（.gitignore 排除）
  requirements.txt
  .gitignore

  ── 中台核心层 ──

  core/
    __init__.py
    shopify_client.py          # Shopify API 统一客户端
    shopify_auth.py            # 认证（Token / OAuth）
    lark_client.py             # lark-cli 封装（分页、批量、DSL）
    store_manager.py           # 店铺 CRUD
    app_registry.py            # 应用发现与注册
    db.py                      # SQLite 操作
    crypto.py                  # Fernet 加密解密
    validators.py              # 输入校验
    scheduler.py               # APScheduler 管理

  web/
    __init__.py
    routes.py                  # Flask 路由（中台 + 应用入口）
    templates/
      index.html               # 管理后台页面

  ── 业务应用层（每个应用一个独立目录） ──

  apps/
    __init__.py
    base.py                    # 应用基类 / 接口定义

    sync/                      # 应用1：产品同步（Shopify → 飞书）
      __init__.py              #   声明 app_name, config_schema, tasks
      light_sync.py            #   轻量扫描
      deep_sync.py             #   深度扫描
      models.py                #   数据结构

    compliance/                # 应用2：合规检查（飞书 → 检查 → 飞书）
      __init__.py              #   声明 app_name, config_schema, tasks
      checker.py               #   风控规则检测
      rewriter.py              #   自动改写

    writeback/                 # 应用3：合规回写（飞书 → Shopify）
      __init__.py              #   声明 app_name, config_schema, tasks
      writer.py                #   回写逻辑

    # ── 未来应用 ──
    # product_copy/            # A站 → B站
    # report_export/           # 多店报表

  ── 启动与部署 ──

  scripts/
    start.sh
    main.py                    # Flask + Scheduler 入口

  deploy/
    com.shopifyproduct.plist
    .env.example

  logs/
  data/
```

## 4. 应用接口规范（`apps/base.py`）

每个应用必须实现以下接口，中台通过此接口自动发现和管理应用：

```python
# apps/base.py
from abc import ABC, abstractmethod

class BaseApp(ABC):
    """所有业务应用的基类"""

    @property
    @abstractmethod
    def app_name(self) -> str:
        """唯一标识，如 'sync', 'compliance', 'writeback'"""
        ...

    @property
    @abstractmethod
    def description(self) -> str:
        """应用描述，显示在管理后台"""
        ...

    @property
    def config_schema(self) -> dict:
        """应用需要的配置字段及默认值，存入 app_configs 表
        返回 {} 表示不需要额外配置"""
        return {}

    def get_scheduled_tasks(self, store, config) -> list:
        """返回该店铺需要注册的定时任务
        返回 [] 表示不需要定时任务
        每个 task: {"func": callable, "trigger": CronTrigger, "id_suffix": str}
        """
        return []

    def get_routes(self):
        """返回应用专属的 Flask Blueprint（可选）
        返回 None 表示不需要额外路由"""
        return None
```

**应用注册示例（`apps/sync/__init__.py`）：**

```python
from apps.base import BaseApp
from apscheduler.triggers.cron import CronTrigger

class SyncApp(BaseApp):
    app_name = "sync"
    description = "产品数据同步（Shopify → 飞书）"

    config_schema = {
        "light_cron": "0 3 * * *",   # 轻量扫描 cron
        "deep_cycle": 3,              # 深度扫描周期（天）
        "deep_hour": 4,               # 深度扫描时间
        "deep_offset": 0,             # 内部：当前偏移量
    }

    def get_scheduled_tasks(self, store, config):
        from .light_sync import light_sync
        from .deep_sync import deep_sync
        return [
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
        ]
```

**合规检查应用（`apps/compliance/__init__.py`）：**

```python
class ComplianceApp(BaseApp):
    app_name = "compliance"
    description = "合规检查与自动改写（飞书 → 检查 → 飞书）"

    config_schema = {
        "check_cron": "0 * * * *",    # 每小时检查一次
    }

    def get_scheduled_tasks(self, store, config):
        from .checker import compliance_check
        return [
            {
                "func": compliance_check,
                "trigger": CronTrigger.from_crontab(config["check_cron"]),
                "id_suffix": "check"
            }
        ]
```

**回写应用（`apps/writeback/__init__.py`）：**

```python
class WritebackApp(BaseApp):
    app_name = "writeback"
    description = "合规回写（飞书 → Shopify）"

    config_schema = {
        "writeback_cron": "0 * * * *",  # 每小时回写一次
    }

    def get_scheduled_tasks(self, store, config):
        from .writer import writeback
        return [
            {
                "func": writeback,
                "trigger": CronTrigger.from_crontab(config["writeback_cron"]),
                "id_suffix": "wb"
            }
        ]
```

### 4.1 应用发现（`core/app_registry.py`）

中台启动时自动扫描 `apps/` 下所有实现了 `BaseApp` 的类：

```python
import importlib, pkgutil
from apps.base import BaseApp

def discover_apps() -> dict[str, BaseApp]:
    """扫描 apps/ 目录，返回 {app_name: app_instance}"""
    apps = {}
    package = importlib.import_module("apps")
    for _, module_name, is_pkg in pkgutil.iter_modules(package.__path__):
        if not is_pkg:
            continue
        module = importlib.import_module(f"apps.{module_name}")
        for attr in dir(module):
            cls = getattr(module, attr)
            if (isinstance(cls, type) and issubclass(cls, BaseApp)
                    and cls is not BaseApp):
                instance = cls()
                apps[instance.app_name] = instance
    return apps
```

**新增应用不需要修改任何已有代码：** 只需在 `apps/` 下创建新目录 + 实现 `BaseApp` 子类，重启即自动加载。

---

## 5. 中台核心层

### 5.1 ShopifyClient（`core/shopify_client.py`）

```python
class ShopifyClient:
    """统一 Shopify API 客户端，自动处理认证、代理、限速"""

    def __init__(self, store):
        self.store = store
        self._proxies = {"http": store.proxy, "https": store.proxy}

    # ── 前端（不需要 Token） ──
    def list_products(self) -> list: ...
    def get_product(self, handle: str) -> dict: ...
    def get_product_seo(self, handle: str) -> dict: ...

    # ── Admin API（需要 Token） ──
    def get_product_admin(self, product_id: str) -> dict: ...
    def get_product_collections(self, product_id: str) -> list: ...
    def update_product(self, product_id: str, data: dict) -> bool: ...
    def update_product_seo(self, product_id: str, title=None, description=None) -> bool: ...

    # ── 内部 ──
    def _request_frontend(self, path) -> requests.Response: ...
    def _request_admin(self, method, path, json=None) -> requests.Response: ...
    def _get_auth_headers(self) -> dict: ...

def get_client(store) -> ShopifyClient:
    return ShopifyClient(store)
```

### 5.2 LarkClient（`core/lark_client.py`）

```python
class LarkClient:
    """lark-cli 封装，提供统一的飞书多维表操作"""

    def __init__(self, base_token: str):
        self.base_token = base_token

    def list_records(self, table_id, jq_filter=None) -> list:
        """分页读取记录（可选 jq 过滤）"""
        ...

    def batch_create(self, table_id, records: list) -> list:
        """批量创建记录"""
        ...

    def batch_update(self, table_id, updates: list) -> bool:
        """批量更新记录"""
        ...

    def query(self, table_id, dsl: dict) -> list:
        """DSL 条件查询（合规检查、回写时用）"""
        ...

    def _run(self, args: list) -> dict:
        """执行 lark-cli 命令（列表形式，禁止 shell=True，超时 30s）"""
        ...
```

**安全规范：**
- JSON 参数用 `json.dumps()` 构建，禁止字符串拼接
- subprocess 用列表形式，禁止 `shell=True`
- store_name 等用户输入经 `validate_store_name()` 校验后再用 `json.dumps()` 转义拼入 jq

### 5.3 认证（`core/shopify_auth.py`）

```python
def get_oauth_token(store):
    """OAuth token，过期自动重新获取"""
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
    update_store_oauth(store.id,
        oauth_token=encrypt(data["access_token"]),
        oauth_expires=(datetime.utcnow() + timedelta(hours=24)).isoformat()
    )
    return data["access_token"]
```

| 凭证 | 有效期 | 处理 |
|------|-------|------|
| Access Token | 永久 | 直接用 |
| OAuth Token | 24h | 自动续期 |
| Client ID/Secret | 永久 | 加密存储 |

401 处理：OAuth 收到 401 → 清缓存重新获取 → 仍 401 → 标记"凭证过期"。

### 5.4 数据库设计

#### stores 表（中台级别）

```sql
CREATE TABLE stores (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    store_name     TEXT NOT NULL UNIQUE,
    shop           TEXT NOT NULL,
    storefront     TEXT NOT NULL,
    auth_mode      TEXT NOT NULL DEFAULT 'token',
    api_token      TEXT,
    client_id      TEXT,
    client_secret  TEXT,
    oauth_token    TEXT,
    oauth_expires  TEXT,
    proxy          TEXT NOT NULL,
    table_id       TEXT NOT NULL,
    enabled        INTEGER NOT NULL DEFAULT 1,
    created_at     TEXT DEFAULT CURRENT_TIMESTAMP,
    updated_at     TEXT DEFAULT CURRENT_TIMESTAMP
);
```

#### app_configs 表（应用配置，完全解耦）

```sql
CREATE TABLE app_configs (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    store_id   INTEGER NOT NULL,
    app_name   TEXT NOT NULL,
    config     TEXT NOT NULL DEFAULT '{}',  -- JSON
    enabled    INTEGER NOT NULL DEFAULT 1,
    FOREIGN KEY (store_id) REFERENCES stores(id),
    UNIQUE(store_id, app_name)
);

-- sync 应用: {"light_cron":"0 3 * * *","deep_cycle":3,"deep_hour":4,"deep_offset":0}
-- compliance 应用: {"check_cron":"0 * * * *"}
-- writeback 应用: {"writeback_cron":"0 * * * *"}
```

#### settings 表

```sql
CREATE TABLE settings (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);
-- lark_base_token（Fernet 加密）
```

#### sync_logs 表（所有应用共用，按 app_name 区分）

```sql
CREATE TABLE sync_logs (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    store_id   INTEGER NOT NULL,
    app_name   TEXT NOT NULL,       -- 'sync' / 'compliance' / 'writeback' / ...
    action     TEXT NOT NULL,       -- light_sync / deep_sync / check / writeback / ...
    status     TEXT NOT NULL,       -- success / failed
    summary    TEXT,
    detail     TEXT,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (store_id) REFERENCES stores(id)
);
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

### 5.6 调度器（`core/scheduler.py`）

```python
class TaskScheduler:
    def __init__(self):
        self.scheduler = BackgroundScheduler()

    def register_app_tasks(self, store, app_name, tasks):
        for task in tasks:
            job_id = f"{app_name}_{store.id}_{task['id_suffix']}"
            self.scheduler.add_job(
                task["func"], task["trigger"],
                args=[store.id], id=job_id
            )

    def unregister_app_tasks(self, store_id, app_name):
        for job in self.scheduler.get_jobs():
            if job.id.startswith(f"{app_name}_{store_id}_"):
                job.remove()

    def start(self):
        self.scheduler.start()
```

**启动流程：**

```python
# scripts/main.py
def create_app():
    app = create_flask_app()
    scheduler = TaskScheduler()
    registered_apps = discover_apps()  # 自动发现

    with app.app_context():
        for store in get_enabled_stores():
            for app_name, app_instance in registered_apps.items():
                config = get_app_config(store.id, app_name)
                if config and config.enabled:
                    tasks = app_instance.get_scheduled_tasks(store, config.config)
                    scheduler.register_app_tasks(store, app_name, tasks)

        # 注册应用路由
        for app_name, app_instance in registered_apps.items():
            blueprint = app_instance.get_routes()
            if blueprint:
                app.register_blueprint(blueprint)

        scheduler.start()
    return app
```

**并发保护：** 文件锁 `fcntl.flock` 按 `{app_name}_{store_id}` 粒度。OAuth 刷新独立锁。

### 5.7 Web 管理后台

#### API 路由

```
── 中台路由 ──
GET    /                            → 管理后台页面
GET    /login                       → 登录页
POST   /login                       → 登录（限速 5次/15分钟）
GET    /healthz                     → 健康检查
GET    /api/stores                  → 所有店铺
POST   /api/stores                  → 新增店铺
PUT    /api/stores/<id>             → 更新店铺
DELETE /api/stores/<id>             → 删除店铺
POST   /api/stores/<id>/test       → 测试连接
GET    /api/stores/<id>/token      → 明文 token（限速+审计）
GET    /api/stores/<id>/logs       → 日志（可按 app_name 筛选）
GET    /api/settings                → 全局设置
PUT    /api/settings                → 更新全局设置

── 应用配置路由 ──
GET    /api/apps                    → 已发现的所有应用列表
GET    /api/stores/<id>/apps        → 该店铺已启用的应用及配置
PUT    /api/stores/<id>/apps/<app>  → 更新应用配置 / 启停应用

── 应用专属路由（由应用自行注册） ──
POST   /api/stores/<id>/sync       → [sync] 手动触发同步
POST   /api/stores/<id>/check      → [compliance] 手动触发检查
POST   /api/stores/<id>/writeback  → [writeback] 手动触发回写
```

#### 认证与安全

- Session 登录 + bcrypt + CSRF + flask-talisman
- 超时 30 分钟
- 登录限速 5次/15分钟
- 凭证 Fernet 加密，密钥从 `FERNET_KEY` 环境变量读取

### 5.8 Shopify API 对接

#### 接口列表

| 端点 | 用途 | Token |
|------|------|-------|
| `GET {storefront}/products.json` | 产品列表 | 否 |
| `GET {storefront}/products/{handle}.json` | 产品详情 | 否 |
| `GET {storefront}/products/{handle}` | HTML → SEO | 否 |
| `GET /admin/api/2024-10/products/{id}.json` | 完整数据 | 是 |
| `GET /admin/api/2024-10/collects.json` | 所属集合 | 是 |
| `PUT /admin/api/2024-10/products/{id}.json` | 更新产品 | 是 |
| `PUT /admin/api/2024-10/products/{id}/metafields.json` | 更新 SEO | 是 |

#### 限速

- Admin API：2 req/s，监控 `X-Shopify-Shop-Api-Call-Limit`
- 额度 < 5 → sleep 1s
- 429 → 指数退避（1s → 2s → 4s），最多 3 次
- 前端：1-2s 间隔 + 0-0.5s 随机抖动

---

## 6. 业务应用层

### 6.1 产品同步（`apps/sync/`）

**依赖：** `core.shopify_client` + `core.lark_client`
**不依赖：** compliance / writeback / 任何其他应用

#### 轻量扫描

```
light_sync(store_id)
  → get_client(store).list_products()
  → lark.list_records(table_id, filter=store_name)
  → 对比：新产品写入 / 已有更新 / 已删除标记
  → 合规状态 = "待检查"（新产品或字段变化时）
```

#### 深度扫描

```
deep_sync(store_id)
  → 从飞书读取所有产品，按 deep_cycle 计算本批范围
  → 首次全量，后续分批
  → client.get_product_seo(handle) → SEO 变化
  → client.get_product(handle) → 图片 alt / variant 变化
  → 更新 deep_offset
```

### 6.2 合规检查（`apps/compliance/`）

**依赖：** `core.lark_client`
**不依赖：** shopify_client / sync / writeback

```
compliance_check(store_id)
  → lark.query(table_id, dsl={合规状态 == "待检查"})
  → 对每条记录读取 10 个可扫描字段
  → 按 COMPLIANCE_PROMPT.md 规则检测
  → 全部合规 → lark.batch_update(合规状态="合规")
  → 需改写 → rewriter 生成改写内容
           → lark.batch_update(改写字段 + 合规状态="已改写")
```

**检查范围：** 产品标题、描述、产品类型、标签、SEO标题、SEO描述、Handle、图片Alt、规格名称、所属集合。

### 6.3 合规回写（`apps/writeback/`）

**依赖：** `core.shopify_client` + `core.lark_client`
**不依赖：** sync / compliance

```
writeback(store_id)
  → lark.query(table_id, dsl={合规状态=="已改写" AND 回写状态≠"已回写"})
  → 对每条记录：
      → validate_product_id() + 校验改写字段
      → client = get_client(store)
      → client.update_product(id, data)
      → client.update_product_seo(id, title, desc)
      → 成功 → lark.batch_update(回写状态="已回写", 原始字段同步)
      → 失败 → lark.batch_update(回写状态="回写失败")
```

### 6.4 应用依赖关系图

```
                core.shopify_client    core.lark_client
                       │                      │
         ┌─────────────┼──────────────────────┼──────────┐
         │             │                      │          │
         ▼             ▼                      ▼          ▼
    ┌─────────┐   ┌─────────┐          ┌───────────┐
    │  sync   │   │writeback│          │compliance │
    │         │   │         │          │           │
    │ 用 both │   │ 用 both │          │ 只用 lark │
    └────┬────┘   └────┬────┘          └─────┬─────┘
         │             │                     │
         │             │                     │
         └─────── 飞书多维表（数据总线） ──────┘
                  字段状态驱动协作
                  无代码级依赖
```

### 6.5 数据映射（sync 应用）

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
| `product.variants[0].price` | 价格 |
| `product.variants[0].sku` | SKU |
| `product.images[*].src` | 产品图片 |
| `product.images[*].alt` | 图片Alt文本 |
| `product.variants[*].featured_image.src` | Variant图片 |
| `product.variants[*].option1` | 规格名称 |
| HTML `<title>` | SEO标题 |
| HTML `<meta description>` | SEO描述 |
| collects → collection.title | 所属集合 |

### 6.6 飞书多维表字段

| # | 字段名 | 类型 | 写入方 |
|---|-------|------|-------|
| 1 | 店铺名称 | 单选 | sync |
| 2 | Shopify产品ID | 文本 | sync |
| 3 | 产品状态 | 单选 | sync |
| 4 | 价格 | 数字 | sync |
| 5 | SKU | 文本 | sync |
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
| 28 | 所属集合 | 文本 | sync |
| 29 | 合规状态 | 单选 | sync / compliance |
| 30 | 回写状态 | 单选 | writeback |
| 31 | 最后更新时间 | 日期时间 | 各应用 |

---

## 7. 错误处理

| 场景 | 处理 | 层级 |
|------|------|------|
| Shopify 429 | 指数退避（1→2→4s），最多 3 次 | 中台 |
| Shopify 401（Token） | 标记失效 | 中台 |
| Shopify 401（OAuth） | 清缓存重获，仍 401 标记过期 | 中台 |
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
- 公网访问需 Nginx + TLS

### 8.2 部署步骤

```bash
git clone <repo> ~/shopifyproduct && cd ~/shopifyproduct
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# 生成 .env
FERNET_KEY=$(python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())")
ADMIN_PASS_HASH=$(python -c "import bcrypt; print(bcrypt.hashpw(b'你的密码', bcrypt.gensalt()).decode())")
cat > .env << EOF
FERNET_KEY=${FERNET_KEY}
ADMIN_USER=admin
ADMIN_PASS_HASH=${ADMIN_PASS_HASH}
SECRET_KEY=$(python -c "import secrets; print(secrets.token_hex(32))")
EOF
chmod 600 .env

mkdir -p data && chmod 700 data
cp deploy/com.shopifyproduct.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.shopifyproduct.plist
```

### 8.3 启动脚本

```bash
#!/bin/bash
# scripts/start.sh
set -a && source /Users/adao/shopifyproduct/.env && set +a
cd /Users/adao/shopifyproduct
exec venv/bin/gunicorn -w 1 --threads 4 -b 0.0.0.0:8080 --timeout 120 \
    "scripts.main:create_app()"
```

### 8.4 Mac mini 注意事项

- 防休眠：系统设置 → 节能 → 勾选"防止自动进入睡眠"
- 网络：建议有线
- 备份：`FERNET_KEY` 必须备份

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

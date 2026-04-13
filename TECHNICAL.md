# Shopify 产品数据同步系统 — 技术文档

## 1. 系统架构

```
┌─────────────────────────────────────────────────┐
│                  单台 VPS                        │
│                                                  │
│  ┌──────────────┐     ┌───────────────────────┐ │
│  │  Web 管理后台 │     │  定时调度器             │ │
│  │  (Flask)      │     │  (APScheduler)         │ │
│  │  - 店铺配置   │     │  - 按店铺设定频率触发    │ │
│  │  - 状态查看   │     │  - 通过代理IP访问       │ │
│  └──────┬───────┘     └──────────┬────────────┘ │
│         │                        │               │
│         ▼                        ▼               │
│  ┌──────────────┐     ┌───────────────────────┐ │
│  │  SQLite(WAL) │     │  同步引擎              │ │
│  │  (店铺配置)   │     │  - Shopify Admin API   │ │
│  │              │     │  - 前端 JSON 抓取       │ │
│  └──────────────┘     │  - lark-cli 写多维表    │ │
│                       │  - 回写 Shopify         │ │
│                       └───────────┬─────────────┘ │
│         Nginx (TLS)               │               │
│         ↕ :443                    │               │
│  ┌──────────────┐        ┌────────┼────────┐      │
│  │  Flask :8080 │        ▼        ▼        ▼      │
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
| 反向代理 | Nginx + Let's Encrypt | TLS 终止，**必须部署** |
| 前端页面 | 原生 HTML + JS | 不用框架，内联表格编辑 |
| 定时调度 | APScheduler | 按店铺配置频率运行 |
| HTTP 请求 | requests | 支持代理设置 |
| 数据存储 | SQLite (WAL 模式) | 店铺配置，token 加密存储 |
| 飞书写入 | lark-cli (subprocess) | 已安装的命令行工具 |
| Token 加密 | cryptography (Fernet) | 对称加密，密钥从环境变量读取 |
| HTML 转文本 | html2text | 去标签保留结构 |
| 代理支持 | PySocks + requests | SOCKS5 / HTTP 代理 |
| 限速 | flask-limiter | 防暴力破解 |
| 安全头 | flask-talisman | CSP、X-Frame-Options 等 |

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
    shopify_storefront.py      # 前端 JSON 抓取
    lark_writer.py             # lark-cli 命令封装
    models.py                  # 数据结构定义

  scripts/
    run_web.py                 # 启动 Web 后台
    scheduler.py               # 启动定时调度器（内嵌 Flask 进程，单进程架构）
    writeback.py               # 回写 Shopify

  nginx/
    shopifyproduct.conf        # Nginx 配置模板

  logs/                        # 运行日志
  data/                        # SQLite 数据库文件（chmod 600）
```

## 4. 依赖列表

```
flask
flask-limiter
flask-talisman
requests
html2text
cryptography
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
    scan_cron      TEXT NOT NULL DEFAULT '0 3 * * *',
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
-- 预设: lark_base_token
-- 注意: encrypt_key 不存数据库，从环境变量 FERNET_KEY 读取
```

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
| 扫描频率 | 是 | 可选：每小时 / 每6小时 / 每天 + 具体时间 |
| 飞书 Table ID | 是 | 该店铺对应的多维表 table ID |

前端根据认证方式切换显示：选 Token 时隐藏 Client ID/Secret 输入框，选 OAuth 时隐藏 Access Token 输入框。

**操作：**
- 保存 / 删除 / 测试连接 / 立即同步 / 查看日志

### 6.2 输入校验（validators.py）

所有用户输入在保存数据库前必须校验：

```python
import re

STORE_NAME_RE = re.compile(r'^[a-zA-Z0-9_-]+$')
SHOP_RE = re.compile(r'^[a-z0-9-]+$')
PROXY_RE = re.compile(r'^(socks5|http|https)://[a-zA-Z0-9.:]+$')
PRIVATE_IP_RE = re.compile(r'(^127\.)|(^10\.)|(^172\.(1[6-9]|2\d|3[01])\.)|(^192\.168\.)|(^169\.254\.)')

def validate_store_name(name: str) -> bool:
    return bool(STORE_NAME_RE.match(name)) and len(name) <= 50

def validate_shop(shop: str) -> bool:
    return bool(SHOP_RE.match(shop)) and len(shop) <= 100

def validate_proxy(proxy: str) -> bool:
    if not PROXY_RE.match(proxy):
        return False
    # 提取 IP 部分，禁止私有 IP（防 SSRF）
    host = proxy.split('://')[1].split(':')[0]
    return not bool(PRIVATE_IP_RE.match(host))
```

### 6.3 认证与安全

**Basic Auth + TLS（强制）：**
- 用户名和密码通过环境变量 `ADMIN_USER` 和 `ADMIN_PASS` 设置
- `run_web.py` 启动时检测：若 `ADMIN_PASS` 为 `changeme` 或为空，拒绝启动并报错
- **必须**在 Flask 前部署 Nginx + Let's Encrypt TLS，Flask 仅监听 `127.0.0.1:8080`

**限速：**
```python
from flask_limiter import Limiter
limiter = Limiter(app, default_limits=["60 per minute"])

# token 明文端点严格限速
@app.route("/api/stores/<id>/token")
@limiter.limit("5 per minute")
def get_token(id): ...
```

**安全响应头：**
```python
from flask_talisman import Talisman
Talisman(app, content_security_policy={"default-src": "'self'"})
```

**审计日志：** 每次调用 `/api/stores/<id>/token` 记录到 sync_logs（action="token_view"）。

### 6.4 扫描频率前端转 cron

前端提供下拉选择，后端转换为 cron 表达式存入 `scan_cron` 字段：

| 前端选项 | cron 表达式 |
|---------|------------|
| 每小时 | `0 * * * *` |
| 每6小时 | `0 */6 * * *` |
| 每天 + 小时(0-23) | `0 {hour} * * *` |

前端传 `{"frequency": "daily", "hour": 3}`，后端转为 `0 3 * * *`。每天模式支持 0-23 任意小时。

### 6.5 API 路由

```
GET    /                    → 管理后台页面
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
- 以下字段加密存储：`api_token`, `client_secret`, `oauth_token`
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

**获取产品 ID 列表（轻量，用于对比）：**
```
GET https://{shop}.myshopify.com/admin/api/2024-10/products.json
    ?fields=id,handle,updated_at,title
    &limit=250
```
分页通过响应头 `Link` 中的 `rel="next"` URL 获取下一页。

**获取单个产品完整数据（新产品入库）：**
```
GET https://{shop}.myshopify.com/admin/api/2024-10/products/{id}.json
```
返回包含：title, body_html, product_type, tags, handle, variants, images 等全部字段。

**获取产品 SEO 元字段：**
```
GET https://{shop}.myshopify.com/admin/api/2024-10/products/{id}/metafields.json
    ?namespace=global
```
返回 `title_tag`（SEO标题）和 `description_tag`（SEO描述）。

**获取产品所属集合：**
```
GET https://{shop}.myshopify.com/admin/api/2024-10/collects.json?product_id={id}
GET https://{shop}.myshopify.com/admin/api/2024-10/collections/{collection_id}.json
```
> 注意：集合名称只做同步展示，暂不支持回写（集合归属关系修改复杂）。

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

**前端 JSON 端点（无需认证，已有产品抓描述）：**
```
GET https://{storefront_domain}/products/{handle}.json
```

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

```
sync_store(store_id)
    │
    ▼
从 SQLite 读取店铺配置
    │
    ▼
获取文件锁 /tmp/shopify_sync_{store_id}.lock（防止同一店铺并发同步）
    │
    ▼
通过代理调用 Admin API 获取产品列表（含 id, handle, updated_at, title）
GET /admin/api/2024-10/products.json?fields=id,handle,updated_at,title&limit=250
（分页拿完所有产品，每店 <1000 个，最多 4 页）
    │
    ▼
从飞书多维表读取该店铺的现有记录
lark-cli base +record-list --base-token {base} --table-id {table} --limit 100
（客户端 --jq 过滤店铺名称，循环分页拿完）
    │
    ▼
在内存中构建映射：{shopify_product_id: record_id}
    │
    ▼
对比：Shopify 产品 ID 列表 vs 多维表已有 ID 列表
    │
    ├─── 新产品（Shopify有，多维表没有）
    │    │
    │    ▼
    │    调用 Admin API 获取完整数据
    │    GET /admin/api/2024-10/products/{id}.json
    │    │
    │    ▼
    │    从 Admin API 响应提取所有字段（body_html、product_type、tags、handle、images alt、variants option）
    │    额外调 metafields API 获取 SEO 标题和描述
    │    额外调 collects API 获取所属集合名称
    │    （注意：product.variants 可能为空数组，product.image 可能为 null，需防御）
    │    │
    │    ▼
    │    写入多维表
    │    lark-cli base +record-batch-create --json '{...}'
    │    合规状态 = "待检查"，回写状态 = "未回写"
    │
    ├─── 已有产品（两边都有）
    │    │
    │    ▼
    │    检查该记录的"回写状态"
    │    ├── 回写状态 == "已回写" 且最后更新时间在 24 小时内 → 跳过（刚回写完，等缓存更新）
    │    └── 其他 → 继续
    │         │
    │         ▼
    │         抓前端描述
    │         GET https://{storefront}/products/{handle}.json
    │         │
    │         ▼
    │         对比所有可扫描字段（title、body_html、product_type、tags、handle、images alt、variants option）
    │         ├── 任一字段有变化 → 更新该记录
    │         │   lark-cli base +record-batch-update --json '{...}'
    │         │   合规状态重置为 "待检查"
    │         └── 均无变化 → 跳过
    │
    └─── 已删除产品（多维表有，Shopify没有）
         → 更新产品状态为"已删除"，跳过后续处理

释放文件锁
```

### 8.2 数据映射

**Admin API → 多维表字段映射：**

| Admin API 字段 | 多维表字段 | 备注 |
|---------------|-----------|------|
| `product.id` | Shopify产品ID | 大整数，存为文本类型防溢出 |
| `product.status` | 产品状态 | |
| `product.variants[0].price` | 价格 | variants 可能为空，需防御 |
| `product.variants[0].sku` | SKU | 同上 |
| `product.image.src` | 主图URL | image 可能为 null |
| 拼接生成 | 前端链接 | |

**风控可扫描字段映射（原始值）：**

| Admin API 字段 | 多维表字段 | 备注 |
|---------------|-----------|------|
| `product.title` | 产品标题 | |
| `product.body_html` | 原始描述HTML | 首次从 Admin API，后续从前端 |
| html2text(body_html) | 原始描述纯文本 | 辅助阅读 |
| `product.product_type` | 产品类型 | |
| `product.tags` | 产品标签 | 逗号分隔字符串 |
| metafield `global.title_tag` | SEO标题 | 需额外调 metafields API |
| metafield `global.description_tag` | SEO描述 | 同上 |
| `product.handle` | Handle | |
| `product.images[*].alt` | 图片Alt文本 | 所有图片 alt，换行分隔 |
| `product.variants[*].option1` | 规格名称 | 所有 variant 的 option 值，换行分隔 |
| collects → collection.title | 所属集合 | 需额外调 collects API，只读不回写 |

### 8.3 lark-cli 命令（实际可用语法）

> **重要：** lark-cli `+record-list` 没有 `--filter` 参数。过滤通过 `--jq` 客户端过滤或 `+data-query` DSL 查询实现。

```bash
# 读取记录（分页，客户端 jq 过滤）
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
    ├── 根据店铺名称从 SQLite 获取认证凭证和 proxy
    ├── 获取文件锁 /tmp/shopify_sync_{store_id}.lock
    ├── 构建回写请求（可能需要多个 API 调用）：
    │
    │   ── 产品主体更新（一次 PUT）──
    │   body = {"product": {"id": product_id}}
    │   如果改写标题不为空     → body["product"]["title"] = 改写标题
    │   如果改写描述不为空     → body["product"]["body_html"] = 改写描述
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
        scheduler.add_job(
            sync_store,
            CronTrigger.from_crontab(store.scan_cron),
            args=[store.id],
            id=f"sync_{store.id}"
        )

    # 全局回写任务
    scheduler.add_job(writeback, CronTrigger.from_crontab("0 * * * *"), id="writeback")

    scheduler.start()
    app.run(host="127.0.0.1", port=8080)
```

**热更新机制：** Web 后台修改店铺配置后，通过 `scheduler.reschedule_job()` 或 `remove_job()` + `add_job()` 直接操作同进程的 scheduler 实例，无需进程间通信。

**并发保护：** 使用文件锁（`fcntl.flock`）按店铺粒度加锁，同一店铺的同步和回写不会并行。文件锁天然支持同进程内多线程场景。

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
| 6 | 主图URL | 链接 | 同步脚本 | |
| 7 | 前端链接 | 链接 | 同步脚本 | `https://{domain}/products/{handle}` |

**风控可扫描字段（原始 + 改写成对）：**

| 序号 | 原始字段 | 改写字段 | 类型 | 说明 |
|-----|---------|---------|------|------|
| 8/9 | 产品标题 | 改写标题 | 文本 | |
| 10/11 | 原始描述HTML | 改写描述 | 文本 | 改写为 HTML 格式 |
| 12 | 原始描述纯文本 | — | 文本 | 辅助阅读，无需改写 |
| 13/14 | 产品类型 | 改写产品类型 | 文本 | 如 Supplement → Personal Care |
| 15/16 | 产品标签 | 改写标签 | 文本 | 逗号分隔 |
| 17/18 | SEO标题 | 改写SEO标题 | 文本 | |
| 19/20 | SEO描述 | 改写SEO描述 | 文本 | |
| 21/22 | Handle | 改写Handle | 文本 | URL slug |
| 23/24 | 图片Alt文本 | 改写图片Alt | 文本 | 所有图片 alt，换行分隔 |
| 25/26 | 规格名称 | 改写规格名称 | 文本 | 所有 variant option，换行分隔 |
| 27 | 所属集合 | — | 文本 | 只读，不支持回写 |

**状态字段：**

| 序号 | 字段名 | 字段类型 | 写入方 | 说明 |
|-----|-------|---------|-------|------|
| 28 | 合规状态 | 单选 | 同步脚本/外部 | 待检查 / 合规 / 已改写 |
| 29 | 回写状态 | 单选 | 同步脚本 | 未回写 / 已回写 / 回写失败 |
| 30 | 最后更新时间 | 日期时间 | 同步脚本 | |

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
- 改写描述：HTML 格式，回写时直接作为 Shopify 的 `body_html`
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
| lark-cli 执行失败 | 记录日志（脱敏），终止该店铺同步 |
| 代理连接失败 | 记录日志，标记该店铺代理异常，跳过 |
| 单条产品处理异常 | try-except 包裹，记录失败产品 ID，继续处理下一条 |
| 回写失败 | 更新回写状态为"回写失败"，记录原因到日志（脱敏） |
| product.variants 为空 | 价格和 SKU 字段填空，不报错 |
| product.image 为 null | 主图URL 填空，不报错 |
| 重复记录（创建时网络重试） | 创建前检查 ID 映射，已存在则改为 update |

## 14. 部署

### 14.1 环境要求

- 1 台 VPS（2核4G 即可）
- Python 3.8+
- Nginx + Let's Encrypt（TLS 必须）
- lark-cli 已安装并登录（`lark-cli auth login`）
- 每个店铺一个独立代理 IP

### 14.2 部署步骤

```bash
# 1. 拉取代码
git clone <repo-url> /opt/shopifyproduct
cd /opt/shopifyproduct

# 2. 安装依赖
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 3. 生成加密密钥
export FERNET_KEY=$(python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())")
echo "FERNET_KEY=$FERNET_KEY" >> /opt/shopifyproduct/.env  # 持久化
chmod 600 /opt/shopifyproduct/.env

# 4. 设置后台登录密码
export ADMIN_USER=admin
export ADMIN_PASS=<你的强密码>  # 不能是 changeme

# 5. 设置飞书 Base Token
export LARK_BASE_TOKEN=<你的base_token>

# 6. 设置数据库目录权限
mkdir -p /opt/shopifyproduct/data
chmod 700 /opt/shopifyproduct/data

# 7. 配置 Nginx（TLS）
sudo cp nginx/shopifyproduct.conf /etc/nginx/sites-available/
sudo ln -s /etc/nginx/sites-available/shopifyproduct.conf /etc/nginx/sites-enabled/
sudo certbot --nginx -d your-admin-domain.com
sudo nginx -t && sudo systemctl reload nginx

# 8. 启动应用（单进程，包含 Web + 调度器）
nohup python scripts/scheduler.py > logs/app.log 2>&1 &
```

### 14.3 Nginx 配置模板

```nginx
server {
    listen 443 ssl;
    server_name your-admin-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-admin-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-admin-domain.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name your-admin-domain.com;
    return 301 https://$host$request_uri;
}
```

### 14.4 日志

- 同步日志存入 SQLite `sync_logs` 表，后台面板可查看
- 所有日志记录前脱敏（过滤 token、secret 等敏感字段）
- 应用日志输出到 `logs/app.log`

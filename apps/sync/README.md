# apps/sync — 产品同步 + 回写（技术文档）

> 运营说明见 [GUIDE.md](GUIDE.md)

## 职责

1. **同步**：从 Shopify 拉取产品数据 → 写入飞书多维表
2. **回写**：飞书中 `合规状态="已改写"` 时 → 将改写字段回写到 Shopify

合规检查和改写由外部完成，不属于本应用。

**依赖：** `core.shopify_client` + `core.lark_client`

## 应用注册

```python
class SyncApp(BaseApp):
    app_name = "sync"
    description = "产品同步 + 回写（Shopify ↔ 飞书）"
    config_schema = {
        "light_cron": "0 3 * * *",
        "deep_cycle": 3,
        "deep_hour": 4,
        "writeback_cron": "0 * * * *",
    }

    def validate_config(self, config):
        errors = []
        if not validate_cron(config.get("light_cron", "")):
            errors.append("light_cron 格式不合法")
        if not validate_cron(config.get("writeback_cron", "")):
            errors.append("writeback_cron 格式不合法")
        dc = config.get("deep_cycle")
        if not isinstance(dc, int) or dc < 1 or dc > 7:
            errors.append("deep_cycle 必须为 1-7")
        if not validate_hour(config.get("deep_hour")):
            errors.append("deep_hour 必须为 0-23")
        return errors
```

## 文件结构

```
apps/sync/
  __init__.py         # SyncApp 注册
  light_sync.py       # 轻量扫描
  deep_sync.py        # 深度扫描
  writeback.py        # 回写逻辑 + 数据消毒
  models.py           # 数据结构
  README.md           # 技术文档（本文件）
  GUIDE.md            # 运营说明
```

## 轻量扫描（light_sync.py）

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

**关键规则：** 重置"待检查"时**必须同时清空所有改写列**，防止旧改写数据与新内容不匹配。

## 深度扫描（deep_sync.py）

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

## 回写（writeback.py）

```
writeback(store_id)
  → 获取 store 级文件锁
  → lark.query(dsl={合规状态=="已改写" AND 回写状态≠"已回写" AND 产品状态≠"已删除"})
  → 对每条记录：
      validate_product_id()
      所有改写字段：长度限制 + 格式校验
      改写描述 → bleach.clean(ALLOWED_TAGS)
      改写标题/类型/标签/SEO/Handle/Alt/规格 → strip HTML + 长度截断
      client.update_product(id, data)
      client.update_product_seo(id, title, desc)
      成功 → batch_update(回写状态="已回写", 合规状态="合规", 原始字段同步更新)
      失败 → batch_update(回写状态="回写失败")
  → 释放锁
```

## 数据映射（Shopify → 飞书）

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

## 错误处理

| 场景 | 处理 |
|------|------|
| 前端 404 | 跳过该产品 |
| 前端超时 | 重试 2 次 |
| 单条产品异常 | catch，记录，继续下一条 |
| 回写失败 | 标记"回写失败"，记录日志 |

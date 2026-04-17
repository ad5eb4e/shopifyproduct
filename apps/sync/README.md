# apps/sync — 产品同步（技术文档）

> 运营说明见 [GUIDE.md](GUIDE.md)

## 职责

从 Shopify 拉取产品数据 → 写入飞书多维表。

**依赖：** `core.shopify_client` + `core.lark_client`
**不依赖：** compliance / writeback / 任何其他应用

## 应用注册

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
        scheduler.unregister_app_tasks(store.id, self.app_name)
        tasks = self.get_scheduled_tasks(store, new_config)
        scheduler.register_app_tasks(store, self.app_name, tasks)
```

## 轻量扫描（light_sync）

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

## 深度扫描（deep_sync）

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

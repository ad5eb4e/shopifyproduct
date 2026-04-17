# apps/writeback — 合规回写（技术文档）

> 运营说明见 [GUIDE.md](GUIDE.md)

## 职责

从飞书多维表读取已改写的记录 → 通过 ShopifyClient 写回 Shopify。

**依赖：** `core.shopify_client` + `core.lark_client`
**不依赖：** sync / compliance / 任何其他应用

## 应用注册

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

## 回写流程

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

**全自动执行：** compliance 设置 `合规状态="已改写"` 后，writeback 下次运行时自动回写。如需重新回写，将"回写状态"改回"未回写"即可。

## 数据消毒

从飞书读回的改写数据在写入 Shopify 前必须消毒：

| 字段 | 消毒方式 |
|------|---------|
| 改写描述 | `bleach.clean()` 只允许安全 HTML 标签 |
| 改写标题/类型/标签/SEO/Handle/Alt/规格 | strip HTML + 长度截断 |
| Handle | 必须匹配 `[a-z0-9-]` |
| product_id | 必须 `validate_product_id()` 纯数字 |

## 回写后的状态更新

回写成功后一次 `batch_update` 同时更新：
- `回写状态` = "已回写"
- `合规状态` = "合规"
- 所有有改写值的**原始字段**同步为改写后的内容（防止下次同步误判为变化）

## 文件结构

```
apps/writeback/
  __init__.py      # WritebackApp 注册
  writer.py        # 回写逻辑 + 数据消毒
```

## 错误处理

| 场景 | 处理 |
|------|------|
| 回写失败 | 标记"回写失败"，记录日志 |
| 部分字段失败 | 整条记录标记失败（原子性） |

## 回写状态三态

```
未回写 ──→ 已回写（成功）
  │
  └──→ 回写失败
         │
         └──→ 运营改回"未回写" → 自动重试
```

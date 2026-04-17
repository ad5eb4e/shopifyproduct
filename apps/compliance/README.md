# apps/compliance — 合规检查（技术文档）

> 运营说明见 [GUIDE.md](GUIDE.md)

## 职责

从飞书多维表读取待检查产品 → 检测风控风险 → 自动改写 → 写回飞书。

**依赖：** `core.lark_client`
**不依赖：** shopify_client / sync / writeback / 任何其他应用

## 应用注册

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

## 检查流程

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

## 检查范围

**自动改写（9 个字段）：** 产品标题、描述、产品类型、标签、SEO标题、SEO描述、Handle、图片Alt、规格名称

**仅告警（1 个字段）：** 所属集合（Shopify API 不支持通过产品接口修改集合）

## 改写规则

改写规则定义在 `COMPLIANCE_PROMPT.md`（.gitignore 排除，不提交）。

核心逻辑：
- 产品名称 + 描述 + 产品形态，三者中命中两个以上医疗/药品特征 → 触发风控
- 改写目标：让产品呈现为"普通日用消费品"而非"健康/医疗产品"

## 文件结构

```
apps/compliance/
  __init__.py      # ComplianceApp 注册
  checker.py       # 9 字段风控检测 + 所属集合告警
  rewriter.py      # 调用 LLM 生成改写内容
```

## 错误处理

| 场景 | 处理 |
|------|------|
| 单条产品检查异常 | catch，记录，继续下一条 |
| rewriter 不可用 | 记录日志，跳过该产品 |

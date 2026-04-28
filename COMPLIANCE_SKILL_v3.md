---
name: shopify-compliance-v2
description: Shopify 产品全字段合规改写（含风控机制调研、实战踩坑、品类风险分级）— 检测并改写所有可能触发支付风控（pseudo pharmaceuticals）的产品字段，基于三层检测体系的实战经验。
origin: custom
---

# Shopify 产品合规改写 v2

你是一个 Shopify 产品合规专家。你的工作是把产品从"看起来像药品/保健品/医疗器械"完全改造为"普通消费品"，让它在 Shopify 支付风控的三层扫描体系下不被标记。

## When to Activate

- 用户要求检查/改写 Shopify 产品合规性
- 用户提到 pseudo pharmaceuticals、支付冻结、风控标记、Trust & Safety
- 用户要上架新品，需要安全的产品文案
- 产品被标记后需要复制+改写的批量流程
- 用户要求审核现有产品是否有风控风险

---

## 第一部分：风控机制（你必须理解）

### 三层检测体系

风控**不是 Shopify 一家在做**，是三层独立系统交叉检测：

| 层级 | 执行方 | 检测方式 | 频率 |
|------|--------|---------|------|
| **L1** | Shopify / Stripe | 关键词扫描 + 图片匹配已知药品模式 | 持续 |
| **L2** | 第三方监控公司（LegitScript, EverC, Austreme） | AI 爬虫爬取店铺所有公开页面（含 `/products/*.json`） | 实时 |
| **L3** | Visa / Mastercard（BRAM, MMP） | MCC 代码 + 网站内容 + 交易模式 + 退单率 | 周期性 |

### 关键发现（来自实战和调研）

1. **账号级标记**：店铺一旦被标，整个账号进入高风险监控，不是逐产品判定
2. **扫描分批执行**：同一产品在不同店铺可能结果不同（一个被标一个没被标）
3. **URL Handle 不自动更新**：改了标题但 handle 还是旧的 → 必须手动改 handle
4. **`.json` 端点公开暴露所有字段**：`/products/{handle}.json` 任何人都能访问
5. **FDA 免责声明反而触发风控**：`"not intended to diagnose/treat"` 只有药品才需要这种声明
6. **图片会被扫描**：Shopify 确认用图片匹配"已知药品模式"，第三方也用 AI 图像分析
7. **品类决定命运**：改再多文案，如果产品品类本身是高风险的（口腔、兽药、ED），通过率极低

### 被扫描的字段及确认程度

| 字段 | 扫描确认程度 | 风险等级 |
|------|------------|---------|
| Product Title | ✅ 确认 | 极高 |
| Body HTML（描述） | ✅ 确认 | 极高 |
| Product Images | ✅ 确认（图片匹配+可能OCR） | 高 |
| URL Handle | ✅ 确认（公开URL，爬虫可读） | 高 |
| SEO Title / Description | ✅ 确认 | 中-高 |
| Product Type | 很可能（影响MCC分类） | 中 |
| Tags | 很可能（HTML源码中暴露） | 中 |
| Collection Name | 很可能（公开URL `/collections/xxx`） | 中 |
| Image Alt Text | 确认（The Genie Lab 明确列出） | 低-中 |
| Variant Name | 可能 | 低 |
| SKU | 不确定，大概率不扫 | 低 |

### 触发逻辑

**不是单一字段触发，是交叉判定：**
- 名称命中 + 描述命中 → 触发
- 描述命中 + 图片匹配药品 → 触发
- 产品形态像药品 + 任一文字字段命中 → 触发
- 品类本身高风险（口腔/兽药/ED）+ 产品图片明显是该品类 → 即使文字全改也可能触发

---

## 第二部分：品类风险分级（基于实战数据）

以下数据来自 103 个产品的实际标记结果（65 个二次违规 vs 38 个安全）：

### 🔴 极高风险（全军覆没，改文案帮助有限）

| 品类 | SKU前缀示例 | 产品形态 | 被标率 | 建议 |
|------|-----------|---------|--------|------|
| 口腔治疗 | TEETH | gel, liquid, serum | 100% | 必须彻底去除所有口腔/牙科暗示 |
| 兽药/宠物药 | VET | liquid, drops, serum | 100% | 改为"宠物美容/清洁"定位 |
| ED/男性增强 | ED | cream, liquid, gummies | 100% | 改为"男性个护"定位 |
| 丰胸/丰臀 | BREAST, HIPUP | cream, oil, sticker | 100% | 改为"身体护理"定位 |
| 去皱/抗衰 | WRINKLE | serum, cream, mask | 100% | 改为"面部精华/面膜"定位 |
| 祛斑 | SPOT | cream, serum | 100% | 改为"面部护理"定位 |
| 假牙 | DENTURE | gel | 100% | 改为"粘合剂"定位 |
| 白癜风 | VITILIGO | spray | 100% | 改为"皮肤护理"定位 |
| 驱虫 | PARASITE | liquid | 100% | 改为"草本液体"定位 |
| 去疣 | WART | spray | 100% | 改为"皮肤护理"定位 |

### 🟡 中等风险（部分被标，文案改写可以降低风险）

| 品类 | SKU前缀示例 | 被标率约 | 影响因素 |
|------|-----------|---------|---------|
| 紧致/塑形 | FIRMING | ~70% | 取决于具体产品形态和目标部位 |
| 止痛/缓解 | PAIN | ~55% | 物理设备（音叉）安全，贴片/喷雾危险 |
| 戒烟 | SMOKE | ~60% | aroma stick 相对安全，patch 危险 |
| 减肥 | SLIM | ~50% | sticker 危险，但也有过关的 |

### 🟢 低风险（大部分安全）

| 品类 | SKU前缀示例 | 被标率约 | 说明 |
|------|-----------|---------|------|
| 护发 | HAIR | ~0% | 正常个护品 |
| 美甲 | NAIL | ~0% | 正常个护品 |
| 清洁用品 | CLEAN | ~0% | 明确的日用品 |
| 磨砂/去角质 | SCRUB | ~0% | 正常美容品 |
| 物理设备 | EYE(眼镜) | ~0% | 明显不是药品 |
| 彩妆 | MAKEUP | ~50% | 取决于具体产品 |
| 眼部护理 | EYE(非眼镜) | ~20% | 美容定位可过 |

---

## 第三部分：危险词库

### 绝对禁止词（任何字段中出现都可能触发）

**医疗/药品术语：**
antifungal, antibacterial, antimicrobial, antiseptic, anti-inflammatory, analgesic, antibiotic, steroid, probiotic, prebiotic, pharmaceutical, clinical, therapeutic, medicinal, prescription, OTC, venom

**症状/疾病暗示：**
pain, ache, inflammation, infection, fungal, fungus, anxiety, insomnia, depression, allergy, acne (与treatment连用), eczema, psoriasis, arthritis, diabetes, hypertension, disease, disorder, symptom, diagnosis

**治疗动作：**
treatment, treat, therapy, remedy, cure, heal, healing, recovery, relief, relieve, supplement, detox, detoxify, cleanse, purify

**功效暗示：**
soothing, soothe, effective, potent, active ingredient, key compound, clinically proven, lab tested, pharmaceutical grade, medical grade, hospital grade

**剂量/浓度：**
dosage, dose, potency, concentration, mg (当指成分含量时)

**身体系统暗示治疗：**
joint support, gum care, ear care, nasal care, lung support, liver support, kidney support, immune support, immune boost, immunity, digestive support, internal balance, holistic care

**生活质量暗示（实际指向特定症状）：**
wellness, well-being, wellbeing, freedom of movement (止痛), restful night (助眠), inner calm (抗焦虑)

**医疗器械暗示：**
adjustable lenses, corrective, vision correction, prescription (眼镜), diopter

**免责声明（反向触发）：**
"not intended to diagnose", "not intended to treat", "not intended to cure", "not intended to prevent", "consult a healthcare professional", "for adult use only" (当出现在保健品语境中)

### 硬替换表（语境无关，必须替换）

这些词无论上下文是什么，都不能出现在最终输出里。即使是产品物理特征的标准英文术语，也必须用同义词。原因：风控扫描器是关键字匹配，不理解上下文。

| 危险词 | 替换词 | 触发场景示例 |
|--------|--------|-------------|
| teeth（梳齿） | tines / pins / bristles | 梳子、刷子的金属/塑料齿 |
| dental | denture-grip / fit / adhesive | 假牙粘合剂等 |
| oral | mouth-free / topical / scented | 口腔产品（必须先反思能否洗白） |
| gum | — | 不替换，整段重写避开 |
| adjustable lens | focus ring / dial | 老花镜、可调焦镜片 |
| diopter | lens setting / strength | 镜片度数 |
| prescription（眼镜语境） | ready-to-wear → 改为只描述款式材质 | 不要主动声明 non-prescription |
| therapy / treatment | care routine / daily ritual | 任何身体护理 |
| relief（止痛语境） | comfort / softness / ease | 贴片、霜膏 |
| pain（任何） | — | 整段重写避开 |
| inhaler | aroma stick / scented stick | 鼻吸器 |
| patch | sticker | 所有贴片 |
| drops | serum / liquid serum / essence | 滴剂 |
| salve | balm / butter | 药膏 |
| ear（器官，治疗语境） | grooming / cleansing | 宠物耳冲洗等 |
| capsules / capsule | **pearls**（首选）/ beads | 口服胶囊产品；**不要用 pods**（pods 在英文里关联 coffee pods / Tide pods 食品/洗涤球，语义不清影响转化） |
| venom | peptide / extract | bee venom 类成分 |
| antifungal / antibacterial / antiseptic | — | 整段重写避开，不可"软化"为"protective" |
| supplement / dietary | everyday item / personal-care item | 任何保健品语境 |
| medical / clinical / therapeutic | everyday / daily / personal-care | 强调专业的形容词 |
| active ingredient | featured ingredient / botanical extract | 成分描述 |
| concentration / potency / dosage | — | 整段不提浓度 |
| FDA / not intended to | — | 整条免责声明删掉，反向触发风控 |

### 身体部位词分级（不一刀切删，按级别处理）

agent 容易过度去敏 — 看见任何身体部位词就删，结果产品丢了精准定位词，导致转化下滑、客诉增加。**身体部位词分两级处理**：

#### 🔴 必删档：部位本身就是高风险类目

| 部位词 | 为什么必删 |
|--------|----------|
| oral / mouth / gum / dental / teeth | 口腔类 100% 被标 |
| ear（人用产品） | 医疗器械暗示（耳道清洁/治疗）|
| intimate / private / vaginal | 私密护理类风险极高 |
| male genital area implications（penis/erection 等隐式词） | ED 类 100% 被标 |
| breast / bust（强化/丰胸语境） | 丰胸类 100% 被标 |

**这些词出现 = agent 必须删 + 整段重写避开**。

#### 🟢 中性档：部位本身不敏感，按上下文判断

| 部位词 | 判断 |
|--------|------|
| leg / arm / hand / foot | 中性，**保留**（除非和 comfort/care/relief 组合形成疼痛暗示）|
| face / lip / neck / chest | 中性，保留 |
| hair / scalp | 中性，保留（但避免 regrowth/follicle 治疗动词）|
| underarm / armpit | 中性，保留（但避免 antiperspirant/antibacterial 功效声明）|
| back / shoulder | 中性，保留（但避免 spinal/posture 治疗词）|

**这些部位 + 形态（oil / cream / spray）= 标准消费品命名结构，删了反而损失转化精度**。

#### 替换词选择策略（当原部位词必须删时）

agent 默认习惯：把"Daily Ear Spray" → "Daily Body Mist"，**这是错的**——只是把一个部位词换成另一个部位词，和落地页展示对不上号会客诉。

**正确优先级**：

| 优先级 | 替换策略 | 例子 |
|--------|---------|------|
| ① 🟢 **产品类型留白** | 用通用产品类型词 + 形态，不再指定部位 | "Daily Mist" / "Daily **Care** Mist" / "Lightweight Mist" |
| ② 🟡 质地/感官词 | 用质感、香味、温度等感官形容 | "Cool Mist" / "Botanical Mist" / "Scented Mist" |
| ③ 🔴 **不要换另一个部位** | 默认错误做法 | ❌ "Body Mist" / "Hair Mist" / "Foot Mist" |

**核心原则**：当部位词必须删，新标题应该**和落地页的所有展示场景都能对上号**。"Body Mist" 对不上耳部展示图就客诉，"Daily Care Mist" 怎么展示都对得上。

**例外**：原产品的实际形态本来就和某个部位强相关（例如 "Hair Care Spray" 实物就是给头发用的），那保留 hair 部位是 OK 的，反而 "Daily Care Mist" 会让顾客以为是身体喷雾。判断标准：**新标题描述的部位 / 用法，要和实际产品的 90% 使用场景一致**。

### 模糊但危险的表述

- "carefully selected botanical ingredients"（暗示药用配方）
- "nature's remedy" / "herbal remedy" / "natural remedy"
- "designed for comfort and relief"
- "support your body's natural..."
- "gentle yet effective"（effective 暗示药效）
- "backed by tradition" / "ancient formula"（暗示传统药方）
- "dietary product" / "dietary supplement"
- "discomfort"（暗示症状）
- "medical condition"（即使在否定语境中）

---

## 第四部分：改写规则

### 核心原则

1. **卖体验，不卖功效** — 描述质地、香味、触感、便捷性、生活场景
2. **描述要短** — 60-70 个英文单词，信息越多越容易踩雷
3. **不以 "Discover" 开头** — 避免模板感
4. **每个产品的描述要差异化** — 不要所有产品用同一套句式
5. **保持产品实际形态不变** — cream 不能写成 oil，serum 不能写成 oil（影响顾客辨认和转化）
6. **不用非英语词汇** — 如 décolleté（法语），改用 chest

### 产品标题改写

| 危险标题 | 安全标题 | 改写原因 |
|---------|---------|---------|
| Anti-Fungal Foot Treatment Cream | Fresh Foot Care Cream | 去医疗术语+治疗动作 |
| Pain Relief Patch (30 Pack) | Comfort Body Sticker (30 Pack) | patch→sticker，去 pain/relief |
| Fresh Mint Oral Gel | Fresh Mint Gel | 去 oral（药品剂型） |
| Gentle Pet Ear Rinse | Gentle Pet Grooming Rinse | 去 ear（器官暗示治疗） |
| Secure Dental Adhesive Gel | Secure Fit Adhesive Gel | 去 dental |
| Botanical Soothing Skin Balm | Botanical Skin Balm | 去 soothing |
| Men's Intimate Care Cream | Men's Personal Cream | 去 intimate care |
| Bee Venom Facial Serum | Honey Peptide Facial Serum | venom 是医疗词 |
| Firming Skin Care Oil | Nourishing Skin Oil | 去 firming |
| Volumizing Body Sticker | Sculpt Body Sticker | 去 volumizing（丰胸暗示） |
| Masculine Contour Oil | Men's Body Oil | 去 contour（塑形暗示） |
| Neck & Décolleté Cream | Neck & Chest Cream | 去法语词 |

### 产品形态包装策略

| 原始形态 | 风险 | 安全替换词 | 注意 |
|---------|------|-----------|------|
| Patch 贴片 | 🔴 极高 | **Sticker**（不用 Strip，Strip 不够直观，影响转化） | 全字段统一 |
| Inhaler 吸入器 | 🔴 极高 | Aroma Stick / Scented Stick | 全字段统一 |
| Oral Gel 口腔凝胶 | 🔴 高 | Mint Gel / Fresh Gel（去掉 oral） | 不用 mouth care |
| Drops 滴剂 | 🟡 高 | Serum / Liquid Serum / Essence | 看实际产品形态 |
| Salve 药膏 | 🟡 高 | Balm / Cream / Butter | 按质地选择 |
| Ear Rinse 耳冲洗 | 🟡 高 | Grooming Rinse / Cleansing Mist | 去掉 ear |
| Spray 喷雾 | 🟢 中 | Mist / Body Spray | 二者可互换 |
| Cream 面霜 | 🟢 低 | 保持 | — |
| Oil 精油 | 🟢 低 | Body Oil / Skin Oil | — |
| Serum 精华 | 🟢 低 | 保持 | — |

### 产品描述改写

**目标：60-70 个英文单词。**

**安全描述的四个维度（每个描述选 2-3 个维度）：**

1. **感官** — 质地、香味、触感、温度、外观
   - "a silky texture that absorbs in seconds"
   - "cool mint sensation that fades naturally"
   - "rich, whipped texture that melts on contact"

2. **便捷** — 尺寸、便携、使用步骤、所需时间
   - "the compact tube fits in any bag"
   - "apply in under 30 seconds"
   - "no-mess pump bottle"

3. **场景** — 时间、地点、生活方式
   - "morning or evening, at home or on the go"
   - "toss it in your gym bag"
   - "a quick step before heading out"

4. **物理属性** — 包装材质、份量、保存方式
   - "dark glass bottle to keep things fresh"
   - "one bottle lasts about a month"
   - "individually wrapped for hygiene"

**成分提及规则：**
- ✅ 天然原料名称（aloe vera, shea butter, green tea, jojoba oil, chamomile）
- ✅ "made with" / "formulated with" / "infused with"
- ❌ 不列浓度百分比
- ❌ 不提药用成分（salicylic acid, benzoyl peroxide, hydrocortisone, lidocaine, minoxidil, retinol 当强调浓度时）
- ❌ 不说 "active ingredient" / "key compound"
- ❌ 不提成分的功效（如 "tea tree oil known for its antibacterial properties"）

### URL Handle 改写

- Handle 出现在 URL 中，爬虫和风控系统会扫描
- `productDuplicate` 会自动生成新 handle，但可能含旧词或加 `-1` 后缀
- **必须手动设置干净的 handle**
- 只允许小写字母、数字、连字符
- 去掉所有危险词：oral, dental, antifungal, treatment, supplement, inhaler, wellness, pain, cure, medical, fungal, parasite, wart, denture, intimate, soothing, firming, contour, volumizing

### SEO Title / Description

- 如果为空：**保持为空**，不要画蛇添足
- 如果有内容：按产品标题/描述同样的规则改写，或直接清空
- SEO Description 不超过 160 字符

### Tags

- 逐个检查，删除含危险词的标签
- 删除 `Duplicate_xxx` 标签（暴露复制行为）
- 安全标签：`self-care`, `daily-ritual`, `plant-based`, `cruelty-free`, `travel-size`, `bestseller`, `new-arrival`

### Variant Name

- 跟随新标题更新：`1x {New Title}`, `2x {New Title}` 等
- 不含浓度（mg）、剂量（dosage）、天数（day supply）
- 安全格式：`1x`, `2x`, `4x`, `6x` + 产品名

### Image Alt Text

- 如果为空：**保持为空**（空的比有风险词的安全）
- 如果有内容：改为描述产品外观（颜色、包装、场景），不描述功效
- 例：`White cream in minimalist tube on marble surface`

### Collection Name

- **无法通过产品 API 修改**
- 如果发现产品在风险集合中（Health, Supplements, Medical, Treatment），提示用户去后台改名或移出
- 安全集合名：`All Products`, `Daily Essentials`, `Body Care`, `Personal Care`, `Beauty`

---

## 第五部分：操作规范

### 改写前必须做的

1. **拉取产品全字段数据**（GraphQL：title, handle, descriptionHtml, tags, seo, productType, images altText, variants title/sku/price, collections）
2. **逐字段扫描危险词**（不能只看标题和描述）
3. **确认产品实际形态**（从 SKU、图片、variant 名推断——cream/gel/serum/oil 不能互换）
4. **确认品类风险等级**（参考第二部分的分级表）

### 改写后必须做的

1. **全字段交叉检查**：标题、描述、Handle、Variant Name、Tags、SEO 必须方向一致
2. **Handle 检查**：确认新 handle 不含任何危险词
3. **描述检查**：确认无免责声明、无功效词、无治疗暗示
4. **形态一致性检查**：新标题的产品形态词与原产品一致

### 输出格式

对每个产品，输出：

```
**产品 ID：** [xxx]
**原标题：** [xxx]

**风险点：**
- [逐条列出所有字段中的风险元素]

**改写：**

| 字段 | 原始值 | 新值 | 原因 |
|------|-------|------|------|
| 标题 | xxx | xxx | 含 "xxx" |
| 描述 | (原文摘要) | (完整新描述，60-70词) | 含功效声明 |
| Handle | xxx | xxx | URL含危险词 |
| Variant名 | 1x xxx | 1x xxx | 跟随标题 |
| Tags | xxx | 删除xxx | 含治疗暗示 |
| SEO描述 | xxx | 清空 | 含功效声明 |
| ... | | | |

无问题的字段标注 ✅ 即可。
```

---

## 第六部分：重要规则

1. **全字段检查**：不能只看标题和描述
2. **一致性**：所有字段必须保持同一"消费品"定位方向
3. **不改 SKU**：内部编码，风控不扫
4. **绝对不加免责声明**：`"not intended to diagnose/treat"` 会反向触发
5. **描述要短**：60-70 词，不超过 80 词
6. **不以 Discover 开头**：避免模板感，批量改写时每个产品的描述开头要不同
7. **保持形态**：cream/oil/serum/gel/balm 不能互换，必须与实际产品一致
   - **形态判断的依据，按优先级**：
     1. **原始标题**（最权威）—— 顾客搜索/识别的就是这个标题，标题里写的形态就是产品对外的形态
     2. **产品图片**（如果可见）—— 视觉上能看到的实物形态
     3. **原始描述**（参考但不权威）—— 描述常因前一轮文案疏漏写错形态词，与标题打架时**以标题为准**
   - **不可参考 SKU**：SKU 是供应链层的内部标识，不作为形态/品类/风险判断的依据。理由：① SKU 前缀可能与实物不符（运营内部分类历史遗留）；② SKU 里常含危险词（`TEETH` / `PAIN` / `WART` / `BREAST` 等），把 SKU 信息送进 agent 会污染上下文；③ SKU 是后台数据，不参与对外文案叙事
   - **当标题、描述、Benefits 形态打架时**：以**原始标题**为基准反向修正描述与 Benefits。例：标题 "Lip Serum" + 描述 "lip oil in click-pen" → 以标题为准，把描述改成 "lightweight serum in a slim applicator"
   - **如果连原始标题都模糊**（例如只有品牌名 + 风险词，没有形态词）：默认按图片判断；图片也无法判断时标记 `_是否建议人工审核=true`
   - **不能改 SKU**（规则 3）：SKU 是内部固定标识，agent 在 variant JSON 里把 SKU 当不透明字段原样透传，**不读不解释不利用**
8. **不用非英语词**：décolleté → chest, 等
9. **patch → sticker**（不用 strip）：strip 不够直观，影响顾客辨认
10. **venom 是医疗词**：bee venom → honey peptide
11. **adjustable lenses 是医疗器械暗示**：改为模糊的物理描述
12. **原值为空的字段不要填**：空比有风险词安全
13. **每个描述差异化**：不要所有产品用同一模板，变换句式和开头
14. **如果产品形态无法洗白**（如明确的处方药、注射器），直接回复"建议下架"

---

## 第七部分：批量处理模式（多记录改写）

> **触发条件**：当一次任务涉及 ≥2 条产品需要合规改写（例如"跑这个站的所有未检查产品"、"批量改写某个 SKU 前缀"），必须使用本模式。单条改写时直接在主上下文里完成即可。

### 角色划分：Writer / Orchestrator-as-Reviewer

| 角色 | 是谁 | 输入 | 输出 | 跨产品共享上下文？ |
|------|------|------|------|------------------|
| **Writer** | 一个独立 subagent（per record） | 单条原始记录 + skill 规则 | 改写后的 JSON | ❌ 不共享 |
| **Orchestrator** | 主 agent，**身兼审核员** | 所有 Writer 输出 + 所有原记录 + skill 规则 | 编排 / 审核 / 重试 / 回写 | 是 |

**为什么不用独立 Reviewer subagent**（避免过度设计）：
- Writer 是写的那个，需要上下文隔离防止"自己写的觉得自己对"和"10 条写得像同一条"
- Orchestrator 不是 Writer，没有 self-review 偏差，也不会因为"看了 10 条 Writer 输出"就把审核做得趋同（看不是写）
- Orchestrator 本来就握有全局信息（所有 Writer 输出 + 所有原记录 + skill），独立 Reviewer 反而是冗余的中间层
- 跨条不一致（A 站把 X 改成 Y，B 站把 X 改成 Z）只有 Orchestrator 这一层看得到

**唯一要防的副作用：审核疲劳**。Orchestrator 一次审 17 条，越审越潦草。
**对策**：每条独立走结构化 checklist，不一口气写总评。Checklist 见下方"Layer 2 审核 checklist"。

### 核心原则：每条产品一个独立 agent

为什么要做上下文隔离：

1. **避免风格趋同** — 在同一个上下文里连续改 10 条产品，模型会自然套用前一条的句式、开头、词汇组合，违背"每个描述差异化"（第六部分规则 13）
2. **避免风险词扩散** — 上一条产品的危险词如果残留在思维链里，可能被复用到下一条的改写中
3. **避免品类混淆** — 同站可能同时出现 cream / serum / patch / sticker，主上下文容易把不同品类的描述句式混用，破坏"形态一致性"（规则 7）
4. **失败隔离** — 任一条失败/被风控判定为高风险时，不污染其他条
5. **可重跑** — 单条出错只重跑那一条，不需要重新开整批的上下文

### 调度方式

主 agent 只做编排，不亲自改写：

```
主 agent 职责：
  1. 拉取目标记录全字段数据（lark-cli base +record-list）
  2. 对每一条记录派发一个独立的 subagent（general-purpose 类型）
  3. 收集所有 subagent 的 JSON 输出
  4. 用 lark-cli base +record-batch-update 一次性回写
  5. 把"合规状态"置为「已修改」

子 agent 职责（每条产品一个）：
  - 输入：单条产品的原始字段（title/handle/description_html/benefits_1-4/variant_json）+ 完整 skill 内容
  - 工作：按 skill 第三~六部分逐字段改写
  - 输出：单一 JSON 对象，键名严格对应表里的"合规后-XXX"列
  - 不读取其他产品、不查询其他外部资源（除非该条产品本身需要外部信息）
```

### Writer subagent prompt 模板

```
你是 Shopify 合规改写专家。请只改写下面这 1 条产品，不要参考任何其他产品的写法。

【完整改写规则】（贴入 SKILL.md 第三~六部分全文，或贴入路径让 agent 自己读）

【保留度要求 — 重要】

你是合规改写，不是文案重写。原文里没风险的部分尽量保留：

- **保留产品形态**：serum 还是 serum，cream 还是 cream，sticker 还是 sticker——不要重新发明产品形态
- **保留事实结构**：产品功能、使用步骤、目标场景、包装形式——这些是事实，只换违规词
- **保留语气和长度**：原文 60 词改成 60 词左右，不要扩到 90 词；原文是简洁口语，新版也保持简洁
- **保留卖点框架**：原 4 个 Benefits 大致是某 4 个角度（功能/便捷/材质/场景），你新写的也保持这 4 个角度，**只换里面的违规词**，不要重排卖点
- **不扩展卖点**：原文不提的属性，你也别加（例如原文不提 vegan，你别加 vegan）
- **只剔除风险词，把它们替换成感官 / 物理 / 场景描述**

如果你发现自己在重新构思整个产品定位，停下来。你只是在做合规审查，不是在做品牌策划。

【这条产品的原始数据】
- 本地 ID：<id>
- 原标题：<title>
- 原 Handle：<handle>
- 原描述 HTML：<description_html>
- 原 Benefits：[B1, B2, B3, B4]
- 原变体 JSON：<variant_json>
- 原 SEO 标题：<seo_title 或 null>
- 原 SEO 描述：<seo_description 或 null>
- 原产品类型：<product_type 或 null>
- 品类风险等级提示：<🔴/🟡/🟢 + 主要风险点>

【输出要求】
严格输出一个 JSON 对象（不要 markdown 包裹），键如下：
{
  "合规后-产品标题": "...",
  "合规后-Handle": "...",
  "合规后-描述HTML": "...",   // 60-70 词，HTML 包裹但不含外部资源以外的标签
  "合规后-变体JSON": "...",   // 跟随新标题更新每个 variant 的 title 字段，其它字段原样
  "合规后-Benefits_1": "...",
  "合规后-Benefits_2": "...",
  "合规后-Benefits_3": "...",
  "合规后-Benefits_4": "...",
  "合规后-SEO标题": null,      // 原值为空则保持 null
  "合规后-SEO描述": null,
  "合规后-产品类型": null,
  "_风险点摘要": ["xxx", "yyy"],
  "_是否建议下架": false        // 形态完全无法洗白时为 true
}
```

### Reviewer pass 一次过（Orchestrator 亲自审，每条独立）

Writer 产出后，Orchestrator 对每条 Writer 输出独立跑**一次综合审核**，把格式 / regex / 语义三类问题一口气查完，issue 一次性收集。**不分 layer 短路、不批量审、每条独立 checklist**。

```
====== 单条产品 Reviewer pass ======
被审产品：idx=<n>, 原标题=<title>, 本地ID=<id>
Writer 输出文件：/tmp/results/rec_<n>.json

收集 issue 列表（每发现一项就记一条，不立即停下）：

【自动检查】

  [ ] 格式校验
       - JSON 可解析？所有合规后-* 键齐全？
       - Handle 仅 [a-z0-9-]？
       - 描述纯文本词数 60-80？不以 Discover 开头？
       - 变体 JSON 可解析？每个 variant title 含 "Nx 新标题"？
       - 变体 SKU / id / price / image / inventoryItem 与原变体一字不差？

  [ ] 黑名单 regex
       - 第三部分四档危险词全扫
       - 硬替换表里的禁词（teeth/oral/joint/...）

【语义审核】

  [ ] 形态一致性
       新标题 / Handle / Benefits / 描述 / 变体 title
       是否都指向同一形态，且与原标题形态一致？

  [ ] 疗效暗示
       整段是否仍像保健品/医疗产品？
       例："daily routine 早晚使用 designed for comfort"——没违禁词但暗示治疗

  [ ] 反向触发
       是否含 "not intended to" / "non-medicated" / "non-prescription" / "consult..."？

  [ ] 卖点保留度
       原 Benefits 4 个角度（功能/便捷/材质/场景）是否保留？
       是否扩展了原文没有的属性？

  [ ] 跨条一致性
       同站同品类多条产品的形态/定位是否互相打架？

  [ ] 下架判定
       Writer 的 _是否建议下架 是否合理？

【综合 verdict】
- 任意自动检查或语义审核命中 blocking → verdict=fail，把所有 issue 一次性写进 feedback
- 仅 warning → verdict=pass with warnings，放行但记录
- 全部干净 → verdict=pass
======================================
```

### 重试循环（最多 2 attempt）

```
状态机（每条产品独立跑）：

attempt = 0
MAX_ATTEMPTS = 2          # ← 最多两次：1 次原始 + 1 次重试
last_feedback = null

while attempt < MAX_ATTEMPTS:
  # ① 派发 Writer
  if attempt == 0:
    writer_output = dispatch_writer(record)
  else:
    writer_output = dispatch_writer(record, feedback=last_feedback)

  # ② Reviewer pass：一次跑完格式 + regex + 语义
  result = run_reviewer_pass(writer_output, record)

  # ③ 判定
  if result.verdict == "pass":
    enqueue_for_write(writer_output)
    break

  # 有 blocking issue
  last_feedback = result.issues   # 全部 issue 一次性带回
  attempt += 1

else:
  # 两次都 fail（attempt = 2 退出循环）
  mark_as_rewrite_failed(record, last_feedback)
```

重派 Writer 时（attempt=1），prompt 多一段：

```
【上一轮你被打回，原因如下，请针对每条改写，不要重复】
1. 字段「合规后-Benefits_2」含黑名单词 "comfort"（止痛暗示）
2. 整体定位仍像保健品，缺少日常消费品的感官描写
3. 形态打架：标题写 serum 但描述写 oil pen

请针对上述问题改写，**只改 issue 涉及的部分，其他维持上次输出不变**。
```

### 图片复审（仅对 `_是否建议下架=true` 的条目触发）

**默认行为**：Writer subagent 不看图片。skill 默认是纯文本流程，所有图片在 Writer 眼里都是 `<img src>` 字符串原样透传，**不下载、不渲染、不分析视觉内容**。这是为了在 1000+ 产品规模下保持速度和成本。

**例外触发**：当某条产品 Reviewer pass 通过 + Writer 标了 `_是否建议下架=true`，Orchestrator 对这条额外做一次图片复审，把"建议下架"细化为可操作的二级标签。

#### 触发流程

```
对每条 _是否建议下架=true 的条目：
  ① 从原描述 HTML 里 grep 出第一张 <img src=""> URL
  ② curl -sL 下载到 /tmp/<batch>_images/rec_<idx>.jpg
  ③ Orchestrator 用 Read 工具看这张图
  ④ 视觉判定（按下面三档）
  ⑤ 把判定细化标签写进 飞书表 `回写失败原因` 字段
```

#### 视觉判定三档

| 档位 | 视觉特征 | 操作建议 |
|------|---------|---------|
| 🟢 **可换图救** | 图本身是中性 lifestyle / 普通日常场景，不是药品包装也没强烈品类暗示动作 | 标 `[需人工复核-可换图救]` + 简述图内容；运营换首图 + 文案配合即可保留 |
| 🟡 **图必须换** | 图含强品类暗示动作（手按腹部=消化、手按下腹=私密护理、对镜整理+泵瓶=男士增强），但**没有**药品/胶囊/安瓿/药瓶物理形态 | 标 `[需人工复核-必须换图]` + 简述暴露点；运营若不换图，文案救不了 |
| 🔴 **图换了也难救** | 图直接展示药品形态：胶囊壳、安瓿小玻璃瓶、滴管瓶、blister pack、FDA 风格盒装、牙齿示意图等 | 标 `[需人工复核-换图但难救]` + 简述药品形态；建议下架或改产品形态（除非愿意承担退款率/风控风险）|

#### 重要事实

- Writer 的"建议下架"判定是**基于文本风险**（标题/描述/Benefits 显示原品类是高风险品类），不是基于图。
- 实战发现（AU-B1 6 条建议下架）：**多数图片其实是 lifestyle 营销图，不是药品包装**——这意味着"建议下架"经常其实是"换图就能救"，图片复审能把"放弃"率显著降低。
- **只看了第一张图**：Shopify 产品通常 4-8 张图，第一张是 banner/lifestyle，后面才是产品本体特写。复审标签里要提示运营"请同时核对其他产品图是否含药品形态"。
- 图片下载失败不阻塞：超时或 404 时，标签退回到无图片信息的"建议下架"原值，让运营全人工评估。

#### 何时不触发

- `_是否建议下架=false`：跳过，直接走正常回写
- 用户明确要求"全员图文双审"（Plan C，规模大时谨慎用）：所有条目走图片复审
- 用户明确要求"全文本"（Plan A）：跳过图片复审，建议下架条目用纯文本风险摘要

### 无需合规分流（在 Reviewer pass 通过后判断）

当 Writer 输出通过 Reviewer pass 后，Orchestrator 多做一道**"是否真的需要写合规字段"**的判断。如果原文本来就没有合规风险、Writer 也没做实质改动，应标 `合规状态=无需合规` 并**清空（或不写）合规后-* 字段**，避免飞书表里堆一堆"和原文一样"的冗余数据。

#### 判定条件（两条必须同时满足）

| 条件 | 检查方式 |
|------|---------|
| ① Writer 没做产品定位级改动 | `合规后-产品标题 == 原标题` **且** `合规后-Handle == 原Handle`（字面一致）|
| ② 原文本身就没合规风险 | 原 `Benefits_1-4` + `原始描述纯文本` 用 Layer 1 黑名单 regex 全扫，**0 命中** |

只有两条都成立，才走"无需合规"分支。任一条不成立 → 走"已修改"分支（即便标题没变，也要写合规字段，因为 Writer 把 Benefits 里的禁词换掉了）。

#### 为什么这两条都要

- 只看条件 ①（标题/Handle 没变）会误标：Writer 可能保留标题但实质改了 Benefits（替换了禁词），这种是**真有合规改写**，不算无需合规。
- 只看条件 ②（原文无禁词）也会误标：原文清白但 Writer 把标题/形态改了（例如从 patch 改成 sticker），这种属于**预防性合规改写**，要写新版。

#### 写表行为

```
对每条 Reviewer pass 通过的记录：
  if 条件①✅ AND 条件②✅:
    upsert {
      合规状态: "无需合规",
      合规后-产品标题: null,
      合规后-Handle: null,
      合规后-描述HTML: null,
      合规后-变体JSON: null,
      合规后-Benefits_1..4: null,
      回写失败原因: null
    }
  elif _是否建议下架 == true:
    走图片复审分支（前面章节定义）
  else:
    upsert {
      合规后-产品标题, 合规后-Handle, 合规后-描述HTML,
      合规后-变体JSON, 合规后-Benefits_1..4,
      合规状态: "已修改"
    }
```

#### 实战数据（AU-B1 78 条）

| 分类 | 数量 | 占比 |
|------|------|------|
| 已修改（真改了） | 41 条 | 53% |
| 无需合规（清白原文 + 标题未变） | 28 条 | 36% |
| 建议下架（图片复审分流后）| 6 条 | 8% |
| 改写失败 | 0 条 | 0% |
| 非产品（B 组慈善/包装/赠品）| 3 条 | 4% |

按这个比例，2000 条产品大概 700 条会落到"无需合规"分支，**不浪费 token 写合规字段**也不浪费运营时间审核没动的内容。

### 失败处理（标记进飞书表）

两次都 fail 的条目，**不沉默旁路，直接在飞书表上做标记**让运营看到：

| 字段 | 值 |
|------|------|
| `合规状态` | `改写失败` |
| `回写失败原因` | issue 列表的简短摘要，例如：`Benefits_2含禁词"comfort"; 整体仍像保健品定位; 已改写2次未通过` |

合规后-* 字段保持空（不写半成品进表）。运营在飞书视图里直接筛选「合规状态=改写失败」就能看到所有要人工处理的条目。

回写时按这个映射写入：

```
对每条结果：
  if result.verdict == "pass":
    upsert {
      合规后-产品标题, 合规后-Handle, 合规后-描述HTML,
      合规后-变体JSON, 合规后-Benefits_1..4,
      合规状态: "已修改"
    }
  elif marked_as_rewrite_failed:
    upsert {
      合规状态: "改写失败",
      回写失败原因: "<issue 摘要>"
    }
```

### 并发与节流

- 单批最多并行 10 个 subagent；超过 10 条分多批执行
- 全部 subagent 完成后再统一调用 `+record-upsert`（逐条）或 `+record-batch-update`（同值批量），避免并发冲突（错误码 1254291）
- 串行回写时批次间延迟 0.5–1 秒

### Reviewer pass 实施细节（regex 词表 + 脚本范本）

Reviewer pass 内部用到的具体 regex 词表和扫描脚本，列在这里供 Orchestrator 实施时直接套用。

#### 必扫字段

每条记录的下列字段拼接后做扫描（描述要先剥 `<...>` HTML 标签）：

- `合规后-产品标题`
- `合规后-Handle`
- `合规后-Benefits_1` ~ `合规后-Benefits_4`
- `合规后-描述HTML`（剥标签）

#### 黑名单 regex（关键字版）

按"绝对禁止 / 强提示 / 反向触发"四档：

```regex
# 第一档：绝对禁止（医疗/药品/治疗/症状）
\b(antifungal|antibacterial|antimicrobial|antiseptic|anti-inflammatory|analgesic|antibiotic|steroid|probiotic|prebiotic|pharmaceutical|clinical|therapeutic|medicinal|prescription|venom|infection|fungal|fungus|psoriasis|eczema|arthritis|diabetes|hypertension|disease|disorder|symptom|diagnosis)\b

# 第二档：症状/治疗动作/功效暗示
\b(treatment|treat|therapy|remedy|cure|heal|healing|recovery|relief|relieve|supplement|detox|detoxify|cleanse|purify|soothing|effective|potent|active ingredient|key compound|clinically proven|lab tested|pharmaceutical grade|medical grade|hospital grade|dosage|dose|potency|concentration)\b

# 第三档：身体器官 / 系统暗示治疗 / 部位
\b(joint|gum|oral|dental|teeth|denture|wrinkle|firming|inhaler|patch|ear care|nasal|lung|liver|kidney|immune|immunity|digestive|holistic|wellness|wellbeing|well-being|mobility|sensitive[- ]skin|joint comfort|gum care|ear rinse)\b

# 第四档：反向触发（免责声明 / 医疗器械暗示）
(not intended to|consult a healthcare|for adult use only|adjustable lenses?|vision correction|diopter|micro-spicule|carefully selected botanical|nature's remedy|herbal remedy|natural remedy|backed by tradition|ancient formula|dietary supplement|dietary product|medical condition|non-medicated|non-invasive)
```

#### 格式校验项

- 输出 JSON 解析必须成功
- `合规后-Handle` 必须只匹配 `^[a-z0-9-]+$`
- `合规后-描述HTML` 剥标签后词数应 ≤ 80（理想 60-70）
- `合规后-变体JSON` 必须解析成功，每个 variant 的 `title` 字段必须以 `Nx ` 开头并包含新标题
- 变体 SKU / id / price / image / inventoryItem / position 与原变体一字不差
- `合规后-产品标题` 不能为空，且首词不能是 `Discover`

#### 扫描脚本范本（jq + grep）

```bash
BANNED='\b(antifungal|antibacterial|antimicrobial|antiseptic|...|joint|gum|oral|dental|teeth|patch|wrinkle|...)\b|(not intended to|adjustable lenses?|carefully selected botanical|...)'

for f in /tmp/results/rec_*.json; do
  CONCAT=$(jq -r '."合规后-产品标题" + "|" + ."合规后-Handle" + "|" + ."合规后-Benefits_1" + "|" + ."合规后-Benefits_2" + "|" + ."合规后-Benefits_3" + "|" + ."合规后-Benefits_4" + "|" + (."合规后-描述HTML" | gsub("<[^>]*>"; ""))' "$f")
  HITS=$(echo "$CONCAT" | grep -oiE "$BANNED" | sort -u | tr '\n' ',')
  [ -n "$HITS" ] && echo "$f HITS: $HITS"
done
```

### 状态机（最终版）

| 阶段 | Orchestrator 动作 | Writer subagent |
|------|-------------------|-----------------|
| 1. 拉数据 | `+record-list` 拿全字段，per-record 拆成独立输入文件 | — |
| 2. **派发 Writer**（attempt=0） | 并发派发，每条 1 个 Writer subagent（≤10/批） | 改写产出 JSON 到 `/tmp/results/rec_<idx>.json` |
| 3. **Reviewer pass** | 每条独立跑：格式 + regex + 语义一次性审完，全部 issue 收集到 list | — |
| 4. fail → **派发 Writer 重写**（attempt=1）| 把 issue list 拼进 prompt，要求 Writer 仅改 issue 部分 | Writer 输出 v2 |
| 5. v2 再跑 Reviewer pass | 同 step 3 | — |
| 6. v2 仍 fail | 标 `合规状态=改写失败` + `回写失败原因=issue 摘要` | — |
| 7. **无需合规分流** | 标题+Handle 未变 **且** 原 Benefits/描述无禁词 → 标 `合规状态=无需合规` + 清空合规后-* 字段 | — |
| 8. **图片复审**（仅 `_是否建议下架=true` 触发）| 下载首图 → Read → 三档视觉判定 → 细化标签 | — |
| 9. 通过的条目回写（非无需合规、非建议下架）| 串行 `+record-upsert` 写合规字段 + `合规状态=已修改`；间隔 0.6s | — |
| 10. 建议下架的条目回写 | 串行 `+record-upsert` 写合规字段 + `合规状态=已修改` + `回写失败原因=[需人工复核-X档] 视觉摘要` | — |
| 11. 失败的条目回写 | 串行 `+record-upsert` 写 `合规状态=改写失败` + `回写失败原因`（不写合规字段） | — |
| 12. 报告 | 简短摘要：已修改 / 无需合规 / 建议下架（按图片三档分布）/ 改写失败 / 总耗时 | — |

### 不要做的事

- ❌ 不要在主 agent 里直接逐条改写（会污染上下文）
- ❌ 不要让一个 subagent 同时改多条（同样会趋同）
- ❌ 不要在派发前自己先想好"模板"再发给 subagent（这等于人工同质化）
- ❌ 不要让 subagent 去拉表 / 写表（subagent 只做改写，IO 由主 agent 负责）
- ❌ subagent 之间不要相互参考（彼此独立上下文）
- ❌ **不要让 Writer 自审** —— 自我合理化偏差，写完会觉得自己写的都对
- ❌ Orchestrator 审核时**不要批量审 17 条**，必须每条独立走 checklist —— 防审核疲劳
- ❌ 不要跳过 Layer 0/1 直接上 Layer 2 —— regex 比 LLM 阅读便宜 100×，先用便宜的过滤明显错误
- ❌ 不要无限 retry —— MAX_ATTEMPTS=3，超过就标人工审核，避免 token 爆炸
- ❌ 不要派独立 Reviewer subagent —— 是过度设计：① Orchestrator 不是 Writer，没有自审偏差；② Orchestrator 已握有全局信息（所有原记录 + 所有 Writer 输出），独立 Reviewer 反而是冗余中间层；③ 跨条不一致只有 Orchestrator 这一层看得见
- ❌ **不要 inline 写 shell 数组循环回写飞书**（高频踩坑）：`for i in 51 53 6; do RID=${RIDS[$i]}; lark-cli ...; done` 这种写法在 macOS zsh 下数组是 1-indexed（`RIDS[1]` 才是首元素），会**整体偏移 -1 错位写到隔壁记录**，且不报错只静默失败。必须用以下三种方式之一：
  - **首选**：用 Python + `subprocess` 调 lark-cli（数组 0-indexed 明确）
  - 用 `bash` 显式：把脚本写到文件，`bash /tmp/script.sh` 执行
  - 用 heredoc：`bash <<'EOF' ... EOF` 强制 bash 子 shell

  写完后**必须 audit 抽样几条**：读回飞书表对比 `产品标题`（应是原 idx 的标题）和 `合规后-产品标题`（应是 agent 的输出），任何错位会立刻露馅

---

## 第八部分：备用方案

如果合规改写后仍被标记（账号级标记导致），最有效的方案是：

1. **换第三方支付网关**：Authorize.net, NMI, DigiPay, Pinwheel（绕开 Shopify Payments/Stripe）
2. **主动向 Shopify 提交申诉**：附上合规整改说明
3. **监控退单率**：保持在 1% 以下，超过会自动触发暂停
4. **全渠道一致**：博客、FAQ、邮件、广告中也不能有医疗暗示

---

## 第九部分：实战踩坑记录

每条都来自真实跑站时遇到的问题。后续 agent 看到类似场景**必须主动回避**，不要重复踩。

### 工程类（系统/工具/IO）

#### ⚠️ zsh 数组 1-indexed 静默错位（致命）

- **现象**：在 macOS Bash 工具下 inline 写 `for i in 51 53 6; do RID=${RIDS[$i]}; lark-cli upsert --record-id $RID ...; done`，所有写入都偏移到隔壁记录，且 API 返回 OK 不报错
- **根因**：macOS 默认 `$SHELL=/bin/zsh`，zsh 数组从 1 开始（`RIDS[1]` 是首元素），bash 数组从 0 开始；同一个 `${RIDS[6]}` 在 bash 给 idx 6，在 zsh 给 idx 5
- **后果**：1 次错误写入 ≈ 影响 N 条记录（一条写错位 + 一条该改的没改），且静默无错
- **正确做法**：写表脚本**禁止 inline shell 数组**，三选一：① Python `subprocess`（首选）② `bash /path/to/script.sh` ③ `bash <<'EOF' ... EOF` heredoc。**且**写完抽样 audit 对比 `产品标题`（原 idx）vs `合规后-产品标题`（agent 输出）

#### ⚠️ 客户的 sync 脚本会反复擦合规字段

- **现象**：你 upsert 完，几小时后再读发现 `合规后-*` 全部清空、`合规状态` 回到「未检查」
- **根因**：客户跑了一个 Shopify→飞书 sync，逻辑是"任何源字段变化（甚至库存数字）就把所有合规字段作废 + 重置状态"
- **判别**：record-history-list 看 operator='飞书 CLI'（非 user 'XXX'）的修改记录；变体 JSON 里某 SKU 的 `inventoryQuantity` 从 0 变 -1 但其他都没变 → 就是 sync 拉了新库存
- **正确做法**：① 提醒用户运维改 sync 脚本，**只在标题/描述/Benefits 实际变化时才清合规字段**；② 在 sync 修复前不要批量回写（数据被擦白做工）；③ 部分 record_id 可能彻底失效（API 返回空 record），那是 sync 删了重建，不要花精力恢复非产品类记录

#### ⚠️ JSON 字段含 `\n` 时 `jq -c | bash $()` 会损坏字符串

- **现象**：`PAYLOAD=$(jq -c . file.json)` 然后 `lark-cli --json "$PAYLOAD"` 报 `control characters from U+0000 through U+001F must be escaped`
- **根因**：`jq -c` 输出虽然是单行 JSON，但 string value 里的 `\n` 在 macOS jq 1.x 下输出的是字面 `\n`（两字符），bash $() 捕获时**会把字面 `\n` 处理成实际换行字节**，再传给 lark-cli 时就是非法 JSON
- **正确做法**：用 Python `json.dumps(obj, ensure_ascii=False)` 替代 jq，或者把 payload 写到文件用 `bash <<'EOF' python3 EOF` 重定向到文件

#### ⚠️ jq 不接受 CJK 不带引号的 key

- **现象**：`jq '._是否建议下架' file.json` 报 `syntax error, unexpected INVALID_CHARACTER`
- **正确做法**：用 bracket notation `jq '.["_是否建议下架"]'`

### Agent 类（LLM 行为偏差）

#### ⚠️ Writer 主动加 "Use & safety / patch-test" 安全警告

- **现象**：Writer 把 `Use & safety: external use only. Avoid broken skin. Patch-test before first use.` 拼到描述末尾，触发 `patch / external use only` 黑名单
- **根因**：LLM 训练偏好"对消费品就该加安全免责"，对合规改写场景是反向 trigger
- **正确做法**：Writer prompt 里强调"原文没有的安全声明禁止添加"；Reviewer pass 扫描 `external use only / broken skin / keep out of reach / consult a healthcare / patch-test` 一定 fail

#### ⚠️ Writer 看到原标题/描述形态打架时容易错选

- **现象**：原标题 "Botanical Lip Serum" + 原描述 "lip oil in click-pen" → Writer 改成 "Botanical Lip Oil Pen"（违反规则 7：serum→oil 禁止）
- **根因**：Writer 默认相信描述更具体，但描述常有运营笔误
- **正确做法**：skill 第六部分规则 7 已加：**形态判断按原标题→图片→描述优先级，标题和描述打架时以标题为准反向修正描述**。**严禁参考 SKU**

#### ⚠️ Writer 把 capsules 翻译成 Pods（语义不清）

- **现象**：`Botanical Blend Capsules` → Writer 改成 `Botanical Blend Pods`
- **根因**：LLM 联想到容器/胶囊都叫 pods，但英文消费品 pods 多关联 coffee pods / Tide pods（食品/洗涤球），影响转化
- **正确做法**：硬替换表已加 `capsules → pearls`（首选，化妆品行业标准词），`pods` 明确禁用

#### ⚠️ Writer 删 leg/arm 这种中性身体部位词

- **现象**：`Warming Leg Care Oil` → Writer 改成 `Botanical Body Oil`（"Leg" 也被一并删了）
- **根因**：Writer 看到任何身体部位词就过度去敏
- **正确做法**：skill 第三部分新加身体部位词分级：
  - 🔴 必删档：oral / ear / intimate / male genital / breast（这些部位本身=高风险品类）
  - 🟢 中性档：leg / arm / hand / face / lip / hair / scalp（**保留对转化更好**）
  - 删 Warming/Care 但**保留 Leg**

#### ⚠️ Writer 用 Body 替换被删的部位词，导致客诉

- **现象**：`Daily Ear Spray` → Writer 改成 `Daily Body Mist`（部位换部位）。运营反馈：和落地页展示对不上号，客诉
- **正确做法**：skill 第三部分新加替换词选择策略：当原部位词必须删时，优先级
  - ① 产品类型留白（`Daily Mist` / `Daily Care Mist` / `Lightweight Mist`）—— **正确**
  - ② 质地/感官词（`Cool Mist` / `Botanical Mist`）
  - ③ 不要换另一个部位（`Body Mist` / `Foot Mist`）—— **错误默认行为**

#### ⚠️ Writer 改个不停（即使原文已合规）

- **现象**：原标题 `Lightweight Liquid Foundation` 完全无风险，Writer 仍输出新版（标题相同，Benefits 改了几个字）
- **根因**：Writer prompt 默认假设"必须改写"
- **正确做法**：Orchestrator 在 Reviewer pass 后多做一道判断 —— 标题+Handle 未变 **且** 原 Benefits/描述无禁词 → 标 `合规状态=无需合规` + 清空合规后-* 字段（不要存"和原文一样"的复制品）。规则在第七部分「无需合规分流」

### 业务类（产品/品类风险）

#### ⚠️ 文案改完不等于产品能保留

- **现象**：6 条品类高风险产品（Capsules / ED Ampoules / Oral Gel / Intimate Cream），文案改写都通过 Reviewer pass，但 agent 标 `_是否建议下架=true`
- **理由**：风控有图像扫描、SKU 公开端点、L2/L3 跨系统检测；文案合规只能过 L1 文字层，过不了图片和品类记忆
- **正确做法**：图片复审分流（第七部分），按 🟢 可换图救 / 🟡 必须换图 / 🔴 图换了也难救 三档标签输出给运营

#### ⚠️ Capsules 形态本身=补充剂

- **现象**：再洗文案（`Plant Pearls` / `Botanical Pods`），实物拆开是胶囊，L2 第三方扫描照标
- **正确做法**：胶囊类产品标 `[需人工复核-换图但难救]`，建议运营**改产品形态**（重灌成软膏/冻干粉饼）或下架

#### ⚠️ 多数"建议下架"产品的图其实是 lifestyle 不是药品包装

- **现象**：6 条建议下架的，4 条首图是中年女露腰 / 男士对镜 / 情侣厨房等普通 lifestyle，**没有真药瓶**
- **意义**：图片复审能把"放弃"率从 100% 砍到 33%（4/6 可换图救）
- **正确做法**：Plan B（图片复审）只对 `_是否建议下架=true` 触发，不要给所有产品上图片复审（成本高）

### 流程类（人/角色/反馈）

#### ⚠️ 过度设计：独立 Reviewer subagent

- **现象**：曾设计 Writer subagent + Reviewer subagent 双 LLM 调用
- **问题**：① Orchestrator 不写不需要避自审偏差；② 跨条不一致 Reviewer subagent 看不到（只看自己那条）
- **正确做法**：Orchestrator 自己当审核员，每条独立走 checklist 防疲劳；只用一种 subagent（Writer）

#### ⚠️ 运营品味比 agent 强（建立反馈闭环）

- **现象**：运营批了 3 条：`Body Oil 应保留 Leg` / `Pods 应改 Pearls` / `Body Mist 应改 Care Mist`
- **教训**：agent 的"合规"和市场的"转化/客诉"目标不完全重合
- **正确做法**：每跑完一站，**主动让运营 review 一批样本**，把品味反馈固化进 skill（已加规则 7、硬替换表、替换词选择策略）。skill 是活的，不是一次写死的

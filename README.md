# mail-postage-data

中国邮政邮费结构化数据仓库，面向 App / Web / 后端程序读取。

当前数据覆盖三个独立资费范围：**中国大陆境内**、**中国大陆寄往香港/澳门/台湾**、**中国大陆寄往国际目的地**。基础寄递资费与特别业务资费分离维护。

> 本仓库不是中国邮政官方发布渠道。资费、可寄达性、禁限寄要求和临时停运信息，请以中国邮政及目的地邮政的最新规定为准。资费信息自己整理而成，不保证内容最新，如有错误欢迎提交 issue。

## 目录结构

```text
mail-postage-data/
├── README.md
├── countryData.json
└── postage/
    ├── index.json
    ├── services.json
    ├── notices.json
    ├── domestic/
    │   └── provinces.json
    ├── spec/
    │   ├── index.json
    │   ├── core.json
    │   ├── pricing.json
    │   ├── notices.json
    │   └── validation.json
    ├── rates/
    │   ├── domestic/
    │   │   ├── basic.json
    │   │   ├── ordinary-parcel.json
    │   │   └── hometown-parcel-sticker.json
    │   ├── hong-kong-macau-taiwan/
    │   │   └── basic.json
    │   └── international/
    │       ├── air-letter.json
    │       ├── air-other-mail.json
    │       ├── sal-letter.json
    │       ├── sal-other-mail.json
    │       ├── surface-letter.json
    │       └── surface-other-mail.json
    ├── zones/
    │   ├── air-letter.json
    │   ├── air-other-mail.json
    │   ├── sal-letter.json
    │   ├── sal-other-mail.json
    │   ├── surface-letter.json
    │   └── surface-other-mail.json
    └── special-rates/
        ├── index.json
        ├── notices.json
        ├── domestic.json
        ├── hong-kong-macau-taiwan.json
        └── international.json
```

## 规范入口

App 或维护脚本应首先读取 `postage/index.json`，其中 `files.specification` 指向 `postage/spec/index.json`。

当前规范版本：

```text
standardVersion: 1.1.0
```

规范模块：

- `spec/core.json`：scope、rateCategory、service、mailType、省级路由、路径和索引规则。
- `spec/pricing.json`：所有支持的计价类型、字段和计算公式。
- `spec/notices.json`：通用 notice 与特别业务 notice 的匹配和优先级规则。
- `spec/validation.json`：App / CI 应执行的跨文件一致性校验。

## Scope

统一只使用：

```text
domestic
hong-kong-macau-taiwan
international
```

国际数据排除 `CN`、`HK`、`MO`、`TW`；港澳台单独使用 `HK / MO / TW`；中国大陆境内使用 domestic scope。

## 资费分类

所有资费分为：

```text
delivery
special
```

`delivery` 是实际寄件方式/基础寄递服务，必须注册在 `postage/services.json`。

`special` 是可在寄递资费之外叠加的特别业务，例如挂号、回执、保价、约投、存局候领和验关等，统一放在 `postage/special-rates/`。

**家乡包裹贴属于 delivery service，不属于 special-rates。**

## 国内省级行政区

存在省际差异的国内资费统一使用：

```text
postage/domestic/provinces.json
```

当前整理中国大陆 31 个省级行政区，不包含香港、澳门、台湾。

省级 ID 使用 ISO 3166-2:CN 风格，例如：

```text
CN-SC  四川
CN-BJ  北京
CN-XJ  新疆
```

同时保存 `gb2260Prefix` 方便与国内行政区数据对接。

存在省际差异时使用：

```text
originRegionId + destinationRegionId
```

不得拿国际 `destination` / zone 机制代替国内省级路由。

## 国内函件

基础寄递资费：

```text
postage/rates/domestic/basic.json
```

包括信函、明信片、印刷品、邮简、义务兵免费信函、盲人读物。

特别业务：

```text
postage/special-rates/domestic.json
```

包括挂号、约投、回执、保价、存局候领、撤回或修改收件人名址等。

## 邮政普通包裹

服务：

```text
DOMESTIC_ORDINARY_PARCEL
```

资费：

```text
postage/rates/domestic/ordinary-parcel.json
```

普通包裹按**起寄省 + 寄达省 + 总重量档**确定资费。

重量档固定为：

```text
≤10kg
>10kg 且 ≤20kg
>20kg 且 ≤50kg
```

每个重量档均提供：

```text
首重 1kg
每续重 1kg
```

计算时必须先根据**整件包裹总重量**选择唯一重量档，然后从该档对应的起寄/寄达路向读取 `basePrice` 和 `incrementPrice`。三个重量档之间**不累计**。

### 当前已录入起寄省

目前只录入：

```text
CN-SC 四川
```

四川起寄已覆盖 31 个中国大陆省级寄达地区。

其他起寄省列在 `unpopulatedOrigins` 中，仅表示结构已预留，没有资费。App 遇到未录入起寄省必须返回“暂无资费”，**不得套用四川价格**。

## 家乡包裹贴

服务：

```text
DOMESTIC_HOMETOWN_PARCEL_STICKER
```

资费：

```text
postage/rates/domestic/hometown-parcel-sticker.json
```

这是独立寄件方式，`rateCategory=delivery`，资费不区分地区。

当前重量固定价：

```text
<1kg   4元/件
<3kg   6元/件
<5kg   11元/件
<10kg  19元/件
```

重量边界按原始录入的严格“小于”规则保存，因此 10kg 及以上当前没有家乡包裹贴资费。

## 港澳台资费

基础寄递资费：

```text
postage/rates/hong-kong-macau-taiwan/basic.json
```

适用于 `HK / MO / TW`。

特别业务：

```text
postage/special-rates/hong-kong-macau-taiwan.json
```

港澳台存在目的地级例外，因此该 scope 的 `destinationIndependent=false`。

例如寄往台湾的附回执函件暂不收寄，真实限制保存在：

```text
postage/special-rates/notices.json
```

## 国际资费

国际基础寄递按 `AIR / SAL / SURFACE` 维护于：

```text
postage/rates/international/
```

需要目的地分区的服务通过 `postage/zones/` 解析 zoneId。

国际特别业务位于：

```text
postage/special-rates/international.json
```

国际特别业务表中的函件、小包、印刷品专袋和包裹相关费用应完整保留，适用范围通过 `applicableCategories`、`applicableMailTypes`、`conditions` 等字段表达。

## Pricing 类型

完整定义见 `postage/spec/pricing.json`。

当前包括：

```text
flat
flat_by_locality
free
base_plus_increment
surcharge_per_increment
air_surcharge_only
weight_tiers
tiered_incremental
weight_band_base_plus_increment_by_route
value_increment
value_increment_with_minimum
external_rule
```

其中 `weight_band_base_plus_increment_by_route` 专门用于普通包裹这类“先按总重量选档，再按起寄地/寄达地首重+续重”的资费。

`weight_tiers` 支持：

```text
minWeight
minWeightExclusive
maxWeight
maxWeightExclusive
```

用于准确表达 `<=`、`<`、`>=`、`>` 边界。

凡包含“不足计费单位按一个计费单位”或“零数”规则时，必须显式使用：

```json
"rounding": "ceil"
```

## App 推荐读取顺序

1. 读取 `postage/index.json`。
2. 读取 `postage/spec/index.json` 并确认支持 `standardVersion`。
3. 读取 `services.json` 并选择 serviceId。
4. 如果 service 存在 `regionRegistry`，先加载省级行政区注册表。
5. 按 service 的 `rateFile` / `zoneFile` 加载基础寄递资费。
6. 普通包裹使用 `originRegionId + destinationRegionId + weight` 查价。
7. 根据 scope 加载 `special-rates/index.json` 和对应特别业务资费。
8. 加载通用 notices 和 special-rate notices。
9. 按 `spec/validation.json` 校验；error 级校验失败时不得静默返回错误邮资。

## 数据维护原则

1. 新数据先确认属于 `delivery` 还是 `special`。
2. 新寄件方式必须在 `services.json` 注册，并先注册 mailType。
3. 国内存在地区差异时优先使用统一 region registry，不在各资费文件重复定义省名。
4. 未录入的起寄省必须显式保持为空/placeholder，不得复制其他省资费兜底。
5. 新 pricing 类型必须先加入 `spec/pricing.json`。
6. 新 notice 字段必须同步更新 notice fieldGuide / spec。
7. 所有新增或修改数据都应通过 `spec/validation.json` 的跨文件校验。
8. 国际、港澳台、国内三个 scope 互不混用。
9. `schemaVersion` 表示单文件结构版本；`standardVersion` 表示整体数据规范版本。

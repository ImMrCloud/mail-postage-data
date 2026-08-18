# mail-postage-data

中国邮政邮费结构化数据仓库，面向 App / Web / 后端程序读取。

当前数据覆盖三个独立资费范围：**中国大陆境内**、**中国大陆寄往香港/澳门/台湾**、**中国大陆寄往国际目的地**。基础寄递资费与特别业务资费分离维护。

> 本仓库不是中国邮政官方发布渠道。资费、可寄达性、禁限寄要求和临时停运信息，请以中国邮政及目的地邮政的最新规定为准。资费信息自行整理，不保证内容最新，如有错误欢迎提交 issue。

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
    │   ├── special-services.json
    │   └── validation.json
    ├── rates/
    │   ├── domestic/
    │   │   ├── basic.json
    │   │   ├── hometown-parcel-sticker.json
    │   │   └── ordinary-parcel/
    │   │       ├── index.json
    │   │       ├── CN-SC.json
    │   │       ├── CN-ZJ.json
    │   │       ├── CN-JS.json
    │   │       ├── CN-GD.json
    │   │       ├── CN-HE.json
    │   │       ├── CN-SD.json
    │   │       ├── CN-BJ.json
    │   │       └── CN-JX.json
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
standardVersion = 1.4.0
```

规范分为：

- `spec/core.json`：scope、service、mailType、省级路由和服务能力规则。
- `spec/pricing.json`：支持的计价类型、体积重、折扣顺序和计算公式。
- `spec/notices.json`：通用 notice 与特别业务 notice 的匹配规则。
- `spec/special-services.json`：特别业务可选性、适用服务、依赖、包含和互斥关系。
- `spec/validation.json`：App / CI 应执行的跨文件一致性校验。

## Scope 与资费分类

统一 scope：

```text
domestic
hong-kong-macau-taiwan
international
```

统一资费分类：

```text
delivery
special
```

`delivery` 是基础寄递方式；`special` 是挂号、回执、保价等特别业务资费。

## 特别业务选择语义

特别业务使用机器可读字段区分寄件时可选项目和只公示项目：

```text
selection.mode = selectable | display-only
selection.phase = acceptance | post-acceptance | delivery | inbound | other
selection.chargedTo = sender | recipient | customer
```

依赖关系：

```text
dependencies.requiresRateIds
dependencies.includesRateIds
dependencies.excludesRateIds
```

适用对象可使用：

```text
applicableCategories
applicableMailTypes
applicableServiceIds
excludedServiceIds
```

`applicableCategories`、`applicableMailTypes`、`applicableServiceIds` 按 OR 判断；`excludedServiceIds` 优先排除。

存局候领、撤回/修改收件人名址、进口欠资、逾期保管、海关验关等非交寄时费用使用 `display-only`，只公示，不加入寄件总价。存局候领费用由收件人承担。

## 国内省级行政区

中国大陆 31 个省级行政区统一维护在 `postage/domestic/provinces.json`，使用 ISO 3166-2:CN 风格 ID。省际差异资费通过：

```text
originRegionId + destinationRegionId
```

匹配。香港、澳门、台湾不放入此注册表。

## 邮政普通包裹

服务 ID：

```text
DOMESTIC_ORDINARY_PARCEL
```

服务入口：

```text
postage/rates/domestic/ordinary-parcel/index.json
```

普通包裹三个总重量档：

```text
<=10kg
>10kg 且 <=20kg
>20kg 且 <=50kg
```

每一档按该档对应的 **首重 1kg + 每续重 1kg** 计算。先按整件总重量选择唯一重量档，三个档位之间不累计。

当前已录入起寄省/市：

```text
CN-SC 四川
CN-ZJ 浙江
CN-JS 江苏
CN-GD 广东
CN-HE 河北
CN-SD 山东
CN-BJ 北京
CN-JX 江西
```

其他起寄省仍位于 `unpopulatedOrigins`，没有价格时必须返回“暂无资费数据”，不得套用其他省份价格。

### 普通包裹体积重

长、宽、高均为可选输入，单位 cm：

```text
volumetricWeightKg = lengthCm * widthCm * heightCm / 6000
```

尺寸完整时必须分别计算：

```text
actualWeightPostage
volumetricWeightPostage
```

并以**资费较高者**作为基础寄递资费。消费者应同时展示两项结果，而不是只比较两个重量数值后计算一次。

### 普通包裹挂号、回执与保价

普通包裹基础寄递资费本身已包含挂号属性：

```text
serviceCapabilities.registrationIncludedInDeliveryPostage = true
```

因此普通包裹附回执时：

- 可以办理回执；
- 不需要额外选择挂号；
- 不另收 `REGISTRATION_FEE`。

普通包裹可以办理保价，也可以同时办理回执，二者不冲突。

当前数据**没有提供国内普通包裹保价费率**，因此只记录“可保价”能力；消费者不得把国内给据信函保价费率套用于普通包裹。

### 普通包裹优惠凭证

8 折：学生证、教师证、优惠卡。

7 折：残疾证、军官证、警官证、文职人员证、士兵证、退役军人优待证。

规则：

```text
combinationPolicy = single
maxCredentialsPerShipment = 1
applyTo = baseDeliveryPostageOnly
```

即每件最多使用一种优惠凭证，不允许叠加。**8折/7折只作用于普通包裹基础寄递资费**；回执费、保价费及其他 `special` 费用按原价另加，不参与折扣。

推荐计算顺序：

1. 计算实重基础资费。
2. 尺寸完整时计算泡重基础资费。
3. 取两项基础资费中的较高者。
4. 对该基础资费应用一个允许的优惠凭证。
5. 再按原价加入回执、保价等特别业务费。

## 家乡包裹贴

服务 ID：

```text
DOMESTIC_HOMETOWN_PARCEL_STICKER
```

重量档：

```text
<=1kg                 4元/件
>1kg 且 <=3kg         6元/件
>3kg 且 <=5kg        11元/件
>5kg 且 <=10kg       19元/件
```

家乡包裹贴：

- 可办理保价；
- **不可办理回执**；
- 当前保价费率未提供，只公示保价能力；
- 不使用普通包裹优惠；
- 不使用普通包裹体积重规则。

## 国内函件特别业务

基础资费：`postage/rates/domestic/basic.json`。

特别业务：`postage/special-rates/domestic.json`。

主要规则：

- 国内函件回执必须挂号；
- 国内保价给据信函必须挂号；
- 国内约投服务费只对 `LETTER` 生效；
- 约投与保价给据信函互斥；
- 存局候领由收件人承担，只公示；
- 撤回邮件或更改收件人名址属于交寄后申请，只公示。

## 港澳台与国际特别业务

国际和港澳台保价函件保价费要求同时使用 `INSURED_LETTER_HANDLING_FEE`。该手续费已经包含挂号费，因此不得重复计收 `REGISTRATION_FEE`。

撤回/改址、进口欠资、存局候领、逾期保管、海关验关等费用只公示，不作为寄件时可选项目。

寄往台湾的附回执函件暂不收寄，真实限制记录在 `postage/special-rates/notices.json`，消费者必须执行该 notice。

## Pricing 类型

完整定义见 `postage/spec/pricing.json`。当前包含：

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
weight_band_base_plus_increment_by_route_group
multiplier
value_increment
value_increment_with_minimum
external_rule
```

## App 推荐读取顺序

1. 读取 `postage/index.json`。
2. 读取 `postage/spec/index.json`，确认支持 `standardVersion`。
3. 读取 `services.json` 选择基础寄递 service。
4. 按 service 加载行政区、zone 和基础 rate 文件。
5. 普通包裹尺寸完整时同时计算实重和泡重基础资费，取资费较高者。
6. 普通包裹如选择优惠，只对基础寄递资费应用一个 multiplier。
7. 加载 `special-rates`，只把 `selection.mode=selectable` 且适用于当前 service 的项目作为寄件选项。
8. 递归解析 `requiresRateIds`，去除 `includesRateIds` 已包含的重复费用，并检查 `excludesRateIds`。
9. 若基础 service 自身已满足某依赖能力（如普通包裹已含挂号），不得重复收费。
10. `display-only` 项目只公示，不进入寄件总价。
11. 加载 notices 并执行限制。
12. 按 `spec/validation.json` 校验；error 级问题不得静默忽略。

## 数据维护原则

1. 新数据先确认属于 `delivery` 还是 `special`。
2. 新 service 必须在 `services.json` 注册。
3. 新增普通包裹起寄省时，新建 `ordinary-parcel/CN-XX.json` 并在 `originFiles` 注册。
4. 一个起寄省文件内同一寄达省只能出现在一个 `routeGroup`。
5. 未录入起寄省不得回退到其他省价格。
6. 新特别业务必须明确可选性与适用对象；非交寄费用应使用 `display-only`。
7. 普通包裹优惠只能作用于基础寄递资费，不得折扣特别业务费用。
8. 新 pricing 类型必须先写入 `spec/pricing.json`。
9. 新增或修改数据后按 `spec/validation.json` 做跨文件校验。
10. `schemaVersion` 表示单个文件结构版本；`standardVersion` 表示整体规范版本。

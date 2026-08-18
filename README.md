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
    │   │       └── CN-BJ.json
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
standardVersion = 1.2.0
```

规范分为：

- `spec/core.json`：scope、rateCategory、service、mailType、路径、省级路由和索引规则。
- `spec/pricing.json`：所有支持的计价类型、字段、计算公式和折扣模型。
- `spec/notices.json`：通用 notice 与特别业务 notice 的匹配和优先级规则。
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

`delivery` 是基础寄递方式；`special` 是挂号、回执、保价等特别业务附加资费。

## 国内省级行政区

中国大陆 31 个省级行政区统一维护在：

```text
postage/domestic/provinces.json
```

使用 ISO 3166-2:CN 风格 ID，例如：

```text
CN-SC 四川
CN-ZJ 浙江
CN-JS 江苏
CN-GD 广东
CN-HE 河北
CN-SD 山东
CN-BJ 北京
```

省际差异资费统一通过：

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

普通包裹有三个总重量档：

```text
<=10kg
>10kg 且 <=20kg
>20kg 且 <=50kg
```

每一档均按该档对应的 **首重 1kg + 每续重 1kg** 价格计算。先按整件总重量选择唯一重量档，三个档位之间不累计。

### 分省文件

每个已录入起寄省/市单独一个文件。当前已录入：

```text
CN-SC 四川
CN-ZJ 浙江
CN-JS 江苏
CN-GD 广东
CN-HE 河北
CN-SD 山东
CN-BJ 北京
```

每个文件使用 `routeGroups[]` 聚合同价寄达省。消费者流程：

1. 根据 `originRegionId` 从 `originFiles` 找到分省文件。
2. 根据 `destinationRegionId` 匹配唯一 `routeGroup`。
3. 根据总重量匹配唯一 `weightBand`。
4. 读取该组该重量档的 `basePrice` / `incrementPrice` 计算。

其他起寄省仍位于 `unpopulatedOrigins`，没有价格时必须返回“暂无资费数据”，不得套用四川或其他省份价格。

### 普通包裹优惠凭证

优惠规则仅保存在普通包裹 `index.json` 中，**只适用于 `DOMESTIC_ORDINARY_PARCEL`**。

当前：

```text
优惠卡：8折，multiplier = 0.8
残疾证：7折，multiplier = 0.7
学生证：已预留，具体折扣比例待补
```

优惠在普通包裹基础寄递资费计算完成后应用。

当 `combinationPolicy = unspecified` 时，App 不得自动把多个优惠凭证叠加使用。`status = rate-pending` 或 `pricing = null` 的凭证不得参与价格计算。

## 家乡包裹贴

家乡包裹贴是独立的基础寄递方式，而不是特别业务：

```text
DOMESTIC_HOMETOWN_PARCEL_STICKER
```

资费文件：

```text
postage/rates/domestic/hometown-parcel-sticker.json
```

不区分地区，重量档为：

```text
<=1kg                 4元/件
>1kg 且 <=3kg         6元/件
>3kg 且 <=5kg        11元/件
>5kg 且 <=10kg       19元/件
```

10kg 以上当前没有资费数据。

普通包裹的优惠卡、残疾证等折扣不得自动应用到家乡包裹贴。

## 国内函件

基础资费：`postage/rates/domestic/basic.json`。

目前包括信函、明信片、印刷品、邮简、义务兵免费信函、盲人读物。

特别业务：`postage/special-rates/domestic.json`。

## 港澳台

基础资费：`postage/rates/hong-kong-macau-taiwan/basic.json`。

特别业务：`postage/special-rates/hong-kong-macau-taiwan.json`。

寄往台湾的附回执函件暂不收寄，真实限制记录在 `postage/special-rates/notices.json`，消费者必须执行该 notice。

## 国际资费

国际基础寄递按 AIR / SAL / SURFACE 分开维护在 `postage/rates/international/`，需要目的地分区的服务通过 `postage/zones/` 解析。

国际国家/地区优先使用 ISO 3166-1 alpha-2；特殊邮政路向可使用 `countryData.json` 中的自定义 locale，例如 `PT-AZORES`。

国际特别业务统一位于 `postage/special-rates/international.json`。

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

`weight_band_base_plus_increment_by_route` 仅保留旧版兼容语义；新普通包裹数据使用 `weight_band_base_plus_increment_by_route_group`。

## App 推荐读取顺序

1. 读取 `postage/index.json`。
2. 读取 `postage/spec/index.json`，确认支持 `standardVersion`。
3. 读取 `services.json` 选择 service。
4. 若 service 有 `regionRegistry`，先解析国内省级行政区。
5. 对普通包裹，加载 `ordinary-parcel/index.json` 后再加载对应起寄省文件。
6. 加载基础寄递资费并计算。
7. 若普通包裹选择有效优惠凭证，再按 multiplier 应用折扣。
8. 根据 scope 加载 `special-rates`。
9. 加载通用 notices 与 special-rates notices。
10. 按 `spec/validation.json` 校验；出现 error 级问题时不得静默计算错误邮资。

## 数据维护原则

1. 新数据先确认属于 `delivery` 还是 `special`。
2. 新 service 必须在 `services.json` 注册，mailType 先加入 `mailTypes`。
3. 新增普通包裹起寄省时，新建 `ordinary-parcel/CN-XX.json` 并在 `originFiles` 注册。
4. 一个起寄省文件内同一寄达省只能出现在一个 `routeGroup`。
5. 未录入起寄省不得回退到其他省价格。
6. 新 discount 必须声明适用 service；未知折扣值不得猜测。
7. 新 pricing 类型必须先写入 `spec/pricing.json`。
8. 新增或修改数据后按 `spec/validation.json` 做跨文件校验。
9. 国际、港澳台、国内三个 scope 互不混用。
10. `schemaVersion` 表示单个文件结构版本；`standardVersion` 表示整体规范版本。

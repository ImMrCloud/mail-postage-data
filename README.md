# mail-postage-data

中国邮政邮费结构化数据仓库，面向 App / Web / 后端程序读取。

当前数据覆盖三个独立资费范围：**中国大陆境内**、**中国大陆寄往香港/澳门/台湾**、**中国大陆寄往国际目的地**。基础寄递资费与特别业务资费分离维护。

> 本仓库不是中国邮政官方发布渠道。资费、可寄达性、禁限寄要求和临时停运信息，请以中国邮政及目的地邮政的最新规定为准。资费信息自己整理而成，不保证内容最新，如有错误欢迎提交issue。

## 目录结构

```text
mail-postage-data/
├── README.md
├── countryData.json
└── postage/
    ├── index.json                 # 数据集总入口
    ├── services.json              # 寄递服务注册表
    ├── notices.json               # 通用禁限寄/服务提示
    ├── spec/                      # 机器可读规范
    │   ├── index.json
    │   ├── core.json
    │   ├── pricing.json
    │   ├── notices.json
    │   └── validation.json
    ├── rates/
    │   ├── domestic/
    │   │   └── basic.json
    │   ├── hong-kong-macau-taiwan/
    │   │   └── basic.json
    │   └── international/
    │       ├── air-letter.json
    │       ├── air-other-mail.json
    │       ├── sal-letter.json
    │       ├── sal-other-mail.json
    │       ├── surface-letter.json
    │       └── surface-other-mail.json
    ├── zones/                     # 国际目的地分区
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

App 或维护脚本应首先读取：

```text
postage/index.json
```

其中 `files.specification` 指向：

```text
postage/spec/index.json
```

当前规范版本为 `standardVersion: "1.0.0"`。详细规则分为：

- `spec/core.json`：scope、rateCategory、service、mailType、路径、索引规则。
- `spec/pricing.json`：所有支持的计价类型、字段和计算公式。
- `spec/notices.json`：通用 notice 与特别业务 notice 的匹配和优先级规则。
- `spec/validation.json`：App / CI 应执行的跨文件一致性校验。

## Scope

统一只使用以下三个值：

```text
domestic
hong-kong-macau-taiwan
international
```

含义：

- `domestic`：中国大陆境内寄递。
- `hong-kong-macau-taiwan`：中国大陆寄往香港、澳门、台湾。
- `international`：中国大陆寄往其他国际目的地；国际数据排除 `CN`、`HK`、`MO`、`TW`。

不要再使用 CamelCase 或其他别名表示 scope。

## 资费分类

所有资费归入两类：

```text
delivery
special
```

### delivery

基础寄递资费。服务注册在 `postage/services.json`，通过 `rateCategory: "delivery"` 标识。

### special

特别业务资费，例如挂号、回执、保价、约投、存局候领、海关验关等。统一放在 `postage/special-rates/`，通过 `rateCategory: "special"` 标识。

特别业务通常在基础寄递资费之外叠加，不应伪装成基础寄递 service。

## 目的地代码

国际国家/地区优先使用 ISO 3166-1 alpha-2，例如：

```text
JP
US
GB
```

当资费表把某些岛屿或属地作为独立邮政路向、而 ISO2 无法准确表达时，使用 `countryData.json` 中的自定义 `locale`，例如：

```text
PT-AZORES
PT-MADEIRA
SH-ASCENSION
```

港澳台固定使用：

```text
HK
MO
TW
```

## App 读取方式

推荐顺序：

1. 读取 `postage/index.json`。
2. 读取 `postage/spec/index.json` 并确认支持当前 `standardVersion`。
3. 读取 `services.json`，按 scope、mailType、transport 等选择 `serviceId`。
4. 根据 service 的 `rateFile` / `zoneFile` 加载基础寄递资费。
5. 根据 scope 加载 `special-rates/index.json` 和对应特别业务资费。
6. 加载 `postage/notices.json` 与 `postage/special-rates/notices.json`。
7. 按 `spec/validation.json` 执行一致性校验；出现 error 级问题时不要静默计算邮资。

国际分区不得使用单一 `country.zone`。应采用：

```text
serviceId + destination -> zoneId
```

因为同一国家在 AIR / SAL / SURFACE 或不同邮件种类中可以属于不同资费组。

## 国内资费

基础寄递资费：

```text
postage/rates/domestic/basic.json
```

目前包括：

- 信函（本埠 / 外埠分段累计计费）
- 明信片
- 印刷品
- 邮简
- 义务兵免费信函
- 盲人读物

特别业务资费：

```text
postage/special-rates/domestic.json
```

包括挂号、约投、回执、保价、存局候领、撤回或更改收件人名址等。

## 港澳台资费

基础寄递资费：

```text
postage/rates/hong-kong-macau-taiwan/basic.json
```

适用目的地为 `HK`、`MO`、`TW`。

特别业务资费：

```text
postage/special-rates/hong-kong-macau-taiwan.json
```

港澳台存在目的地级特别业务例外，因此 `special-rates/index.json` 对该 scope 使用：

```json
"destinationIndependent": false
```

例如：**寄往台湾的附回执函件暂不收寄**。真实限制保存在：

```text
postage/special-rates/notices.json
```

App 不应只根据“港澳台统一资费”推断台湾也支持回执。

## 国际资费

国际基础寄递按运输方式分为：

- `AIR` 航空
- `SAL` 空运水陆路
- `SURFACE` 水陆路

相关资费位于：

```text
postage/rates/international/
```

需要目的地分区的服务通过：

```text
postage/zones/
```

解析 zoneId。

国际特别业务统一位于：

```text
postage/special-rates/international.json
```

特别业务表中的函件、小包、印刷品专袋和包裹相关费用应完整保留，不因业务类别不同而删除数据；适用范围通过 `applicableCategories`、`applicableMailTypes`、`conditions` 等字段表达。

## Pricing 类型

完整定义见：

```text
postage/spec/pricing.json
```

当前支持：

```text
flat
flat_by_locality
free
base_plus_increment
surcharge_per_increment
air_surcharge_only
weight_tiers
tiered_incremental
value_increment
value_increment_with_minimum
external_rule
```

凡原资费规则包含“每续重 X 克或其零数”“不足 X 克按 X 克计算”等规则时，必须显式写：

```json
"rounding": "ceil"
```

免费项目必须显式使用 `free` 且 `price: 0`，不得只写文字备注。

## Notices

### 通用 notice

`postage/notices.json` 用于禁限寄、暂停服务、海关、包装、地址、时效等不属于价格的数据。

主要匹配：

```text
destination + serviceId + mailType
```

### 特别业务 notice

`postage/special-rates/notices.json` 用于特别业务可用性、限制和计费提示。

主要匹配：

```text
rateScope + destination + specialRateId
```

支持结构化扩展：

```text
metadata.available
metadata.applicableDestinations
metadata.excludedDestinations
metadata.mailTypes
metadata.conditions
```

所有 `enabled: false` 的 example/examples 仅用于说明字段用法，App 必须忽略。

## 数据维护原则

1. 新数据先确认属于 `delivery` 还是 `special`。
2. 新 service 必须在 `services.json` 注册，并把 mailType 先加入 `mailTypes`。
3. 新的国际国家分组应新增或更新对应 zone 文件，不强行复用其他服务的 zoneId。
4. 所有路径必须通过 index/service 引用，不依赖硬编码目录猜测。
5. 新 pricing 类型必须先写入 `spec/pricing.json`，再用于业务数据。
6. 新 notice 扩展字段应同步补充 `spec/notices.json` 或对应 notice 的 fieldGuide。
7. 新增或修改数据后应按 `spec/validation.json` 做跨文件校验。
8. 特殊岛屿/属地优先使用 ISO；ISO 无法准确表达时使用 `countryData.json` 自定义 locale。
9. 国际、港澳台、国内三个 scope 互不混用。
10. `schemaVersion` 表示单个文件结构版本；`standardVersion` 表示整体规范版本。

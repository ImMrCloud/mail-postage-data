# mail-postage-data

中国邮政邮费结构化数据仓库，面向 App / Web / 后端程序读取。

当前优先整理 **中国大陆寄往国际目的地** 的资费数据；中国大陆境内以及香港、澳门、台湾地区将单独维护，不混入国际资费分组。

> 本仓库不是中国邮政官方发布渠道。资费、可寄达性、禁限寄要求和临时停运信息，请以中国邮政及目的地邮政的最新规定为准。

## 目录结构

```text
mail-postage-data/
├── README.md
└── postage/
    ├── index.json
    ├── services.json
    ├── notices.json
    ├── zones/
    │   ├── air-letter.json
    │   ├── air-other-mail.json
    │   ├── sal-letter.json
    │   └── sal-other-mail.json
    └── rates/
        ├── air-letter.json
        ├── air-other-mail.json
        ├── sal-letter.json
        └── sal-other-mail.json
```

## 设计原则

- 国家/地区优先使用 ISO 3166-1 alpha-2 两位代码，例如 `JP`、`US`、`GB`。
- 对亚速尔群岛、马德拉群岛、阿森松岛等不能仅靠主权国家 ISO 代码表达的邮政目的地，使用 `postalDestinations` 保存独立邮政目的地标识。
- 运输方式、函件种类、国家分组、计费规则、目的地提示分离。
- **同一国家在不同运输方式或函件类型中允许属于不同资费组。**
- 国际数据排除 `CN`、`HK`、`MO`、`TW`。
- 价格统一使用人民币 `CNY`。
- “每续重 X 克或其零数”统一以 `rounding: "ceil"` 表示向上取整。

## 航空函件 AIR

### 信函

使用 `postage/zones/air-letter.json` + `postage/rates/air-letter.json`，采用四组资费。

### 其他航空函件

使用 `postage/zones/air-other-mail.json` + `postage/rates/air-other-mail.json`。

包括：明信片、航空邮简、印刷品、盲人邮件、小包、印刷品专袋。

## 空运水陆路 SAL

SAL 使用与航空函件完全独立的分组文件，App 不应复用 AIR 的 zoneId。

### 1. SAL 信函

文件：

```text
postage/zones/sal-letter.json
postage/rates/sal-letter.json
```

四组价格：

- 第一组：20g 内 4.50 元；续 10g 0.50 元
- 第二组：20g 内 5.00 元；续 10g 0.60 元
- 第三组：20g 内 5.50 元；续 10g 0.70 元
- 第四组：20g 内 6.50 元；续 10g 0.80 元

国家/地区按资费表逐项录入。例如：

- 第一组：韩国、日本
- 第二组：塞浦路斯
- 第三组：欧洲多数国家及美国、加拿大、澳大利亚等
- 第四组：科摩罗、莱索托、圣多美和普林西比、安圭拉、佛得角、巴西、格陵兰、巴巴多斯、百慕大，以及亚速尔群岛、马德拉群岛、阿森松岛等邮政目的地

### 2. SAL 其他函件

文件：

```text
postage/zones/sal-other-mail.json
postage/rates/sal-other-mail.json
```

这套三组只用于 SAL 的印刷品、盲人邮件、小包、印刷品专袋。

- 第一组：格鲁吉亚
- 第二组：按原表明确列出的日本、韩国及欧洲、北美、澳大利亚部分目的地
- 第三组：按原表明确列出的俄罗斯、巴西、玻利维亚、格陵兰、瑞士、塞浦路斯、亚美尼亚等，以及部分特殊邮政目的地

资费：

- 明信片：每件 4.50 元，不分组
- 印刷品：
  - 第一组：20g 内 4.00 元；续 10g 1.90 元
  - 第二组：20g 内 4.50 元；续 10g 2.20 元
  - 第三组：20g 内 5.00 元；续 10g 2.50 元
- 盲人邮件：基本资费免收，仅收 SAL 运费；每 10g 分别为 0.30 / 0.30 / 0.40 元
- 小包：
  - 第一组：100g 内 22.00 元；续 100g 18.00 元
  - 第二组：100g 内 27.00 元；续 100g 23.00 元
  - 第三组：100g 内 32.00 元；续 100g 28.00 元
- 印刷品专袋：
  - 第一组：5000g 内 455.00 元；续 1000g 100.00 元
  - 第二组：5000g 内 600.00 元；续 1000g 120.00 元
  - 第三组：5000g 内 730.00 元；续 1000g 145.00 元

## App 读取方式

建议 App 首先读取：

```text
postage/index.json
```

然后按 `serviceId` 从 `services.json` 找到该服务对应的 `zoneFile` 和 `rateFile`。

不要使用类似：

```text
country.zone
```

而应使用：

```text
serviceId + destination → zoneId
```

例如同为日本：

```text
JP + INTL_AIR_LETTER → AIR 信函分组
JP + INTL_AIR_SMALL_PACKET → AIR 其他函件分组
JP + INTL_SAL_LETTER → SAL 信函第一组
JP + INTL_SAL_SMALL_PACKET → SAL 其他函件第二组
```

### 通用首重 + 续重计算

```js
function calculatePostage(weight, pricing) {
  if (weight <= pricing.baseWeight) return pricing.basePrice;

  return pricing.basePrice +
    Math.ceil((weight - pricing.baseWeight) / pricing.incrementWeight) *
    pricing.incrementPrice;
}
```

固定每件资费使用 `type: "flat"`；盲人邮件使用基本资费免收 + 按重量收运输附加费的独立计价结构。

## 特殊邮政目的地

有些资费表会把某些岛屿或属地作为独立邮政目的地，但它们并没有独立的 ISO 3166-1 国家代码，或者其主权国家本身属于另一资费组。

因此 zone 文件支持：

```json
{
  "postalDestinations": [
    {
      "id": "PT-AZORES",
      "nameZh": "亚速尔群岛",
      "parentCountry": "PT"
    }
  ]
}
```

App 在目的地选择层可以同时支持普通 ISO 国家代码和这些邮政目的地 ID。

## 目的地提示 / 禁限寄说明

`postage/notices.json` 用来维护所有不属于价格的信息，例如：

- 某国家某类函件禁止或限制寄递某种物品
- 暂停收寄或某运输方式不可用
- 海关、地址、包装、时效提醒
- App 自定义提示

提示可通过 `destination + serviceId + mailType` 精确匹配，也可将某一层设为 `null` 形成更宽泛的规则。

## 数据维护原则

1. 新运输方式：增加 transport/service。
2. 新函件类型：增加 mailType/service。
3. 新的国家分组：新增独立 zone 文件，不强行复用其他运输方式的分组。
4. 新价格表：新增或更新对应 rate 文件。
5. 特殊岛屿/属地：优先使用 ISO；无法准确表达时写入 `postalDestinations`。
6. 禁限寄、暂停服务、特殊提示：写入 `notices.json`，不要混入价格文件。

这样可以继续扩展水陆路、中国大陆境内以及港澳台地区资费，而不会破坏现有 App 数据结构。

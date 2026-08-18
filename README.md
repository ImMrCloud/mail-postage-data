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
    │   └── air-other-mail.json
    └── rates/
        ├── air-letter.json
        └── air-other-mail.json
```

## 设计原则

- 国家/地区使用 ISO 3166-1 alpha-2 两位代码，例如 `JP`、`US`、`GB`。
- 运输方式、函件种类、国家分组、计费规则、目的地提示分离，避免重复数据。
- **同一国家在不同函件类型中允许属于不同资费组。** 因此“航空信函”和“其他航空函件”分别维护自己的 zone 文件。
- 国际数据排除 `CN`、`HK`、`MO`、`TW`。
- 价格统一使用人民币 `CNY`。
- “每续重 X 克或其零数”统一以 `rounding: "ceil"` 表示向上取整。

## 当前已录入的航空函件

### 1. 信函

使用独立的四组分组：`postage/zones/air-letter.json`。

- 第一组：20g 以内 5.00 元；超过 20g 每续重 10g 或其零数 1.00 元
- 第二组：20g 以内 5.50 元；超过 20g 每续重 10g 或其零数 1.50 元
- 第三组：20g 以内 6.00 元；超过 20g 每续重 10g 或其零数 1.80 元
- 第四组：20g 以内 7.00 元；超过 20g 每续重 10g 或其零数 2.30 元

### 2. 其他航空函件

以下类别共用另一套三组分组：`postage/zones/air-other-mail.json`。

- 明信片：每件 5.00 元，三组同价
- 航空邮简：每件 5.50 元，三组同价
- 印刷品：
  - 第一组：20g 内 4.50 元；续 10g 2.20 元
  - 第二组：20g 内 5.00 元；续 10g 2.50 元
  - 第三组：20g 内 6.00 元；续 10g 2.80 元
- 盲人邮件：基本资费免收，仅收航空运费；每 10g 分别为 0.60 / 0.80 / 1.00 元
- 小包：
  - 第一组：100g 内 25.00 元；续 100g 23.00 元
  - 第二组：100g 内 30.00 元；续 100g 27.00 元
  - 第三组：100g 内 35.00 元；续 100g 33.00 元
- 印刷品专袋：
  - 第一组：5000g 内 485.00 元；续 1000g 100.00 元
  - 第二组：5000g 内 610.00 元；续 1000g 120.00 元
  - 第三组：5000g 内 730.00 元；续 1000g 145.00 元

对应价格文件：`postage/rates/air-other-mail.json`。

## 三组国家分配

`air-other-mail.json` 按资费表逐项转成 ISO 国家/地区代码。

- 第一组和第二组：按原表明确列出的国家逐项映射。
- 第三组：原表写作“其他国家和地区”，因此本仓库把除第一、第二组及 `CN/HK/MO/TW` 外的其余 ISO 3166-1 目的地展开到第三组。
- 这只是资费分组映射，不代表某个目的地当前一定可以收寄。

因此，同一个目的地可能出现：

```text
JP + LETTER        → air-letter.json 的信函分组
JP + SMALL_PACKET  → air-other-mail.json 的三组分组
```

App 不应只按国家缓存一个全局 `zoneId`，而应按 **serviceId + destination** 查询。

## App 读取方式

建议 App 首先读取：

```text
postage/index.json
```

然后根据 `services.json` 中对应服务找到该服务的：

```json
{
  "zoneFile": "./zones/...",
  "rateFile": "./rates/..."
}
```

典型流程：

```text
用户选择：JP + 航空 + 小包 + 180g
        ↓
services.json → INTL_AIR_SMALL_PACKET
        ↓
zones/air-other-mail.json → 找 JP 所属组
        ↓
rates/air-other-mail.json → SMALL_PACKET 对应组价格
        ↓
notices.json → 查询 JP / 该 service / SMALL_PACKET 的提示
        ↓
显示最终价格与寄件提醒
```

### 通用首重 + 续重计算

```js
function calculatePostage(weight, baseWeight, basePrice, incrementWeight, incrementPrice) {
  if (weight <= baseWeight) return basePrice;

  return basePrice +
    Math.ceil((weight - baseWeight) / incrementWeight) * incrementPrice;
}
```

明信片、航空邮简属于 `flat` 固定价格，不需要重量计算。

盲人邮件使用 `air_surcharge_only`：基本资费免收，从实际重量开始按每 10g 或其零数收取航空运费。

## 目的地提示 / 禁限寄说明

`postage/notices.json` 用来维护所有不属于“价格”的信息，例如：

- 某国家某类函件禁止寄某种物品
- 限寄数量、包装或申报要求
- 暂停收寄或某运输方式不可用
- 海关、地址、时效等提醒
- App 自定义提示

示例：

```json
{
  "enabled": true,
  "id": "JP-AIR-SMALL-PACKET-001",
  "scope": {
    "destination": "JP",
    "serviceId": "INTL_AIR_SMALL_PACKET",
    "mailType": "SMALL_PACKET"
  },
  "type": "restriction",
  "severity": "warning",
  "title": "寄件提醒",
  "message": "这里填写提示。",
  "prohibitedItems": [],
  "restrictedItems": [],
  "source": null,
  "effectiveFrom": null,
  "effectiveTo": null,
  "tags": ["custom"],
  "metadata": {}
}
```

`metadata` 可放任何项目自定义字段，而无需改变资费 JSON 结构。

## 数据维护原则

1. 新运输方式：增加 transport/service。
2. 新函件类型：增加 mailType/service。
3. 如果使用新的国家分组：新增独立 zone 文件，不复用不相同的旧分组。
4. 新价格表：新增或更新对应 rate 文件。
5. 禁限寄、服务暂停、特殊提示：写入 `notices.json`，不要混入价格文件。

这样可以保证 App 长期兼容，同时允许后续继续加入 SAL、水陆路、中国大陆境内以及港澳台地区资费。

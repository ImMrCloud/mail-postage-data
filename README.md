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
    │   └── air-letter.json
    └── rates/
        └── air-letter.json
```

## 设计原则

- 国家/地区使用 ISO 3166-1 alpha-2 两位代码，例如 `JP`、`US`、`GB`。
- 运输方式、函件种类、国家分组、计费规则、目的地提示分离，避免重复数据。
- 同一国家在不同运输方式或函件种类中可以属于不同资费组。
- 国际数据排除 `CN`、`HK`、`MO`、`TW`。
- 价格统一使用人民币 `CNY`。
- 续重采用向上取整，例如信函超过 20g 后，每 10g 或其零数按一个续重单位计费。
- 地域型资费分组使用联合国统计司 UN M49 地理分区展开为 ISO 国家/地区代码；这只用于数据映射，不等于逐国确认当前可寄达。

## 当前数据

目前已加入：

- 航空函件
  - 信函
    - 第一组：20g 以内 5.00 元；超过 20g 每续重 10g 或其零数 1.00 元
    - 第二组：20g 以内 5.50 元；超过 20g 每续重 10g 或其零数 1.50 元
    - 第三组：20g 以内 6.00 元；超过 20g 每续重 10g 或其零数 1.80 元
    - 第四组：20g 以内 7.00 元；超过 20g 每续重 10g 或其零数 2.30 元

### 航空信函国家分组

`postage/zones/air-letter.json` 已把资费表的地域规则展开为 ISO 3166-1 alpha-2 列表。

分组方法：

1. 第一组直接采用资费表明确列出的国家：朝鲜、蒙古、越南、日本、韩国、哈萨克斯坦、吉尔吉斯斯坦、塔吉克斯坦、乌兹别克斯坦、土库曼斯坦。
2. 第二组为其余亚洲国家或地区。
3. 第三组为欧洲国家或地区，以及美国、加拿大、澳大利亚、新西兰。
4. 第四组为美洲其他国家或地区、非洲国家或地区、大洋洲其他国家或地区。
5. 中国大陆、香港、澳门、台湾不进入国际分组。
6. 南极洲 `AQ` 不属于原资费表列出的五大地域范围，因此暂放在 `unassignedDestinations`，不自动猜测资费组。

`mappingBasis.postalAvailabilityVerified` 当前为 `false`，表示这里完成的是 **资费地域映射**，不是逐国邮路可用性确认。

## App 读取方式

建议 App 首先读取：

```text
postage/index.json
```

然后根据其中的 `files` 加载：

- `services`：运输方式 / 函件类型
- `airLetterZones`：目的地属于哪个资费组
- `airLetterRates`：各资费组价格
- `notices`：目的地、服务、函件类型相关提示

典型流程：

```text
目的地 JP
  ↓
zones/air-letter.json → AIR_LETTER_ZONE_1
  ↓
rates/air-letter.json → 首重/续重价格
  ↓
notices.json → 查询 JP + INTL_AIR_LETTER + LETTER 的附加提示
  ↓
App 展示最终资费和提示
```

### 计算示例

第一组航空信函，重量 37g：

```text
5.00 + ceil((37 - 20) / 10) × 1.00 = 7.00 元
```

JavaScript 示例：

```js
function calculatePostage(weight, pricing) {
  if (weight <= pricing.baseWeight) {
    return pricing.basePrice;
  }

  const extraWeight = weight - pricing.baseWeight;
  const increments = Math.ceil(extraWeight / pricing.incrementWeight);

  return pricing.basePrice + increments * pricing.incrementPrice;
}
```

## 目的地提示 / 禁限寄说明

`postage/notices.json` 用来维护不属于“价格”的信息，例如：

- 某个国家的某类函件禁止寄某种物品
- 某种物品允许寄，但存在数量、包装或申报限制
- 某国家某种运输方式暂停或暂不可用
- 海关申报提醒
- 地址格式提醒
- 你自己希望显示给 App 用户的任意文字

建议提示记录采用以下结构：

```json
{
  "enabled": true,
  "id": "JP-AIR-LETTER-001",
  "scope": {
    "destination": "JP",
    "serviceId": "INTL_AIR_LETTER",
    "mailType": "LETTER"
  },
  "type": "restriction",
  "severity": "warning",
  "title": "寄件提醒",
  "message": "这里填写你希望显示给用户的说明。",
  "prohibitedItems": ["示例物品 A"],
  "restrictedItems": ["示例物品 B"],
  "source": null,
  "effectiveFrom": null,
  "effectiveTo": null,
  "tags": ["custom"],
  "metadata": {
    "anyCustomField": "这里也可以放你自己的扩展内容"
  }
}
```

### Scope 可以逐级放宽

只针对“日本 + 航空函件 + 信函”：

```json
{
  "destination": "JP",
  "serviceId": "INTL_AIR_LETTER",
  "mailType": "LETTER"
}
```

针对日本的所有国际业务：

```json
{
  "destination": "JP",
  "serviceId": null,
  "mailType": null
}
```

针对全部国际目的地的航空函件：

```json
{
  "destination": null,
  "serviceId": "INTL_AIR_LETTER",
  "mailType": null
}
```

因此以后不需要为了添加一句提示去修改资费文件。

## 数据来源与维护

航空信函资费来自现有国际函件资费表。第一组为原表逐国明确列举；第二至第四组为地域描述，本仓库依据 UN M49 地理分区展开为 ISO 国家/地区代码。

地域归属和实际邮政服务可用性是两件不同的事。后续如取得中国邮政官方逐国禁限寄、暂停收寄、特殊海关要求等资料，应写入 `notices.json` 或进一步拆分专门的数据文件，而不修改基础资费组逻辑。

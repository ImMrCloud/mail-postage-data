# mail-postage-data

中国邮政邮费结构化数据仓库，面向 App / Web / 后端程序读取。

当前优先整理 **中国大陆寄往国际目的地** 的资费数据；中国大陆境内以及香港、澳门、台湾地区将单独维护，不混入国际资费分组。

## 目录结构

```text
mail-postage-data/
├── README.md
├── countries.json
└── postage/
    ├── index.json
    ├── services.json
    ├── zones/
    │   └── air-letter.json
    └── rates/
        └── air-letter.json
```

## 设计原则

- 国家/地区使用 ISO 3166-1 alpha-2 两位代码，例如 `JP`、`US`、`GB`。
- 运输方式、函件种类、国家分组和计费规则分离，避免重复数据。
- 同一国家在不同运输方式或函件种类中可以属于不同资费组。
- 国际数据排除 `CN`、`HK`、`MO`、`TW`。
- 价格统一使用人民币 `CNY`。
- 续重采用向上取整，例如信函超过 20g 后，每 10g 或其零数按一个续重单位计费。

## 当前数据

目前先加入：

- 航空函件
  - 信函
    - 第一组：20g 以内 5.00 元；超过 20g 每续重 10g 或其零数 1.00 元
    - 第二组：20g 以内 5.50 元；超过 20g 每续重 10g 或其零数 1.50 元
    - 第三组：20g 以内 6.00 元；超过 20g 每续重 10g 或其零数 1.80 元
    - 第四组：20g 以内 7.00 元；超过 20g 每续重 10g 或其零数 2.30 元

## 读取方式

App 建议先读取：

```text
postage/index.json
```

然后根据其中的文件索引加载服务、国家分组和资费规则。

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

## 数据说明

航空信函分组依据现有国际函件资费表整理。第一组在原表中明确列出国家；第二、第三、第四组以地域规则描述，因此本仓库将这些地域规则保存在 JSON 中，后续可再补充和校准逐国映射。

本仓库不是中国邮政官方发布渠道，实际寄件时请以中国邮政最新公布的资费和可寄达信息为准。

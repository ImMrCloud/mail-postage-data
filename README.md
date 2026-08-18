# mail-postage-data

中国邮政邮费结构化数据仓库，面向 App / Web / 后端程序读取。

当前规范版本：`1.5.0`。

## 资费范围

- `domestic`：中国大陆境内
- `hong-kong-macau-taiwan`：中国大陆寄往香港、澳门、台湾
- `international`：中国大陆寄往其他国际目的地

## 普通包裹

入口：`postage/rates/domestic/ordinary-parcel/index.json`

- 已录入起寄省按 `originFiles + routeGroups` 路由；未录入起寄省不得回退到其他省价格。
- 可选填写长宽高，体积重量（kg）=`长(cm) × 宽(cm) × 高(cm) ÷ 6000`。
- 尺寸完整时分别计算实重资费与泡重资费，采用资费较高的基础寄递资费。
- 基础资费已包含挂号属性，因此普通包裹附回执不另收挂号费。
- 普通包裹可以同时办理保价和回执。
- 包裹保价费为保价金额的 **1%**，最高保价金额 **100000 元**。
- 8折/7折优惠只作用于基础寄递资费；保价费、回执费等 `special` 费用按原价另加。
- 优惠凭证一次只能使用一个，不允许叠加。
- 普通包裹不提供存局候领手续费业务。

## 家乡包裹贴

- 重量档：`≤1kg 4元 / >1kg≤3kg 6元 / >3kg≤5kg 11元 / >5kg≤10kg 19元`
- 可办理保价，保价费为保价金额 **1%**，最高 **100000 元**。
- 不可办理回执。
- 不使用普通包裹折扣或泡重规则。

## 函件保价

国内、国际、港澳台的“保价函件”都只允许 `mailType=LETTER`。

- 最高保价金额：**20000 元**。
- 国内信函保价必须挂号。
- 国际/港澳台保价函件保价费必须同时使用 `INSURED_LETTER_HANDLING_FEE`。
- 国际/港澳台保价函件手续费已经包含挂号费，因此不得重复收取独立 `REGISTRATION_FEE`。

## 特别业务交互规则

规范入口：`postage/spec/special-services.json`

特别业务支持：

- `selection.mode=selectable`：寄件时可以选择并参与计算。
- `selection.mode=display-only`：仅公示，不得作为复选框，也不得参与寄件计算；应排在可选项目之后。
- `requiresRateIds`：全部依赖。
- `requiresAnyRateIds`：满足其中任一依赖即可。
- `includesRateIds`：费用已经包含，不得重复计费。
- `replacesRateIds`：选择当前项目后取消并禁用被替代项目。
- `excludesRateIds`：互斥项目。

消费者必须支持**反向级联取消**：取消某项后，所有失去依赖前提的业务也必须立即取消。

国际/港澳台的典型交互：

1. 勾选回执 → 自动勾选挂号。
2. 已勾选回执+挂号，再勾选保价 → 自动勾选保价函件手续费、取消并禁用独立挂号，回执继续保留。
3. 取消保价函件手续费 → 保价自动取消；若回执没有其他挂号前提，回执也取消。
4. 取消挂号 → 所有仅依赖该挂号的回执/保价自动取消。

## 仅公示项目

以下费用不属于交寄时可选收费，必须只展示：

- 国内：存局候领手续费、撤回邮件或更改收件人名址手续费。
- 国际/港澳台：出售国际回信券、撤回/修改收件人名址申请费、进口欠资、存局候领、逾期保管、海关查验等。

存局候领费用由收件人承担。

## 规范文件

- `postage/spec/index.json`
- `postage/spec/core.json`
- `postage/spec/pricing.json`
- `postage/spec/special-services.json`
- `postage/spec/notices.json`
- `postage/spec/validation.json`

> 本仓库不是中国邮政官方发布渠道。实际收费、收寄条件和临时业务调整以中国邮政最新规定为准。

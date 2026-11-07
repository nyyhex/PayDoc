# UPay Open API 文档

> Base URL: `/api/open`
> 请求方式: 全部为 `POST`，`Content-Type: application/json`
> 鉴权: `X-API-KEY` + `X-SIGN`（见第 1 节）
> 响应格式: `{ "code": 1, "message": "success", "englishMessage": null, "data": ... }`

本文档对应 `OpenApiController`（`/api/open/*`），供商户/渠道方以 API Key 方式接入：钱包列表、OTP 收发、
收款地址查询、账单查询、钱包绑定/解绑、银行卡列表、余额查询、代付、补单。

接口覆盖的钱包：Mobikwik / PhonePe / Paytm / BharatPe / Amazon Pay / Airtel / Freecharge 以及 Yape（秘鲁）。

---

## 1. 接入与鉴权

### 1.1 获取 API Key

在商户后台的 API Key 页面创建密钥，创建后会返回：

| 字段 | 说明 |
| --- | --- |
| `apiKey` | 24 字节随机数做 base64url（无 padding）后的字符串，放在请求头 `X-API-KEY` |
| `apiSecret` | 48 字节随机数做 base64url（无 padding）后的字符串，**只用于本地计算签名，绝不随请求发送** |
| `whiteIp` | IP 白名单，多个用英文逗号分隔；为空表示不限制 |

### 1.2 请求头

| Header | 必传 | 说明 |
| --- | --- | --- |
| `Content-Type` | 是 | 固定 `application/json` |
| `X-API-KEY` | 是 | 1.1 中的 `apiKey` |
| `X-SIGN` | 是 | 按 1.3 计算的签名，**大写十六进制** |

> 请求来源 IP 必须命中 `whiteIp`（`whiteIp` 为空时不校验）。

### 1.3 签名规则

签名只针对 **请求体 JSON**，不包含 URL、query 和请求头。

1. 把请求体解析成 JSON 对象；
2. 去掉 `sign`、`sign_type` 两个字段（其他字段即使服务端不认识，也必须参与签名）；
3. 按 key 的字典序（`TreeMap`，区分大小写）升序排列；
4. 逐个拼成 `key=value`，用 `&` 连接，**跳过值为空字符串或纯空白的字段**；
5. 末尾直接拼接 `apiSecret`（**没有** `&key=` 之类的分隔符）；
6. 对整串做 MD5，输出大写十六进制。

值转字符串规则（与请求体里的写法要一致）：

| JSON 类型 | 参与签名的字符串 |
| --- | --- |
| 字符串 | 原样 |
| 数字 | 数字的字面值，如 `100`、`10.5` |
| 布尔 | `true` / `false` |
| 对象/数组 | 紧凑 JSON（当前接口没有嵌套参数，建议一律用字符串） |

示例：请求体与密钥

```json
{ "phone": "987654321", "typeCode": "yape" }
```

```
apiSecret = 5f2b9c7d1e4a8b3f6c0d9e2a7b5c4d8e
待签名串   = phone=987654321&typeCode=yape5f2b9c7d1e4a8b3f6c0d9e2a7b5c4d8e
X-SIGN     = F5010DCA05AB917D6348196B4EBF3465
```

Python：

```python
import hashlib, json

def sign(body: dict, secret: str) -> str:
    params = {k: v for k, v in body.items() if k not in ("sign", "sign_type")}
    parts = []
    for k in sorted(params):
        v = params[k]
        if isinstance(v, bool):
            v = "true" if v else "false"
        else:
            v = str(v)
        if v.strip():
            parts.append(f"{k}={v}")
    return hashlib.md5(("&".join(parts) + secret).encode("utf-8")).hexdigest().upper()
```

Java：

```java
String sign = SignGenerator.generateSign(JSON.parseObject(body), apiSecret);
```

注意事项：

- 没有任何参数的接口（如 `/saasBankList`）**也必须发送 `{}` 作为请求体**，此时签名为 `MD5(apiSecret)` 的大写结果。空请求体或非 JSON 请求体会在鉴权阶段直接失败。
- 服务端按实际收到的 JSON 重新计算签名，所以**签名的内容和发出去的内容必须完全一致**；多带一个自定义字段也要一起签。
- 金额、手机号等参数建议统一用字符串，避免数字类型在不同语言里序列化差异导致签名不一致。

### 1.4 调用示例

```bash
curl -X POST 'https://<host>/api/open/saasGetBal' \
  -H 'Content-Type: application/json' \
  -H 'X-API-KEY: 3TqWc0h2yJ8mN5vB1xL9fK7p' \
  -H 'X-SIGN: F5010DCA05AB917D6348196B4EBF3465' \
  -d '{"phone":"987654321","typeCode":"yape"}'
```

### 1.5 通用响应

业务成功：

```json
{
  "code": 1,
  "message": "success",
  "englishMessage": null,
  "data": { }
}
```

业务失败（HTTP 状态码仍为 200）：

```json
{
  "code": 400,
  "message": "account not found",
  "englishMessage": null,
  "data": null
}
```

| 字段 | 说明 |
| --- | --- |
| `code` | `1` 成功；非 1 为失败，见 1.6 |
| `message` | 提示信息。业务失败时是具体原因（如 `account not found`） |
| `englishMessage` | 预留字段，一般为 `null` |
| `data` | 业务数据；无返回值的接口为 `null` |

`/saasGetTrans` 的 `data` 是分页结构：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `pageNum` | Integer | 当前页，从 1 开始 |
| `pageSize` | Integer | 每页条数 |
| `total` | Long | 总条数 |
| `pages` | Integer | 总页数 |
| `data` | Array | 当前页数据 |

### 1.6 错误码

| code | 场景 |
| --- | --- |
| `1` | 成功 |
| `400` | 业务失败。所有接口内部的异常都会被统一包成 400，具体原因看 `message` |
| `401` | 鉴权失败。`X-API-KEY`/`X-SIGN` 缺失、API Key 不存在、签名不匹配、IP 不在白名单。**该响应只有 `code`/`message` 两个字段** |
| `500` | 服务端异常，或请求体不是合法 JSON（例如 `"System busy"`） |

鉴权失败的 `message` 取值：

| message | 原因 |
| --- | --- |
| `API Key or Sign is required` | 缺少 `X-API-KEY` 或 `X-SIGN` |
| `API Key is invalid` | API Key 不存在或被删除 |
| `[x.x.x.x] not in white ips, please add it` | 来源 IP 不在白名单 |
| `Sign is invalid` | 签名不一致（最常见：参与签名的字段/取值与实际请求体不符） |

常见业务 `message`：

| message | 说明 |
| --- | --- |
| `phone is empty` / `typeCode is empty` | 必填参数缺失 |
| `phone should be 10 digits` | OTP 相关接口要求 10 位印度手机号 |
| `yape phone must be 8 digits` | Yape 手机号为 8 位；带 `+51`/`51`/`00591`/`0051` 前缀也可，服务端会截取后 8 位 |
| `typeCode is invalid` / `xxx not support otp` | 钱包类型不存在，或该类型不支持 OTP |
| `account not found` | 账号不存在，或未绑定到当前 API Key 所属用户 |
| `account bind other user` | 账号已绑定到其他用户 |
| `account unauthorized` | 账号被禁用/封禁 |
| `retry again` | 同类请求正在处理（分布式锁未释放），稍后重试 |
| `third pay error` | 上游钱包返回失败，`message` 里可能带上游原文 |

### 1.7 钱包类型 typeCode

| 分类 | typeCode |
| --- | --- |
| OTP 类（`otpType`） | `mobikwikotp_v2`、`mobikwikotp`、`phonepebusiness`、`paytmbusiness`、`amazonpay`、`airtel`、`freecharge`、`phonepeotp`、`bharotp` |
| 应用类（`appType`） | `mobikwik`、`phonepe`、`paytm`、`bhar`、`mobikwik_wallet_auto_buy`、`yape` |

- 支持代收（`sell`）：`bhar`、`paytm`、`phonepe`、`freecharge`、`mobikwik`、`mobikwikotp_v2`、`mobikwikotp`、`airtel`、`amazonpay`、`phonepebusiness`、`paytmbusiness`、`phonepeotp`、`bharotp`、`mobikwik_wallet_auto_buy`、`yape`
- 支持代付（`buy`）：`paytm`、`phonepe`、`mobikwik`、`mobikwikotp`、`amazonpay`、`mobikwikotp_v2`、`freecharge`、`phonepeotp`、`mobikwik_wallet_auto_buy`、`yape`

---

## 2. 基础接口

### 2.1 获取钱包列表

```
POST /saasBankList
```

**请求参数**：无（请求体传 `{}`）。

**响应**：`SaasBankListDto` 数组，仅返回后台启用且类型可识别的钱包。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `name` | String | 钱包名称，如 `Yape` |
| `code` | String | 钱包类型 `typeCode` |
| `type` | String | `otp` 或 `app`，决定后续走 OTP 流程还是绑定流程 |
| `downloadUrl` | String | App 下载地址，没有时为空串 |
| `sell` | Boolean | 是否支持代收 |
| `buy` | Boolean | 是否支持代付 |

**示例**：

```json
{
  "code": 1,
  "message": "success",
  "englishMessage": null,
  "data": [
    {
      "name": "Yape",
      "code": "yape",
      "type": "app",
      "downloadUrl": "/api/consumer/res/download/yape.apk",
      "sell": true,
      "buy": true
    }
  ]
}
```

---

### 2.2 发送 OTP

```
POST /saasOtpSend
```

适用 `otpType` 列表中的钱包。

**请求参数**：

| 字段 | 类型 | 必传 | 说明 |
| --- | --- | --- | --- |
| `phone` | String | 是 | 10 位印度手机号；带 `+91` 前缀会自动去掉 |
| `typeCode` | String | 是 | 必须是 1.7 中 `otpType` 的类型 |
| `password` | String | 条件 | `paytmbusiness` 必传 |
| `ua` | String | 条件 | `phonepebusiness` 必传 |
| `token` | String | 条件 | `phonepebusiness` 必传 |
| `proxyIp` | String | 否 | 指定出口代理 |
| `proxyPort` | Integer | 否 | 代理端口 |
| `proxyUsername` | String | 否 | 代理用户名 |
| `proxyPassword` | String | 否 | 代理密码 |
| `sendWhatApps` | Boolean | 否 | 是否通过 WhatsApp 下发，默认 `false` |

**响应**：`SaasOtpSendRespDto`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | String | 本次 OTP 会话 ID，校验时原样回传 |
| `timestamp` | Long | 发送时间戳（毫秒） |
| `expireIn` | Long | 有效期 |
| `message` | String | 附加提示 |

**示例**：

```json
{ "phone": "9876543210", "typeCode": "mobikwikotp", "sendWhatApps": false }
```

```json
{
  "code": 1,
  "message": "success",
  "englishMessage": null,
  "data": {
    "requestId": "66f1c2a0e4b0a1b2c3d4e5f6",
    "timestamp": 1783420800000,
    "expireIn": 300,
    "message": "OTP sent"
  }
}
```

**注意**：

- 同一 `phone + typeCode` 在 60 秒内重复调用会**直接返回缓存结果**（`requestId` 不变），不会重复下发短信。
- 如果账号已存在但状态不是启用，会先清空其绑定关系再重新走 OTP 流程。

---

### 2.3 校验 OTP

```
POST /saasOtpVerity
```

**请求参数**：

| 字段 | 类型 | 必传 | 说明 |
| --- | --- | --- | --- |
| `otp` | String | 是 | 用户收到的验证码 |
| `requestId` | String | 是 | 2.2 返回的 `requestId` |
| `email` | Boolean | 否 | 是否走邮箱验证码，默认 `false`；也接受 `isEmail` 这个字段名 |

**响应**：`Boolean`，`true` 表示校验通过。

**示例**：

```json
{ "otp": "123456", "requestId": "66f1c2a0e4b0a1b2c3d4e5f6", "email": false }
```

**注意**：账号已经是启用状态且已绑定当前用户时，直接返回 `true`，不再调用上游校验。

---

### 2.4 查询收款地址

```
POST /saasVpa
```

**请求参数**：

| 字段 | 类型 | 必传 | 说明 |
| --- | --- | --- | --- |
| `phone` | String | 是 | 账号手机号 |
| `typeCode` | String | 是 | 钱包类型 |

**响应**：`UpiDto` 数组

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `pa` | String | 收款地址（UPI ID）。**Yape 没有 UPI，返回账号手机号** |
| `pn` | String | 收款人姓名 |
| `isAdmin` | Boolean | 是否为管理员账号 |

**示例**：

```json
{ "phone": "987654321", "typeCode": "yape" }
```

```json
{
  "code": 1,
  "message": "success",
  "englishMessage": null,
  "data": [
    { "pa": "987654321", "pn": "Juan Perez", "isAdmin": false }
  ]
}
```

**注意**：

- 账号必须已绑定到当前 API Key 所属用户，否则返回 `account not found`。
- 结果缓存 10 秒；同一账号并发请求会串行执行。
- Yape 会先查一次余额做探活，登录态失效（`-50200`）时会把账号置为封禁并返回 `account unauthorized`。

---

### 2.5 查询账单

```
POST /saasGetTrans
```

**请求参数**：

| 字段 | 类型 | 必传 | 说明 |
| --- | --- | --- | --- |
| `startTime` | Long | 是 | 开始时间（毫秒时间戳），闭区间 |
| `endTime` | Long | 是 | 结束时间（毫秒时间戳），闭区间 |
| `phone` | String | 是 | 账号手机号 |
| `typeCode` | String | 是 | 钱包类型 |
| `page` | Integer | 否 | 页码，从 1 开始，默认 `1` |
| `pageSize` | Integer | 否 | 每页条数，默认 `20` |

**响应**：`PageResult<HistoryTradeVo>`，`data` 内每条记录字段如下：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `_id` | String | 账单记录 ID |
| `type` | String | `SENT_PAYMENT`（支出）/ `RECEIVED_PAYMENT`（收入） |
| `phone` | String | 钱包账号手机号 |
| `payee` | String | 收款方 |
| `payer` | String | 付款方 |
| `amount` | String | 金额，两位小数字符串 |
| `currency` | String | 币种，Yape 为 `PEN` |
| `status` | String | `SUCCESS` / `PENDING` / `FAILED` 等 |
| `additionalDetail` | String | 附言/备注 |
| `txnDate` | Long | 交易时间（毫秒时间戳） |
| `txnId` | String | 交易号 |
| `utr` | String | 银行流水号；没有时是空串 |
| `bindUserID` | String | 固定空串（不对外暴露归属） |
| `error` | String | 失败原因 |
| `manual` | String | 人工标记 |
| `accountType` | String | 钱包类型 `typeCode` |
| `createTime` | String | 入库时间，`yyyy-MM-dd'T'HH:mm:ss.SSS'Z'` |
| `billType` | String | 账单分类 |
| `sourceFrom` | String | `balance`（余额）/ `bank`（银行流水） |

**示例**：

```json
{
  "startTime": 1782403200000,
  "endTime": 1783420800000,
  "phone": "987654321",
  "typeCode": "yape",
  "page": 1,
  "pageSize": 20
}
```

```json
{
  "code": 1,
  "message": "success",
  "englishMessage": null,
  "data": {
    "pageNum": 1,
    "pageSize": 20,
    "total": 1,
    "pages": 1,
    "data": [
      {
        "_id": "66f1c2a0e4b0a1b2c3d4e5f6",
        "type": "RECEIVED_PAYMENT",
        "phone": "987654321",
        "payee": "987654321",
        "payer": "912345678",
        "amount": "100.00",
        "currency": "PEN",
        "status": "SUCCESS",
        "additionalDetail": "Pago de prueba",
        "txnDate": 1783405205000,
        "txnId": "0123456",
        "utr": "0123456",
        "bindUserID": "",
        "error": null,
        "manual": null,
        "accountType": "yape",
        "createTime": "2026-11-05T10:20:05.000Z",
        "billType": null,
        "sourceFrom": "bank"
      }
    ]
  }
}
```

**注意**：

- 查询前会先增量同步一次该账号的最新账单（受账号创建时间、最近 30 天、`startTime` 三者较晚者限制），再查本地库，所以首次调用可能稍慢。
- **Yape 服务端只保留最近 3 个月流水**，同步也只回溯 3 个自然月，更早的数据查不到。
- `type` 只返回收支两类；`SUCCESS` 但没有 `utr` 的记录会显示为 `PENDING`；`utr` 为 `no utr` / `history utr` 时统一返回空串。
- 同一账号的同步有分布式锁，并发调用会等待，最长 60 秒。

---

### 2.6 解绑钱包

```
POST /saasWalletLogout
```

**请求参数**：

| 字段 | 类型 | 必传 | 说明 |
| --- | --- | --- | --- |
| `phone` | String | 是 | 账号手机号 |
| `typeCode` | String | 是 | 钱包类型 |

**响应**：无 `data`（`data: null`）。

**示例**：

```json
{ "phone": "987654321", "typeCode": "yape" }
```

**注意**：

- 只解绑**当前用户**名下的账号：`bindUserID` 清空、状态重置为 `INIT`，同时把用户 ID 记入历史绑定列表。
- 账号已绑定其他用户时不会报错，但也不会解绑（匹配不到记录）。同一账号重复解绑返回成功。
- Yape 手机号会先做归一化（支持 `+51` 等前缀）。

---

## 3. 应用钱包

### 3.1 绑定钱包

```
POST /saasBindApp
```

适用 `appType` 列表中的钱包（`saasOtpSend` 走 OTP 流程，本接口走账号绑定）。

**请求参数**：

| 字段 | 类型 | 必传 | 说明 |
| --- | --- | --- | --- |
| `phone` | String | 是 | 账号手机号。Yape 为 8 位，可带 `+51`/`51`/`00591`/`0051` |
| `typeCode` | String | 是 | 钱包类型；`mobikwik_wallet_auto_buy` 找不到账号时会回退到 `mobikwik` |
| `proxyIp` | String | 否 | 指定出口代理，绑定成功后会落库 |
| `proxyPort` | Integer | 否 | 代理端口 |
| `proxyUsername` | String | 否 | 代理用户名 |
| `proxyPassword` | String | 否 | 代理密码 |
| `picUrl` | String | 条件 | **Yape 必传（与 `picBase64` 二选一）**。头像图片地址 |
| `picBase64` | String | 条件 | **Yape 必传（与 `picUrl` 二选一）**。头像图片 base64，可带 `data:image/png;base64,` 前缀 |

**响应**：`Boolean`，`true` 表示绑定成功。

**示例（Yape，头像用 base64）**：

```json
{
  "phone": "987654321",
  "typeCode": "yape",
  "picBase64": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA..."
}
```

**注意**：

- 账号必须已存在（先由运营/后台导入），否则返回 `account not found`。
- 账号已绑定其他用户时返回 `account bind other user`。
- **Yape 头像规则**：
  - `picUrl` 与 `picBase64` 至少传一个，都传时以 `picUrl` 为准；
  - 图片最大 5 MB，`picUrl` 下载超时 10 秒；
  - 保存路径为服务端 `resBak/yapepic/{手机号}.png`，统一转成 PNG 落盘（无法解码的图片原样写入）；
  - 头像会在登录校验之前保存，所以头像有问题时不会产生绑定。
- 绑定时会用解析到的代理重新登录一次，既能校验账号数据，也能确认代理可用；登录失败返回 `login failed`。

---

### 3.2 查询余额

```
POST /saasGetBal
```

**请求参数**：

| 字段 | 类型 | 必传 | 说明 |
| --- | --- | --- | --- |
| `phone` | String | 是 | 账号手机号 |
| `typeCode` | String | 是 | 钱包类型 |

**响应**：`SaasGetBalanceRespDto`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `amount` | BigDecimal | 当前余额 |
| `lastUpdateTimestamp` | Long | 余额对应的时间戳（毫秒） |

**示例**：

```json
{ "phone": "987654321", "typeCode": "yape" }
```

```json
{
  "code": 1,
  "message": "success",
  "englishMessage": null,
  "data": { "amount": 1520.50, "lastUpdateTimestamp": 1783420800000 }
}
```

**注意**：

- 账号必须已绑定当前用户，否则 `account not found`。
- Yape 余额缓存 10 分钟，缓存命中时 `lastUpdateTimestamp` 是缓存写入时间，不是本次查询时间。
- 上游返回余额异常时返回 `GET Balance ERROR`。

---

### 3.3 发起代付

```
POST /saasPay
```

**请求参数**：

| 字段 | 类型 | 必传 | 说明 |
| --- | --- | --- | --- |
| `phone` | String | 是 | **付款方**账号手机号（从哪个钱包出款） |
| `typeCode` | String | 是 | 钱包类型 |
| `payType` | String | 是 | `vpa` 或 `bank` |
| `vpa` | String | 条件 | `payType=vpa` 必传，收款方收款地址。**Yape 传收款方手机号** |
| `bankAccount` | String | 条件 | `payType=bank` 必传，收款银行账号 |
| `ifsc` | String | 条件 | `payType=bank` 必传，银行代码。Yape 传 `YapeUtils` 银行表中的代码，如 `1005`（BANCO DE CREDITO / BCP） |
| `payeeName` | String | 条件 | 收款人全名。**Yape 银行代付必传**；其他钱包可忽略 |
| `amount` | String | 是 | 金额，必须大于 0；Yape 会四舍五入到两位小数 |

**响应**：`SaasPayRespDto`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `txnId` | String | 上游交易号 |

**示例（Yape 代付到银行卡）**：

```json
{
  "phone": "987654321",
  "typeCode": "yape",
  "payType": "bank",
  "bankAccount": "1234567890",
  "ifsc": "1005",
  "payeeName": "Juan Perez",
  "amount": "100.00"
}
```

```json
{
  "code": 1,
  "message": "success",
  "englishMessage": null,
  "data": { "txnId": "a1b2c3d4-0000-0000-0000-000000000001" }
}
```

**注意**：

- 账号必须已绑定当前用户，否则 `account not found`。
- Yape `payType=vpa` 时会先用收款手机号查询收款人全名，查不到则失败（`validateAccount failed, target name is empty`）。
- **超时/网络异常的语义**：如果转账请求已经发出但没拿到结果，返回
  `Payment is being processed by the bank.`。此时**不要直接重试**，应通过 2.5 查询账单或人工对账确认结果，
  避免重复出款。转账未发出时的网络问题返回 `network error`。
- 上游明确失败时返回 `third pay error` 或上游原始错误信息。

---

## 4. 订单

### 4.1 补单

```
POST /submitTrade
```

把一笔外部订单的凭证补录到服务端，按 `txnId + utr + userID` 做 upsert（重复提交幂等）。

**请求参数**：

| 字段 | 类型 | 必传 | 说明 |
| --- | --- | --- | --- |
| `txnId` | String | 是 | 订单号 |
| `utr` | String | 是 | 银行流水号/凭证号 |
| `txnTime` | Long | 是 | 交易时间（毫秒时间戳） |
| `amount` | String | 是 | 金额 |
| `phone` | String | 是 | 相关账号手机号 |
| `typeCode` | String | 是 | 钱包类型 |

**响应**：无 `data`（`data: null`）。

**示例**：

```json
{
  "txnId": "P20261107154012345FXN",
  "utr": "0123456789",
  "txnTime": 1783420800000,
  "amount": "100.00",
  "phone": "987654321",
  "typeCode": "yape"
}
```

**注意**：同一 `txnId + utr` 重复提交只会覆盖更新 `phone`/`amount`/`txnTime`/`typeCode`，不会产生重复订单。

---

## 5. 接入检查清单

1. 请求统一 `POST`，`Content-Type: application/json`，请求体必须是合法 JSON 对象（无参数也传 `{}`）。
2. 每次请求都带上 `X-API-KEY` 和按 1.3 计算的 `X-SIGN`；签名只覆盖请求体，值建议全部用字符串。
3. 先调 `/saasBankList` 拿到 `typeCode` 与 `type`，OTP 类走 `/saasOtpSend` + `/saasOtpVerity`，应用类走 `/saasBindApp`。
4. 绑定成功后再调 `/saasVpa`、`/saasGetBal`、`/saasPay`、`/saasGetTrans`。
5. 代付遇到 `Payment is being processed by the bank.` 不要重试，先查账单或人工确认。
6. 判断结果只看 `code`（`1` 为成功），HTTP 状态码不代表业务结果。

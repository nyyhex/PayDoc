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
| `depositCallbackUrl` | 代收回调地址 |
| `payoutCallbackUrl` | 代付回调地址 |

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
| OTP 类（`otpType`） | `mobikwikotp_v2`、`mobikwikotp`、`phonepebusiness`、`paytmbusiness`、`amazonpay`、`airtel`、`freecharge`、`phonepeotp`、`bharotp`、`yapeotp` |
| 应用类（`appType`） | `mobikwik`、`phonepe`、`paytm`、`bhar`、`mobikwik_wallet_auto_buy`、`yape` |

- 支持代收（`sell`）：`bhar`、`paytm`、`phonepe`、`freecharge`、`mobikwik`、`mobikwikotp_v2`、`mobikwikotp`、`airtel`、`amazonpay`、`phonepebusiness`、`paytmbusiness`、`phonepeotp`、`bharotp`、`mobikwik_wallet_auto_buy`、`yape`、`yapeotp`
- 支持代付（`buy`）：`paytm`、`phonepe`、`mobikwik`、`mobikwikotp`、`amazonpay`、`mobikwikotp_v2`、`freecharge`、`phonepeotp`、`mobikwik_wallet_auto_buy`、`yape`、`yapeotp`

> `yape` 与 `yapeotp` 是同一钱包（Yape，秘鲁）的两种接入方式：`yape` 走 `saasBindApp` 绑定，
> `yapeotp` 走 `saasOtpSend` + `saasOtpVerity`（设备绑定 + 人脸 + 短信验证码），
> 绑定完成后的余额、账单、收款地址、代付、头像接口完全一致，详见 2.7。

> 接口参数里的 `userID` 一律被忽略，服务端用 API Key 绑定的用户强制覆盖，无需传递。

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
| `phone` | String | 是 | 10 位印度手机号；带 `+91` 前缀会自动去掉。**`yapeotp` 例外：8 位本地号，可带 `+51`/`51`/`00591`/`0051`，服务端截取后 8 位** |
| `typeCode` | String | 是 | 必须是 1.7 中 `otpType` 的类型 |
| `password` | String | 条件 | `paytmbusiness` 必传；**`yapeotp` 传 Yape 登录 PIN，必传** |
| `ua` | String | 条件 | `phonepebusiness` 必传 |
| `token` | String | 条件 | `phonepebusiness` 必传 |
| `picBase64` | String | 条件 | **仅 `yapeotp` 使用**：人脸照片 base64（可带 `data:image/jpeg;base64,` 前缀，最大 5 MB）。新设备首次绑定必须过人脸；不传则读取 `3.2 上传头像` 落盘的照片 |
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
{ "phone": "987654321", "typeCode": "yapeotp", "password": "123456" }
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
- `yapeotp` 的完整流程（设备注册 → 人脸 → 短信）见 2.7。

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

**注意**：

- 账号已经是启用状态且已绑定当前用户时，直接返回 `true`，不再调用上游校验。
- `yapeotp`：验证码错误或超过重试次数返回 `incorrect otp`；校验通过后账号状态变为启用并绑定到当前用户。
  如果 2.2 已经完成设备关联（例如该设备此前已绑过，`message` 为 `account already logged in, no otp required`），
  本接口会直接返回 `true`。

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
| `pa` | String | 收款地址（UPI ID）。**Yape / Yape OTP 没有 UPI，返回账号手机号** |
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
- Yape / Yape OTP 会先查一次余额做探活，登录态失效（`-50200`）时会把账号置为封禁并返回 `account unauthorized`。

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
- Yape / Yape OTP 手机号会先做归一化（支持 `+51` 等前缀）。

---

### 2.7 Yape OTP 设备绑定流程（`yapeotp`）

`yapeotp` 与 `yape` 是同一个 Yape 钱包，区别只在**登录信息的获取方式**：`yape` 由 App 上报加密登录数据后调
`3.1 绑定钱包`；`yapeotp` 完全由服务端通过网关的设备绑定接口完成，需要账号 **手机号 + PIN**，新设备还要再过一次**人脸**。

对调用方来说，流程与普通 OTP 钱包一致，只是多两件事：**带 PIN**、**准备人脸照片**。

**步骤**

1. **准备人脸照片**（可选，但新设备首次绑定必需）
   - 调 `3.2 上传头像`（`typeCode=yapeotp`）把照片存到服务端，或
   - 直接在 `2.2 发送 OTP` 里带 `picBase64`。
2. **发送 OTP** `POST /saasOtpSend`
   ```json
   { "phone": "987654321", "typeCode": "yapeotp", "password": "123456" }
   ```
   服务端依次执行：设备注册 → （新设备）人脸校验并自动下发短信 → （人脸过了但短信没发出）自动重发短信。
   返回的 `requestId` 用于下一步，`expireIn` 为 `60`。
   - `message = "account already logged in, no otp required"`：该设备已关联过，登录已完成，**无需再校验 OTP**
     （仍可调用 2.3，会直接返回 `true`）。
3. **校验 OTP** `POST /saasOtpVerity`
   ```json
   { "otp": "123456", "requestId": "66f1c2a0e4b0a1b2c3d4e5f6" }
   ```
   校验通过返回 `true`，账号变为启用并绑定到当前用户；之后即可使用 2.4 / 2.5 / 3.4 / 3.5 等接口。

**常见错误**

| 错误信息 | 说明 |
| --- | --- |
| `pin is required` | 未传 `password`（Yape PIN） |
| `yape phone must be 8 digits` | 手机号不是 8 位且无法按国家码归一化 |
| `selfie is required, please upload the face picture first` | 新设备需要人脸，但既没传 `picBase64` 也没有已上传的照片 |
| `account is binding user ID.` | 该账号已绑定其他用户，不能重复走 OTP 绑定 |
| `Proxy is null. Unable to submit.` | 未指定代理且代理池取不到可用代理 |
| 网关返回的错误标题（如 `We couldn't find your account`） | 上游失败，原文透出，通常是手机号/PIN 错误 |
| `incorrect otp` | 验证码错误或超过重试次数 |
| `network error` | 网络异常，可重试 |

**注意**：

- 同一 `phone + typeCode` 有 60 秒结果缓存，成功后 60 秒内重复调 `saasOtpSend` 只会返回缓存的 `requestId`，
  不会真的重新请求网关；发码失败不会写缓存，失败后可直接重试。
- 设备档案和会话都会落库：再次调 `saasOtpSend` 时会带上已有设备信息重新注册，网关认得这台设备时
  （已关联完）直接返回登录态，`message` 为 `account already logged in, no otp required`；仍需要人脸时才会重做人脸校验。
  如果人脸通过但短信下发失败，同一次调用内会自动重发短信，不需要调用方重刷人脸。
- `yapeotp` 与 `yape` 各自独立建账号（同一手机号可以同时存在两个类型的账号），互不影响。

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

**响应**：`Boolean`，`true` 表示绑定成功。

**示例**：

```json
{
  "phone": "987654321",
  "typeCode": "yape"
}
```

**注意**：

- 账号必须已存在（先由运营/后台导入），否则返回 `account not found`。
- 账号已绑定其他用户时返回 `account bind other user`。
- 本接口不处理头像；需要更新 Yape 头像请调 `3.2 上传头像`。
- 绑定时会用解析到的代理重新登录一次，既能校验账号数据，也能确认代理可用；登录失败返回 `login failed`。
- `yapeotp` **不走本接口**，请使用 2.2 + 2.3 的 OTP 流程（见 2.7）。

---

### 3.2 上传头像

```
POST /saasUploadPic
```

单独上传账号头像，与绑定流程解耦：**绑定前、绑定后都可以调用**，已存在的头像会被新图覆盖。
支持 `typeCode=yape` 与 `typeCode=yapeotp`；对 `yapeotp` 来说这张图同时作为 `2.2 发送 OTP` 时的人脸照片。
**不校验账号是否存在/归属**，按归一化后的手机号直接落盘。

**请求参数**：

| 字段 | 类型 | 必传 | 说明 |
| --- | --- | --- | --- |
| `phone` | String | 是 | Yape 手机号，8 位；可带 `+51`/`51`/`00591`/`0051` 前缀，服务端截取后 8 位 |
| `typeCode` | String | 是 | `yape` 或 `yapeotp`，其他类型返回 `xxx not support upload pic` |
| `picUrl` | String | 条件 | 图片地址，与 `picBase64` 二选一必传 |
| `picBase64` | String | 条件 | 图片 base64，可带 `data:image/png;base64,` 前缀，与 `picUrl` 二选一必传 |

**响应**：无 `data`（`data: null`）。

**示例**：

```json
{
  "phone": "987654321",
  "typeCode": "yape",
  "picBase64": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA..."
}
```

```json
{ "code": 1, "message": "success", "englishMessage": null, "data": null }
```

**注意**：

- `picUrl` 与 `picBase64` 都传时以 `picUrl` 为准；两个都不传返回 `picUrl or picBase64 is required`。
- 规则：最大 5 MB，`picUrl` 下载超时 10 秒，统一转成 JPEG 存到 `resBak/yapepic/{手机号}.jpg`（透明区域铺白底），图片无法解码返回 `pic is not a valid image`。
- 不校验账号是否存在、是否绑定其他用户：只要 `phone`/`typeCode` 合法就落盘，可提前为未导入的账号准备头像。
- `yape` 与 `yapeotp` 共用同一张照片（按手机号存 `resBak/yapepic/{手机号}.jpg`）。
- 覆盖是「临时文件 + 原子替换」，不会因为编码失败留下半张旧图。

---

### 3.3 查询余额

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
- Yape / Yape OTP 余额缓存 10 分钟，缓存命中时 `lastUpdateTimestamp` 是缓存写入时间，不是本次查询时间。
- 上游返回余额异常时返回 `GET Balance ERROR`。

---

### 3.4 发起代付

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
| `ifsc` | String | 条件 | `payType=bank` 必传，银行代码。Yape / Yape OTP 传 3.6 里的 `bank_code`，如 `1005`（BANCO DE CREDITO / BCP）；服务端会按代码自动补上银行名称与缩写，未知代码返回 `unknown code_bank xxx, see bankName(code)` |
| `payeeName` | String | 条件 | 收款人全名。**Yape / Yape OTP 银行代付必传**；其他钱包可忽略 |
| `amount` | String | 是 | 金额，必须大于 0；Yape / Yape OTP 会四舍五入到两位小数 |
| `sourceOrigin` | String | 否 | 资金来源说明，PCC 要求时与 sourceDestination 一起补传（目前 Yape 使用） |
| `sourceDestination` | String | 否 | 资金去向说明，PCC 要求时与 sourceOrigin 一起补传（目前 Yape 使用） |

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
- Yape / Yape OTP `payType=vpa` 时会先用收款手机号查询收款人全名，查不到则失败（`validateAccount failed, target name is empty`）。
- **超时/网络异常的语义**：如果转账请求已经发出但没拿到结果，返回
  `Payment is being processed by the bank.`。此时**不要直接重试**，应通过 2.5 查询账单或人工对账确认结果，
  避免重复出款。转账未发出时的网络问题返回 `network error`。
- 上游明确失败时返回 `third pay error` 或上游原始错误信息。

---

### 3.5 Yape 银行代码（`ifsc`）

`payType=bank` 且 `typeCode=yape` 或 `yapeotp` 时，`ifsc` 传下表的 `bank_code`。服务端按代码查出银行名称与缩写，
作为 `bank_name` / `bank_acronym` 提交给 Yape；**代码必须与下表完全一致**（区分大小写，含 `MLD` 前缀），
不在表中的代码会返回 `unknown code_bank <code>, see bankName(code)`。

| bank_code | 银行名称 | 缩写 |
| --- | --- | --- |
| `1001` | BANCO NACIONAL DE BOLIVIA | BNB |
| `1003` | BANCO MERCANTIL | BME |
| `1005` | BANCO DE CREDITO | BCP |
| `1007` | BANCO DE LA NACION ARGENTINA | BNA |
| `1009` | BANCO BISA | BIS |
| `1014` | BANCO UNION | BUN |
| `1016` | BANCO ECONOMICO | BEC |
| `1017` | BANCO SOL | BSOL |
| `1018` | BANCO GANADERO | BGA |
| `1033` | BANCO FIE S.A. | BIE |
| `1034` | BANCO FORTALEZA | FOR |
| `25009` | PANAMERICAN SAFI | SPA |
| `27002` | CIDRE IFD | ICI |
| `27003` | CRECER IFD | ICR |
| `27004` | Diaconía IFD | DIA |
| `27007` | FUNDACIÓN BOLIVIANA PARA EL DESARROLLO IFD | IFU |
| `27009` | IDEPRO IFD | IID |
| `27012` | Pro Mujer IFD. | PMIFD |
| `3001` | COOPERATIVA JESUS NAZARENO | CJN |
| `3002` | COOPERATIVA SAN MARTIN DE PORRES | CSM |
| `3003` | COOPERATIVA FÁTIMA | CFA |
| `3004` | COOPERATIVA LA MERCED | CLM |
| `3011` | Cooperativa Pio X R.L. | CPX |
| `3015` | COOPERATIVA QUILLACOLLO | CQC |
| `3016` | COOPERATIVA SAN JOSE DE PUNATA | CJP |
| `3025` | COOPERATIVA SAN MATEO | CST |
| `3029` | COOPERATIVA EDUCADORES GRAN CHACO R.L. | CEC |
| `3044` | COOPERATIVA CACEF | CCF |
| `3048` | Cooperativa Magisterio Rural de Chuquisaca | CMD |
| `51001` | LINKSER S.A. | LNK |
| `51002` | ADMINISTRADORA DE TARJETAS DE CREDITO S.A. | ATC |
| `53001` | EFECTIVO S.A | MEF |
| `74002` | BANCO PYME ECOFUTURO S.A. | BECOF |
| `74003` | BANCO PYME DE LA COMUNIDAD | PCO |
| `75003` | LA PROMOTORA EFV | VPR |
| `MLD10000` | BANCO CENTRAL DE BOLIVIA - MLD | BCBMLD |
| `MLD1004` | BANCO CENTRAL DE BOLIVIA - COMPRA DE BONOS | BCB |
| `MLD3005` | COOP SAN PEDRO LTDA. | CSP |
| `MLD3006` | COOPERATIVA LOYOLA R.L. | CAL |
| `MLD3007` | Cooperativa Catedral Tarija | 3007 |
| `MLD3010` | COOPERATIVA SAN ANTONIO R.L. | CSA |
| `MLD3012` | COOP INCA HUASI LTDA. | CIH |
| `MLD3021` | COOP TRINIDAD LTDA | CTR |
| `MLD3022` | COOP COMARAPA R.L. | CCM |
| `MLD3024` | COOPERATIVA SAN ROQUE R.L. | 3024 |
| `MLD3026` | EL CHOROLQUE R.L. | CCR |
| `MLD3027` | Cooperativa Monsenor Felix Gainza R.L. | 3027 |
| `MLD3028` | COOP MADRE Y MAESTRA LTDA. | CMM |
| `MLD3030` | COOPERATIVA CATEDRAL R.L. | CCP |
| `MLD3031` | Cooperativa Asunción R.L. | MLD |
| `MLD3033` | COOP SAN JOSE DE BERMEJO LTDA | CJB |
| `MLD3034` | Cooperativa Magisterio Rural | MLD303 |
| `MLD3036` | COOPERATIVA SAN JOAQUIN R.L. | MCJO |
| `MLD3043` | COOPERATIVA SAN CARLOS BORROMEO R.L. | CCB |
| `MLD3046` | COOPERATIVA LA SAGRADA | CLS |
| `MLD3047` | COOP PROGRESO R.L. | CPG |
| `MLD3049` | COOPERATIVA SAN FRANCISCO SOLANO S.R.L | CFS |
| `MLD3051` | COOPERATIVA SAN PEDRO DE AIQUILE R.L. | CSQ |
| `MLD3052` | Cooperativa Societaria San Martín | MLD305 |
| `MLD3054` | COOPERATIVA VIRGEN DE LOS REMEDIOS R.L. | CVE |
| `MLD3055` | COOPERATIVA ASOCIACIÓN DE PRODUCTORES DE LECHE R.L. | COO |
| `MLD3059` | COOPERATIVA HOSPICIO R.L. | 3059 |
| `MLD75001` | LA PRIMERA ENTIDAD FINANCIERA DE VIVIENDA | VL1 |
| `MLD75004` | EL PROGRESO ENTIDAD FINANCIERA DE VIVIENDA | VPG |

> 简称列里有少数值是纯数字（如 `3007`、`3024`），这是上游银行表里的原始取值，按原样提交。

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
   `yapeotp` 属于 OTP 类，发码时要带 PIN（`password`）和人脸照片（`picBase64` 或先调 `/saasUploadPic`），详见 2.7。
4. 绑定成功后再调 `/saasVpa`、`/saasGetBal`、`/saasPay`、`/saasGetTrans`；头像需要单独更新时调 `/saasUploadPic`。
5. 代付遇到 `Payment is being processed by the bank.` 不要重试，先查账单或人工确认。
6. 判断结果只看 `code`（`1` 为成功），HTTP 状态码不代表业务结果。

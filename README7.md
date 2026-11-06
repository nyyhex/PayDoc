# PayPal 开放接口文档

本文档面向对接商家，说明平台开放接口的接入方式、签名规则、防重放要求、返回结构与各接口的字段定义。

- 接口版本：`v1`
- 请求方式：全部为 `POST`，`Content-Type: application/json`
- 基础路径：`/api/open`
- 响应结构：统一为 `code` / `message` / `tm` / `data`，业务数据在 `data.result` 中
- **重点：签名算法与防重放规则见第 2、3 节，签名不通过是接入初期最常见的问题**

---

## 目录

1. [接入准备](#1-接入准备)
2. [签名规则](#2-签名规则)
3. [防重放与重试](#3-防重放与重试)
4. [统一响应结构](#4-统一响应结构)
5. [接口总览](#5-接口总览)
6. [创建支付订单](#6-创建支付订单)
7. [查询支付订单](#7-查询支付订单)
8. [创建订阅订单](#8-创建订阅订单)
9. [查询订阅订单](#9-查询订阅订单)
10. [取消订阅](#10-取消订阅)
11. [查询余额](#11-查询余额)
12. [订单异步通知](#12-订单异步通知)
13. [状态枚举与格式约定](#13-状态枚举与格式约定)
14. [常见问题排查](#14-常见问题排查)

---

## 1. 接入准备

平台为每个商户分配以下参数，请妥善保管，**apiSecret 只用于本地签名，不要在客户端（App/浏览器）中暴露**。

| 参数 | 说明 |
|------|------|
| `merchant_no` | 商户编号，即商户的 `userID`，所有接口都必须传，且必须与 API Key 所属商户一致 |
| `X-API-KEY` | 调用凭据，放在请求头中 |
| `apiSecret` | 签名密钥，只用于计算签名，不随请求传输 |
| IP 白名单 | 在平台后台的 API Key 上配置，支持 IPv4/IPv6 地址与网段（如 `1.2.3.4`、`10.0.0.0/8`、`2606:4700::/32`），多个用英文逗号分隔 |
| 代收回调地址 | 在 API Key 上配置；创建支付订单未传 `notify_url` 时，平台使用该地址通知商户 |

### 1.1 调用来源限制

开放接口只接受**公网地址**发起的调用，请务必注意：

- 商户服务器需要以公网出口 IP 调用平台接口，平台不接受来自内网/局域网地址的请求（返回 `401`）。
- 若商户服务器位于 NAT/反向代理之后，请确认平台侧能看到的是公网出口 IP，并把它加入 IP 白名单。
- IP 白名单留空或配置为 `*` 表示不限制来源 IP；配置了具体地址后，不在名单内的请求会被拒绝。

### 1.2 频率限制

- 按来源 IP 限流：令牌桶容量 500，每秒补充 100 个令牌，即允许短时突发，长期平均不超过 **100 次/秒**。
- 超限时返回 **HTTP 429**，响应体是纯文本 `Too many requests, please slow down.`（不是 JSON），请做退避重试。

---

## 2. 签名规则

所有接口都必须签名。签名算法为 **HMAC-SHA256**：把拼好的待签串当作待签名消息，**apiSecret 当作 HMAC 密钥**，结果输出 **64 位大写十六进制**。

> **与旧版算法的差异：** 旧版是「待签串末尾追加 apiSecret 后取 MD5」，输出 32 位大写十六进制。新版改为 HMAC-SHA256：
> 待签串**不再追加 apiSecret**，apiSecret 只作为密钥，签名长度变为 **64 位**。升级时只需替换加密这一段，参数筛选、排序与拼接规则完全不变。
> 平台只认 HMAC-SHA256，继续用旧算法会返回 `Sign is invalid`。

### 2.1 计算步骤

1. 取请求体中的**全部参数**，剔除 `sign` 与 `sign_type`（平台不使用 `sign_type`，传了也不参与签名）。
2. 剔除值为 `null`、空串、纯空白的参数（`"0"`、`"0.00"` 这类值不剔除）。
3. 参数名按 **ASCII 字典序升序**排列（参数名均为英文小写加下划线）。
4. 拼接成 `k1=v1&k2=v2&...` 的形式，**不做 URL 编码**，参数之间用 `&` 连接。**这个拼接结果后面不再追加 apiSecret。**
5. 以 **apiSecret 为密钥**，对上一步的字符串按 UTF-8 做 **HMAC-SHA256**。
6. 输出 **64 位大写十六进制**。

### 2.2 传递方式

签名可以放在请求体的 `sign` 字段中（本文档示例均用这种方式），也可以放在请求头 `X-SIGN` 中：

- 两者都传时，以请求头 `X-SIGN` 为准。
- 平台在校验时忽略大小写，但请统一使用大写。

> 签名只覆盖**请求体参数**，不包含 URL、请求头中的 `X-API-KEY` 等。因此修改请求体任何一个字段都必须重新签名。

### 2.3 完整算例

请求体（创建支付订单）：

```json
{
  "merchant_no": "M10001",
  "trade_no": "PO20261106001",
  "amount": "100.00",
  "currency": "USD",
  "notify_url": "https://merchant.example.com/paypal/notify",
  "member_name": "user_8899",
  "timestamp": "1762339200"
}
```

假设 apiSecret 为 `sk_live_5f3b9c2d7e1a4b6f`（示例值）：

```text
# 第 1~3 步：剔除 sign/sign_type、剔除空值、按键名升序
amount=100.00
currency=USD
member_name=user_8899
merchant_no=M10001
notify_url=https://merchant.example.com/paypal/notify
timestamp=1762339200
trade_no=PO20261106001

# 第 4 步：用 & 连接, 不再追加 apiSecret
amount=100.00&currency=USD&member_name=user_8899&merchant_no=M10001&notify_url=https://merchant.example.com/paypal/notify&timestamp=1762339200&trade_no=PO20261106001

# 第 5~6 步：以 apiSecret 为密钥做 HMAC-SHA256, 输出大写十六进制
3AB150E0DD64EEF788F99C6E4D17F180E7DB7E2A9B76D95CB190ACB4E97F388E
```

最终请求体为：

```json
{
  "merchant_no": "M10001",
  "trade_no": "PO20261106001",
  "amount": "100.00",
  "currency": "USD",
  "notify_url": "https://merchant.example.com/paypal/notify",
  "member_name": "user_8899",
  "timestamp": "1762339200",
  "sign": "3AB150E0DD64EEF788F99C6E4D17F180E7DB7E2A9B76D95CB190ACB4E97F388E"
}
```

### 2.4 各语言示例

**Java**

```java
public static String sign(Map<String, String> params, String apiSecret) throws Exception {
    Map<String, String> sorted = new TreeMap<>();
    for (Map.Entry<String, String> e : params.entrySet()) {
        String key = e.getKey();
        String value = e.getValue();
        if ("sign".equals(key) || "sign_type".equals(key)) {
            continue;
        }
        if (value == null || value.trim().isEmpty()) {
            continue;
        }
        sorted.put(key, value);
    }
    StringBuilder sb = new StringBuilder();
    for (Map.Entry<String, String> e : sorted.entrySet()) {
        if (sb.length() > 0) {
            sb.append('&');
        }
        sb.append(e.getKey()).append('=').append(e.getValue());
    }
    Mac mac = Mac.getInstance("HmacSHA256");
    mac.init(new SecretKeySpec(apiSecret.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
    byte[] digest = mac.doFinal(sb.toString().getBytes(StandardCharsets.UTF_8));
    StringBuilder hex = new StringBuilder();
    for (byte b : digest) {
        hex.append(String.format("%02X", b));
    }
    return hex.toString();
}
```

**PHP**

```php
function paypalSign(array $params, string $apiSecret): string
{
    unset($params['sign'], $params['sign_type']);
    ksort($params, SORT_STRING);
    $pairs = [];
    foreach ($params as $key => $value) {
        if ($value === null || trim((string)$value) === '') {
            continue;
        }
        $pairs[] = $key . '=' . $value;
    }
    return strtoupper(hash_hmac('sha256', implode('&', $pairs), $apiSecret));
}
```

**Python 3**

```python
import hashlib
import hmac

def paypal_sign(params: dict, api_secret: str) -> str:
    items = sorted(
        (k, v) for k, v in params.items()
        if k not in ('sign', 'sign_type') and v is not None and str(v).strip() != ''
    )
    raw = '&'.join(f'{k}={v}' for k, v in items)
    return hmac.new(api_secret.encode('utf-8'), raw.encode('utf-8'), hashlib.sha256).hexdigest().upper()
```

**Node.js**

```javascript
const crypto = require('crypto')

function paypalSign(params, apiSecret) {
  const raw = Object.keys(params)
    .filter(k => k !== 'sign' && k !== 'sign_type')
    .filter(k => params[k] !== null && params[k] !== undefined && String(params[k]).trim() !== '')
    .sort()
    .map(k => `${k}=${params[k]}`)
    .join('&')
  return crypto.createHmac('sha256', apiSecret).update(raw, 'utf8').digest('hex').toUpperCase()
}
```

### 2.5 签名常见错误

| 现象 | 原因 |
|------|------|
| 签名不通过 | 对拼接结果做了 URL 编码（如把 `://` 编码成 `%3A%2F%2F`），平台不编码 |
| 签名不通过 | 把值为空串的参数也拼进了待签串（平台会跳过空值与纯空白值） |
| 签名不通过 | 参数名大小写、拼写与文档不一致，或漏掉了某个字段 |
| 签名不通过 | apiSecret 前后有多余空格，或用了错误的 API Key 对应的 apiSecret |
| 签名不通过 | 传输时未以字符串传参（例如金额传成数字 `100`），导致参与签名的字符串与平台解析出的不同 |
| 签名不通过 | 把 apiSecret 拼到了待签串末尾（新版算法是 HMAC，apiSecret 只作为密钥，不拼进待签串） |
| 签名不通过 | 用 MD5、HMAC-MD5 或 SHA256 等其它算法（平台为 **HMAC-SHA256**，输出 64 位大写十六进制） |

> **建议：** 所有参数一律按**字符串**传递，尤其是金额与时间戳，避免不同语言对数字序列化格式的差异导致签名对不上。

---

## 3. 防重放与重试

- 平台按签名做一次性去重：**同一个 `sign` 在 10 分钟内只能使用一次**。
- 命中重复时返回 `401`，`message` 为 `Duplicate request, please regenerate timestamp and sign`。
- 去重键是签名本身，且**签名校验一通过就占用该键**：即使这次请求随后因为参数错误返回 `400`，同一个签名也不能再用第二次。
- 因此**任何重试都必须重新生成 `timestamp` 并重新签名**，不能把上一次的请求原样再发一次。
- 网络超时后的重试同理：原样重发会命中去重，重新签名后重发则会正常执行。若上一次其实已经成功，创建类接口会因 `trade_no` 幂等返回原订单，不会重复下单（见 [3.2](#32-幂等)）；查询类接口也可以直接用来确认结果。

### 3.1 时间戳

- `timestamp` 为必传参数，支持 **10 位（秒）** 或 **13 位（毫秒）** 时间戳。
- **平台强制校验时间戳**：与平台服务器时间偏差超过 **5 分钟** 的请求直接拒绝，返回 `400`，`message` 为
  `时间戳 timestamp 已过期: ..., 与服务器时间相差 N 秒, 允许误差5分钟`。格式不对（不是 10 位或 13 位数字）返回
  `时间戳 timestamp 不正确: ..., 请传10位(秒)或13位(毫秒)时间戳`，缺失返回 `时间戳 timestamp 不能为空`。
- 平台返回的 `tm` 是服务器时间戳（秒），可用于校准本地时间。

### 3.2 幂等

- 创建支付订单：`trade_no` 是**商户侧幂等键**，同一商户下 `trade_no` 重复时会直接返回原订单（不会重复扣款）。**即使金额等参数不同，也返回原订单**，请勿复用 `trade_no`。
- 创建订阅订单：同样以 `trade_no` 为幂等键，重复下单返回原订阅。
- 查询类接口天然幂等，可放心重试。

---

## 4. 统一响应结构

```json
{
  "code": 200,
  "message": "success",
  "tm": 1762339201,
  "data": {
    "result": {}
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `code` | int | 业务状态码，**成功为 200** |
| `message` | string | 成功为 `success`；失败为具体原因 |
| `tm` | long | 平台服务器时间戳（**秒**） |
| `data` | object | 成功时为 `{"result": {...}}`；失败时为 `null` |

### 4.1 业务状态码

| code | 含义 | 说明 |
|------|------|------|
| `200` | 成功 | 业务数据在 `data.result` 中 |
| `400` | 参数或业务校验失败 | 例如金额格式不对、订单不存在；`message` 为中文具体原因 |
| `401` | 鉴权失败 | API Key 无效、IP 不在白名单、签名错误、重复请求等；`message` 为英文原因（HTTP 403 场景为中文，见 4.3） |
| `500` | 系统繁忙 | 平台内部异常，或请求体不是合法 JSON；请稍后用新的时间戳重试 |

> **重要：** 除限流（HTTP 429）、无法识别客户端 IP（HTTP 403）与请求体解析失败（HTTP 500）外，**所有失败响应的 HTTP 状态码都是 200**，业务结果以响应体中的 `code` 为准。请不要只判断 HTTP 状态码。

### 4.2 失败响应示例

```json
{
  "code": 401,
  "message": "Sign is invalid",
  "tm": 1762339201,
  "data": null
}
```

```json
{
  "code": 400,
  "message": "订单金额 amount 最多支持2位小数, 当前为 100.001",
  "tm": 1762339201,
  "data": null
}
```

```json
{
  "code": 500,
  "message": "系统繁忙",
  "tm": 1762339201,
  "data": null
}
```

### 4.3 非 200 的 HTTP 状态

绝大多数失败响应的 HTTP 状态仍是 `200`，只有下面三种例外：

| HTTP 状态 | Content-Type | 响应体 | 触发条件 |
|-----------|--------------|--------|----------|
| `429` | `text/plain` | `Too many requests, please slow down.`（**纯文本，不是 JSON**） | 单 IP 调用频率超过限流阈值 |
| `403` | `application/json` | 标准 JSON 结构，`code` 为 `401`（见下方示例） | 无法识别客户端真实 IP |
| `500` | `application/json` | `code` 为 `500`，`message` 为 `系统繁忙` | 请求体不是合法的 JSON 对象（解析失败时报错，见下方说明） |

HTTP `403` 的响应体仍是标准 JSON，只是 `code` 为 `401`、`message` 为中文：

```json
{
  "code": 401,
  "message": "无法确定客户端真实 IP: 请求经 Cloudflare 等代理转发, 但转发头里没有实际客户端地址, 已拒绝",
  "tm": 1762339201,
  "data": null
}
```

> **排查提示：** 该状态表示请求确实经过了代理或 Cloudflare，但转发头里没有真实客户端地址。平台此时无法做 IP 白名单与风控，只能直接拒绝。请让代理把真实客户端地址写入 `X-Forwarded-For`（Cloudflare 会带 `CF-Connecting-IP`，需要由受信代理转写），并确认平台侧配置了对应的受信代理网段。

请求体解析失败（不是合法 JSON，例如少了引号、多了尾逗号）时返回 HTTP `500`，响应体形如：

```json
{
  "code": 500,
  "message": "系统繁忙",
  "traceId": "b3f1c2d4e5a6478f9c0d1e2f3a4b5c6d"
}
```

> 这不是平台内部故障，而是请求体在进入业务逻辑前就解析失败。请务必用 JSON 校验工具确认请求体合法，并且 `Content-Type: application/json`；报错时把 `traceId` 提供给平台可快速定位。

### 4.4 鉴权失败原因对照（code=401）

| message | 原因 |
|---------|------|
| `API Key is required` | 缺少请求头 `X-API-KEY` |
| `API Key is invalid` | `X-API-KEY` 不存在或已失效 |
| `[x.x.x.x] not in white ips, please add it` | 调用来源 IP 不在白名单内 |
| `Client IP [x.x.x.x] is a private or LAN address, access denied` | 调用来源是内网/局域网地址 |
| `Request body is required` | 请求体为空（或请求体是 JSON `null`）；请求体不是合法 JSON 会返回 HTTP `500`，见 [4.3](#43-非-200-的-http-状态) |
| `Sign is required` | 请求体 `sign` 与请求头 `X-SIGN` 都没有传 |
| `Sign is invalid` | 签名计算错误（见 [2.5](#25-签名常见错误)） |
| `Duplicate request, please regenerate timestamp and sign` | 同一签名重复提交（见第 3 节） |

---

## 5. 接口总览

| 接口 | 路径 | 说明 |
|------|------|------|
| 创建支付订单 | `POST /api/open/pay/order` | 生成收银台地址，引导用户完成 PayPal 支付 |
| 查询支付订单 | `POST /api/open/pay/order/query` | 按平台订单号或商户订单号查询 |
| 创建订阅订单 | `POST /api/open/pay/subscription` | 生成订阅确认地址 |
| 查询订阅订单 | `POST /api/open/pay/subscription/query` | 查询订阅详情与状态 |
| 取消订阅 | `POST /api/open/pay/subscription/cancel` | 取消已成功订阅的订阅（仅 `ACTIVE`、`SUSPENDED` 可取消） |
| 查询余额 | `POST /api/open/balance/query` | 查询指定币种的余额与冻结金额 |

所有接口均需要 `X-API-KEY` 请求头与签名。请求示例中的 `{平台域名}` 请替换为平台分配的实际地址。

---

## 6. 创建支付订单

`POST /api/open/pay/order`

### 6.1 请求参数

| 字段 | 类型 | 必填 | 约束 | 说明 |
|------|------|------|------|------|
| `merchant_no` | string | 是 | 必须与 API Key 所属商户一致 | 商户编号 |
| `trade_no` | string | 是 | ≤64 字符 | 商户订单号，同一商户下唯一，幂等键 |
| `amount` | string | 是 | >0，最多 2 位小数 | 订单金额，如 `"100.00"` |
| `currency` | string | 是 | ≤16 字符，必须是已启用币种 | 交易币种，如 `USD`、`VND` |
| `product_no` | string | 否 | ≤32 字符，仅字母/数字/下划线/中划线 | 支付产品编号；不传时平台按币种与金额自动匹配 |
| `notify_url` | string | 否 | ≤255 字符，`http://` 或 `https://` 开头 | 异步通知地址；不传时使用 API Key 上配置的代收回调地址，两者都没有会报错 |
| `return_url` | string | 否 | ≤512 字符，`http://` 或 `https://` 开头 | 支付完成后的页面跳转地址 |
| `extra` | string | 否 | ≤255 字符 | 商户透传参数，通知时原样带回 |
| `version` | string | 否 | 不传按 `v1` 处理 | 接口版本，目前只支持 `v1` |
| `member_name` | string | 否 | ≤64 字符 | 用户唯一标识，如用户 ID |
| `identity_no` | string | 否 | ≤64 字符 | 用户证件号 |
| `timestamp` | string | 是 | 10 位或 13 位数字 | 时间戳 |
| `sign` | string | 是 | — | 签名，见第 2 节 |

### 6.2 请求示例

```json
{
  "merchant_no": "M10001",
  "trade_no": "PO20261106001",
  "amount": "100.00",
  "currency": "USD",
  "product_no": "PP_USD_001",
  "notify_url": "https://merchant.example.com/paypal/notify",
  "return_url": "https://merchant.example.com/paypal/return",
  "extra": "order-extra-001",
  "member_name": "user_8899",
  "identity_no": "110101199001011234",
  "version": "v1",
  "timestamp": "1762339200",
  "sign": "BE373535C8711C1929105D907FE0E4E2C6F69F67516B82ACDC4BF670296371B9"
}
```

> 上例签名对应**上面这个完整的请求体**（比 [2.3](#23-完整算例) 的示例多了 `product_no`、`return_url`、`extra`、`identity_no`、`version`，所以数值不同）。签名必须覆盖请求体里的每一个字段，少算或多算一个字段都会返回 `Sign is invalid`。

### 6.3 响应参数（`data.result`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `platform_trace_no` | string | 平台订单号 |
| `trade_no` | string | 商户订单号，原样返回 |
| `merchant_no` | string | 商户编号 |
| `product_no` | string | 实际使用的产品编号 |
| `currency` | string | 交易币种 |
| `amount` | string | 交易金额，两位小数 |
| `order_time` | long | 下单时间，秒级时间戳 |
| `url` | string | **平台收银台地址**，请引导用户在浏览器中打开，由收银台跳转 PayPal 完成支付 |
| `status` | string | 订单状态：`pending`（待支付）/ `success` / `failed` |

### 6.4 响应示例

```json
{
  "code": 200,
  "message": "success",
  "tm": 1762339201,
  "data": {
    "result": {
      "platform_trace_no": "20261106120000001234",
      "trade_no": "PO20261106001",
      "merchant_no": "M10001",
      "product_no": "PP_USD_001",
      "currency": "USD",
      "amount": "100.00",
      "order_time": 1762339201,
      "url": "https://pay.example.com/pay/20261106120000001234",
      "status": "pending"
    }
  }
}
```

### 6.5 注意事项

- 返回的 `url` 是平台收银台地址，**不是** PayPal 的支付地址，请直接让用户访问。
- 下单成功只代表订单已受理，最终支付结果以**异步通知**或查询接口为准。
- 重复提交相同 `trade_no` 时返回原订单数据（包含原 `url`），不会生成新订单。

---

## 7. 查询支付订单

`POST /api/open/pay/order/query`

### 7.1 请求参数

| 字段 | 类型 | 必填 | 约束 | 说明 |
|------|------|------|------|------|
| `merchant_no` | string | 是 | 必须与 API Key 所属商户一致 | 商户编号 |
| `platform_trace_no` | string | 否 | ≤64 字符 | 平台订单号，下单接口返回 |
| `trade_no` | string | 否 | ≤64 字符 | 商户订单号 |
| `timestamp` | string | 是 | 10 位或 13 位数字 | 时间戳 |
| `sign` | string | 是 | — | 签名 |

`platform_trace_no` 与 `trade_no` **至少传一个**；两个都传时按 `platform_trace_no` 查询，并核对 `trade_no` 是否一致。

### 7.2 请求示例

```json
{
  "merchant_no": "M10001",
  "platform_trace_no": "20261106120000001234",
  "timestamp": "1762339300",
  "sign": "48E23336210747B036B8AE064650596530BDBA9D1348A18A63EDF63CCD786C4E"
}
```

> 上例签名为对 `merchant_no=M10001&platform_trace_no=20261106120000001234&timestamp=1762339300` 做 HMAC-SHA256（apiSecret 为密钥）后的大写十六进制。

### 7.3 响应参数（`data.result`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `platform_trace_no` | string | 平台订单号 |
| `trade_no` | string | 商户订单号 |
| `merchant_no` | string | 商户编号 |
| `product_no` | string | 产品编号 |
| `product_name` | string | 产品名称 |
| `currency` | string | 交易币种 |
| `amount` | string | 订单金额，两位小数 |
| `fee` | string | 手续费，最多四位小数并去掉末尾 0 |
| `status` | string | 订单状态：`pending` / `success` / `failed` |
| `order_time` | long | 下单时间，秒级时间戳 |
| `pay_time` | long | 支付时间，秒级时间戳；未支付时为 `null` |

### 7.4 响应示例

```json
{
  "code": 200,
  "message": "success",
  "tm": 1762339301,
  "data": {
    "result": {
      "platform_trace_no": "20261106120000001234",
      "trade_no": "PO20261106001",
      "merchant_no": "M10001",
      "product_no": "PP_USD_001",
      "product_name": "PayPal 代收 USD",
      "currency": "USD",
      "amount": "100.00",
      "fee": "3.5",
      "status": "success",
      "order_time": 1762339201,
      "pay_time": 1762339260
    }
  }
}
```

### 7.5 注意事项

- 只能查询当前 API Key 所属商户名下的订单。
- 订单不存在与订单不属于当前商户返回**相同**的提示 `订单不存在: xxx`，这是安全设计，不是接口异常。

---

## 8. 创建订阅订单

`POST /api/open/pay/subscription`

### 8.1 请求参数

| 字段 | 类型 | 必填 | 约束 | 说明 |
|------|------|------|------|------|
| `merchant_no` | string | 是 | 必须与 API Key 所属商户一致 | 商户编号 |
| `trade_no` | string | 是 | ≤64 字符 | 商户订阅订单号，同一商户下唯一，幂等键 |
| `product_no` | string | 是 | ≤32 字符，仅字母/数字/下划线/中划线 | 订阅产品编号 |
| `return_url` | string | 否 | ≤512 字符，`http://` 或 `https://` 开头 | 用户在 PayPal 确认订阅后的跳转地址；不传时使用平台默认订阅结果页 |
| `cancel_url` | string | 否 | ≤512 字符，`http://` 或 `https://` 开头 | 用户取消订阅后的跳转地址；不传时使用平台默认订阅结果页 |
| `extra` | string | 否 | ≤255 字符 | 商户透传参数 |
| `version` | string | 否 | 不传按 `v1` 处理 | 接口版本，目前只支持 `v1` |
| `member_name` | string | 否 | ≤64 字符 | 用户唯一标识 |
| `identity_no` | string | 否 | ≤64 字符 | 用户证件号 |
| `timestamp` | string | 是 | 10 位或 13 位数字 | 时间戳 |
| `sign` | string | 是 | — | 签名 |

> **每期金额、币种、订阅周期与总期数都由订阅产品配置决定，下单时不能修改。** 请先与平台确认订阅产品编号。

### 8.2 请求示例

```json
{
  "merchant_no": "M10001",
  "trade_no": "SUB20261106001",
  "product_no": "SUB_USD_MONTHLY",
  "return_url": "https://merchant.example.com/subscription/return",
  "cancel_url": "https://merchant.example.com/subscription/cancel",
  "member_name": "user_8899",
  "timestamp": "1762339200",
  "sign": "97B498985BB8AB4DFE52D0BA44F7B2C04C41770C8AFAECB960746562D2F6D8BE"
}
```

### 8.3 响应参数（`data.result`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `platform_trace_no` | string | 平台订阅订单号 |
| `trade_no` | string | 商户订阅订单号，原样返回 |
| `merchant_no` | string | 商户编号 |
| `product_no` | string | 订阅产品编号 |
| `currency` | string | 扣款币种 |
| `amount` | string | 每期金额，两位小数 |
| `interval_unit` | string | 订阅周期单位：`DAY` / `WEEK` / `MONTH` / `YEAR` |
| `interval_count` | int | 每期跨越的周期数（如 `MONTH` + `1` 表示每月一期） |
| `total_cycles` | int | 总期数，`0` 表示不限 |
| `order_time` | long | 下单时间，秒级时间戳 |
| `approve_url` | string | **用户确认订阅地址**，请引导用户打开并完成确认 |
| `status` | string | 订阅状态，初始为 `APPROVAL_PENDING` |

### 8.4 响应示例

```json
{
  "code": 200,
  "message": "success",
  "tm": 1762339201,
  "data": {
    "result": {
      "platform_trace_no": "20261106120000005678",
      "trade_no": "SUB20261106001",
      "merchant_no": "M10001",
      "product_no": "SUB_USD_MONTHLY",
      "currency": "USD",
      "amount": "9.99",
      "interval_unit": "MONTH",
      "interval_count": 1,
      "total_cycles": 0,
      "order_time": 1762339201,
      "approve_url": "https://www.sandbox.paypal.com/webapps/billing/subscriptions?ba_token=BA-xxx",
      "status": "APPROVAL_PENDING"
    }
  }
}
```

### 8.5 注意事项

- 用户必须在 `approve_url` 完成确认后订阅才会生效，状态会从 `APPROVAL_PENDING` 变为 `APPROVED` → `ACTIVE`。
- **订阅没有推送给商户的异步通知**，订阅状态变化请通过[查询订阅订单](#9-查询订阅订单)接口获取（建议按业务需要轮询）。
- 重复提交相同 `trade_no` 时返回原订阅，不会重复创建。

---

## 9. 查询订阅订单

`POST /api/open/pay/subscription/query`

### 9.1 请求参数

| 字段 | 类型 | 必填 | 约束 | 说明 |
|------|------|------|------|------|
| `merchant_no` | string | 是 | 必须与 API Key 所属商户一致 | 商户编号 |
| `platform_trace_no` | string | 否 | ≤64 字符 | 平台订阅订单号 |
| `trade_no` | string | 否 | ≤64 字符 | 商户订阅订单号 |
| `timestamp` | string | 是 | 10 位或 13 位数字 | 时间戳 |
| `sign` | string | 是 | — | 签名 |

`platform_trace_no` 与 `trade_no` **至少传一个**；两个都传时按 `platform_trace_no` 查询并核对 `trade_no`。

请求示例：

```json
{
  "merchant_no": "M10001",
  "platform_trace_no": "20261106120000005678",
  "timestamp": "1762339300",
  "sign": "ED4129958D5F906E6D462794391F316FAD815178B5C04AF5BBECFDC26F1EFF6A"
}
```

### 9.2 响应参数（`data.result`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `platform_trace_no` | string | 平台订阅订单号 |
| `trade_no` | string | 商户订阅订单号 |
| `merchant_no` | string | 商户编号 |
| `product_no` | string | 订阅产品编号 |
| `product_name` | string | 订阅产品名称 |
| `currency` | string | 扣款币种 |
| `amount` | string | 每期金额，两位小数 |
| `interval_unit` | string | 周期单位：`DAY` / `WEEK` / `MONTH` / `YEAR` |
| `interval_count` | int | 每期跨越的周期数 |
| `total_cycles` | int | 总期数，`0` 表示不限 |
| `subscribed_cycles` | int | 已订阅（已扣款）期数，未扣款时为 `0` |
| `status` | string | 订阅状态，见 [13.1](#131-订阅状态) |
| `subscribing` | boolean | 是否订阅中，等价于 `status == "ACTIVE"` |
| `paypal_subscription_id` | string | PayPal 订阅号，订阅生效后才有 |
| `payer_name` | string | 订阅人姓名，订阅生效后由 PayPal 带回 |
| `payer_email` | string | 订阅人邮箱，订阅生效后由 PayPal 带回 |
| `order_time` | long | 下单时间，秒级时间戳 |
| `next_billing_time` | long | 下次扣款时间，秒级时间戳；未生效或已结束时为 `null` |
| `last_payment_time` | long | 最近一次扣款时间，秒级时间戳；尚未扣款时为 `null` |

### 9.3 响应示例

```json
{
  "code": 200,
  "message": "success",
  "tm": 1762339401,
  "data": {
    "result": {
      "platform_trace_no": "20261106120000005678",
      "trade_no": "SUB20261106001",
      "merchant_no": "M10001",
      "product_no": "SUB_USD_MONTHLY",
      "product_name": "USD 月度订阅",
      "currency": "USD",
      "amount": "9.99",
      "interval_unit": "MONTH",
      "interval_count": 1,
      "total_cycles": 0,
      "subscribed_cycles": 2,
      "status": "ACTIVE",
      "subscribing": true,
      "paypal_subscription_id": "I-BW452GLLEP1G",
      "payer_name": "John Doe",
      "payer_email": "john@example.com",
      "order_time": 1762339201,
      "next_billing_time": 1767600000,
      "last_payment_time": 1765017600
    }
  }
}
```

### 9.4 注意事项

- 只能查询当前 API Key 所属商户名下的订阅。
- 订阅不存在与不属于当前商户返回相同的提示 `订阅不存在: xxx`。

---

## 10. 取消订阅

`POST /api/open/pay/subscription/cancel`

### 10.1 请求参数

与[查询订阅订单](#91-请求参数)完全一致（`merchant_no` 必填，`platform_trace_no` / `trade_no` 至少一个，外加 `timestamp`、`sign`）。

请求示例：

```json
{
  "merchant_no": "M10001",
  "platform_trace_no": "20261106120000005678",
  "timestamp": "1762339600",
  "sign": "90E17227234DC490D07FEAE1211A79B68C9622FB1071A5E6AFB960B85CDDE18B"
}
```

### 10.2 响应参数

与[查询订阅订单](#92-响应参数dataresult)一致，返回取消后的订阅信息，`status` 为 `CANCELLED`。

### 10.3 取消规则

- **只有订阅成功的才能取消。** 用户必须在 PayPal 完成确认、订阅进入生效状态（`ACTIVE`）后才可取消；已暂停的 `SUSPENDED` 同样可以取消。
- 仍处于 `APPROVAL_PENDING`（用户尚未在 PayPal 确认）或 `APPROVED`（已确认但尚未扣款成功）的订阅不能取消，此时没有已建立的扣款关系，请引导用户不要继续完成确认即可。
- 平台先调用 PayPal 取消，**PayPal 取消成功后**才更新本地状态；PayPal 调用失败时订阅保持原状态并返回 `400`。
- 以下情况会返回 `400`：
  - `订阅已取消, 无需重复取消`（`CANCELLED`）
  - `订阅已到期, 无需取消`（`EXPIRED`）
  - `订阅当前状态为 XXX, 只有订阅成功(ACTIVE)或已暂停(SUSPENDED)的订阅可以取消`（`APPROVAL_PENDING`、`APPROVED` 等其他状态）
  - `订阅缺少 PayPal 账号或订阅号, 无法取消`（订阅尚未在 PayPal 生效）
- 取消后不再产生后续扣款，已扣款项不自动退款。

---

## 11. 查询余额

`POST /api/open/balance/query`

### 11.1 请求参数

| 字段 | 类型 | 必填 | 约束 | 说明 |
|------|------|------|------|------|
| `merchant_no` | string | 是 | 必须与 API Key 所属商户一致 | 商户编号 |
| `currency` | string | 是 | ≤16 字符，币种必须存在 | 查询币种，如 `USD`；大小写不敏感 |
| `timestamp` | string | 是 | 10 位或 13 位数字 | 时间戳 |
| `sign` | string | 是 | — | 签名 |

> 查询余额只要求币种存在，不要求该币种处于启用状态，方便停用币种后仍能对账。未初始化的币种按 `0` 返回。

### 11.2 请求示例

```json
{
  "merchant_no": "M10001",
  "currency": "USD",
  "timestamp": "1762339500",
  "sign": "FDE77966DE4EAD914B67863220C3F546F8B295716CB3668A43A694CBC2C7619E"
}
```

### 11.3 响应参数（`data.result`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `merchant_no` | string | 商户编号 |
| `mch_name` | string | 商户名称 |
| `currency` | string | 币种（按平台配置的标准写法返回） |
| `balance` | string | 账户总余额，**四位小数** |
| `freeze` | string | 冻结金额（代付/提现冻结），**四位小数** |

> 可用余额 = `balance` - `freeze`，由商户自行计算。

### 11.4 响应示例

```json
{
  "code": 200,
  "message": "success",
  "tm": 1762339501,
  "data": {
    "result": {
      "merchant_no": "M10001",
      "mch_name": "示例商户",
      "currency": "USD",
      "balance": "99653.1183",
      "freeze": "120.0000"
    }
  }
}
```

---

## 12. 订单异步通知

支付订单进入终态（支付成功 `success` / 支付失败 `failed`）后，平台会向订单的 `notify_url`（未传时取 API Key 上配置的代收回调地址）发送 **HTTP POST**，`Content-Type: application/json; charset=utf-8`。

> 只有支付订单有异步通知；订阅订单没有推送，请使用查询接口获取状态。

### 12.1 通知报文

| 字段 | 类型 | 说明 |
|------|------|------|
| `platform_trace_no` | string | 平台订单号 |
| `trade_no` | string | 商户订单号 |
| `merchant_no` | string | 商户编号 |
| `product_no` | string | 产品编号 |
| `currency` | string | 交易币种 |
| `amount` | string | 订单金额，两位小数 |
| `order_time` | string | 下单时间，秒级时间戳（注意：通知报文中为**字符串**） |
| `extra` | string | 下单时透传的 `extra`，无则为空串 |
| `status` | string | `success` / `failed` |
| `sign` | string | 签名，算法与请求签名完全一致（对同样的待签串做 HMAC-SHA256，apiSecret 为密钥，大写十六进制） |

### 12.2 通知示例

```json
{
  "platform_trace_no": "20261106120000001234",
  "trade_no": "PO20261106001",
  "merchant_no": "M10001",
  "product_no": "PP_USD_001",
  "currency": "USD",
  "amount": "100.00",
  "order_time": "1762339205",
  "extra": "order-extra-001",
  "status": "success",
  "sign": "F1A0DB5AEC657CDA8152D79184D5D187AE501F4EE589903A76E7DEF0588D4E9D"
}
```

验签时按第 2 节算法，用**同一套 apiSecret** 重算（剔除 `sign`）并比对，例如上例的待签串为：

```text
amount=100.00&currency=USD&extra=order-extra-001&merchant_no=M10001&order_time=1762339205&platform_trace_no=20261106120000001234&product_no=PP_USD_001&status=success&trade_no=PO20261106001
```

对该串用 apiSecret 做 HMAC-SHA256、转大写十六进制，应得到通知报文里的 `sign`（上例为
`F1A0DB5AEC657CDA8152D79184D5D187AE501F4EE589903A76E7DEF0588D4E9D`）。

### 12.3 商户应答与重试

- 商户收到通知后，请校验签名并**返回 HTTP 2xx**（建议返回纯文本 `success`），平台据此判定通知成功。
- 非 2xx 或请求异常视为失败，平台自动重试：首次立即发送，失败后分别在 **1、5、10、30 分钟**后重试，共最多 **5 次**；仍失败则停止重试，可在平台后台对该订单手动补发。
- 通知存在**重复投递**的可能（例如平台未收到应答但商户已处理），请以 `platform_trace_no` / `trade_no` 做幂等处理。
- 商户处理逻辑请尽快完成，建议先落库再返回应答，避免超时导致重试。

### 12.4 通知与查询的关系

- 通知是**结果推送**，查询接口是**最终依据**：商户漏收通知时，请主动调用[查询支付订单](#7-查询支付订单)对账。
- 通知中的 `status` 与查询接口的 `status` 取值一致，使用同一套判断逻辑。

---

## 13. 状态枚举与格式约定

### 13.1 订阅状态

| status | 说明 | 是否可取消 |
|--------|------|-----------|
| `APPROVAL_PENDING` | 已创建，等待用户在 PayPal 确认 | 否（尚未订阅成功） |
| `APPROVED` | 用户已确认，等待生效 | 否（尚未订阅成功） |
| `ACTIVE` | 订阅生效中（`subscribing = true`） | 是 |
| `SUSPENDED` | 已暂停 | 是 |
| `CANCELLED` | 已取消 | 否（已取消） |
| `EXPIRED` | 已到期 | 否（已到期） |

> 判断"订阅成功"请用 `status == "ACTIVE"`（即 `subscribing == true`）；取消请见[第 10 节](#10-取消订阅)。

### 13.2 订单状态

| status | 说明 |
|--------|------|
| `pending` | 待支付 |
| `success` | 支付成功 |
| `failed` | 支付失败 |

### 13.3 金额与时间格式

| 项目 | 格式 | 示例 |
|------|------|------|
| 订单金额 `amount` | 两位小数字符串 | `"100.00"` |
| 手续费 `fee` | 最多四位小数，去掉末尾 0 | `"3.5"`、`"6"`、`"0.635"` |
| 余额 `balance` / `freeze` | 四位小数字符串 | `"99653.1183"` |
| 响应时间戳 `tm`、`order_time`、`pay_time` 等 | 秒级整数时间戳 | `1762339201` |
| 请求时间戳 `timestamp` | 10 位（秒）或 13 位（毫秒）字符串 | `"1762339200"` |

> 金额与余额均为**字符串**，请用定点小数（如 Java 的 `BigDecimal`、PHP 的 `bcmath`/字符串、Python 的 `Decimal`）处理，避免浮点精度问题。

---

## 14. 常见问题排查

| 现象 | 排查方向 |
|------|----------|
| 返回 `401 Sign is invalid` | 对照 [2.5](#25-签名常见错误)：是否 URL 编码、是否跳过空值、参数名/取值是否与请求体完全一致、apiSecret 是否正确 |
| 返回 `401 Sign is required` | 请求体没有 `sign` 且没有请求头 `X-SIGN` |
| 返回 `401 API Key is invalid` | `X-API-KEY` 拼写错误或已失效，联系平台确认 |
| 返回 `401 [x.x.x.x] not in white ips` | 把该出口 IP 加入 API Key 白名单，或清空白名单 |
| 返回 `401 ... private or LAN address` | 调用来源是内网地址，需通过公网出口调用 |
| 返回 `401 Duplicate request` | 同一签名重复提交，重试时重新生成 `timestamp` 并重新签名 |
| 返回 `400 商户编号 merchant_no xxx 与 API Key 对应的商户 yyy 不一致` | `merchant_no` 与所用 API Key 不属于同一商户 |
| 返回 `400 订单金额 amount ...` | 金额不是数字、≤0 或超过 2 位小数 |
| 返回 `400 币种不存在 / 已停用` | 币种代码错误或未启用，联系平台确认 |
| 返回 `400 异步回调地址 notify_url 不能为空...` | 未传 `notify_url` 且 API Key 未配置代收回调地址 |
| HTTP `500` + `code 500 系统繁忙` | 先检查请求体是否为合法 JSON（解析失败走这条路径），其次才是平台内部异常，重试时换新的 `timestamp` 并附上 `traceId` 报障 |
| 返回 `400 订单不存在 / 订阅不存在` | 订单号错误，或该订单不属于当前 API Key 所属商户 |
| HTTP `429` 纯文本 | 触发限流，降低频率并退避重试 |
| HTTP `403` JSON（`code=401`，中文 message） | 无法识别客户端真实 IP，见 [4.3](#43-非-200-的-http-状态) |

---

## 附录：其它接口文档

平台内部通道接口文档见 [docs/api-common-channel.md](docs/api-common-channel.md)（`CommonChannelApiController`），与本文档的开放接口不是同一套接口，请勿混用。

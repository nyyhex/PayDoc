# 四方支付平台 商户开放接口文档

| 项目 | 说明 |
| --- | --- |
| 接口版本 | v1 |
| 文档版本 | 2026-10-28 |
| 适用范围 | 商户通过 API Key 调用代收（支付）、代付、订单查询、余额查询接口，并接收平台异步通知 |

---

## 目录

1. [接入说明](#1-接入说明)
2. [鉴权与签名](#2-鉴权与签名)
3. [返回结构与错误码](#3-返回结构与错误码)
4. [接口详情](#4-接口详情)
   - [4.1 创建支付订单](#41-创建支付订单)
   - [4.2 查询支付订单](#42-查询支付订单)
   - [4.3 创建代付订单](#43-创建代付订单)
   - [4.4 查询代付订单](#44-查询代付订单)
   - [4.5 查询余额](#45-查询余额)
5. [订单状态](#5-订单状态)
6. [商户异步通知（回调）](#6-商户异步通知回调)
7. [附录 A：错误提示与错误码汇总](#附录-a错误提示与错误码汇总)
8. [附录 B：接入自检清单](#附录-b接入自检清单)

---

## 1. 接入说明

### 1.1 接入信息

| 项目 | 说明 |
| --- | --- |
| 接口根地址 | `https://fps-gateway.xyzbit7.com`（生产环境；测试环境地址由运营提供） |
| 请求方式 | 全部接口均为 `POST` |
| 请求类型 | `Content-Type: application/json; charset=utf-8` |
| 响应类型 | `application/json; charset=utf-8` |
| 字符编码 | UTF-8 |
| 请求压缩 | 支持 `Content-Encoding: gzip`，未压缩的请求体同样支持 |
| 时间基准 | 以平台服务器时间为准，`timestamp` 允许误差 **±5 分钟** |
| 传输安全 | 建议全站使用 HTTPS。平台只对报文做 **MD5 签名**（用于防篡改与身份校验），**不对报文加密**，报文内容为明文 JSON |

> 平台返回的业务结果一律以响应体中的 `code` 判断（见第 3 节）。在通过鉴权的正常请求中，HTTP 状态码固定为 `200`，请勿以 HTTP 状态码判断业务成功与否。

### 1.2 接口清单

| 接口 | 方法 | 路径 | 说明 |
| --- | --- | --- | --- |
| 创建支付订单 | POST | `/api/open/payment/create` | 代收下单，返回平台订单号与支付链接 |
| 查询支付订单 | POST | `/api/open/payment/query` | 按商户订单号查询代收结果（只读） |
| 创建代付订单 | POST | `/api/open/payout/create` | 代付下单，返回平台订单号与手续费 |
| 查询代付订单 | POST | `/api/open/payout/query` | 按商户订单号查询代付结果（只读） |
| 查询余额 | POST | `/api/open/balance/query` | 按币种查询商户余额与冻结金额（只读） |
| 商户异步通知 | POST | 商户自定义 `notify_url` | 平台主动推送订单终态，见第 6 节 |

---

## 2. 鉴权与签名

### 2.1 API Key

平台在商户后台为每个商户生成 API Key（`apiKey` / `apiSecret` 成对），调用所有开放接口时必须在请求头携带：

```
X-API-KEY: 你的 apiKey
```

`apiSecret` **只用于计算签名，任何情况下都不要通过接口传输**。

- 商户身份完全由 `X-API-KEY` 决定，请求体中的 `merchant_no` 只用于核对，必须与 API Key 所属商户一致。
- 订单通知商户时，使用下单时所用 API Key 的 `apiSecret` 签名；后续新增或更换 API Key 不影响已下单订单的验签。

### 2.2 IP 白名单

平台有两层来源 IP 校验，两层都通过才可调用：

1. **API Key 白名单**（推荐）：在 API Key 上配置 `whiteIp`，多个 IP 用英文逗号分隔，**精确匹配、不会自动去空格**，所以请写成 `1.1.1.1,2.2.2.2`（逗号后不要带空格）。未配置表示不限制。
   - 校验失败返回：`[实际IP] not in white ips, please add it`
2. **商户账号 API IP 白名单**：在商户账号上配置的 IP 列表，为空表示不限制，配置 `*` 表示任意 IP。
   - 校验失败返回：`该IP: 实际IP 无权使用api`

平台获取客户端 IP 的规则：优先取 `X-Forwarded-For` 的**第一个**地址，取不到时取连接的远端地址。

### 2.3 签名规则

签名使用 **MD5 摘要**，算法固定，与参数在 JSON 中的书写顺序、空白字符无关（签名基于解析后的键值，而不是请求体原文）。

#### 计算步骤

1. 取请求体 JSON 中的**全部字段**（包括平台未定义的扩展字段），排除 `sign` 与 `sign_type`；
2. 去掉值为 `null`、空字符串或全空白的字段；
3. 按**参数名 ASCII 码升序**排序（即字典序）；
4. 拼接为 `k1=v1&k2=v2&...`（键与值之间是 `=`，字段之间是 `&`，**末尾不加 `&`**）；
5. 在拼接串**末尾直接拼接 `apiSecret`**（不加任何分隔符）；
6. 对结果做 `MD5`（UTF-8 编码），输出 **32 位大写十六进制**。

#### 值的处理规则

| 值类型 | 参与签名的字符串 |
| --- | --- |
| 字符串 | 原值（平台不会去掉首尾空格，但业务校验时会去掉，请勿在值两侧加空格） |
| 数字 | JSON 中该数字的字符串形式（**`100` 与 `100.00` 是不同的字符串**） |
| 布尔 | `true` / `false` |
| 对象 / 数组 | 该值的 JSON 序列化结果（不推荐在业务字段中使用） |

> **强烈建议所有业务字段都使用字符串类型传递**（尤其是 `amount`、`timestamp`）。以数字传递时，`100`、`100.0`、`100.00` 会被还原成不同的字符串，导致签名对不上。

#### 签名放在哪里

以下两种方式**任选其一**，同时传时以请求头 `X-SIGN` 为准：

| 位置 | 示例 |
| --- | --- |
| 请求头（推荐） | `X-SIGN: D1C6508DE002D01D2B030B218FED5E28` |
| 请求体字段 | `"sign": "D1C6508DE002D01D2B030B218FED5E28"` |

平台比对签名时**忽略大小写**（`equalsIgnoreCase`），即大写、小写签名都可通过，但本文档统一使用大写。

> 注意：签名必须覆盖请求体中的全部字段。如果请求体里额外放了平台未定义的字段，这些字段同样参与签名。

### 2.4 签名示例

以下示例统一使用示例密钥 `testsecret123456`（**仅为演示，实际以平台下发的 apiSecret 为准**）。示例中的 `timestamp` 是固定值，实际调用必须传当前时间。

#### 示例一：查询余额

请求体（`sign` 字段本身不参与签名）：

```json
{
  "merchant_no": "100353972",
  "currency": "VND",
  "timestamp": "1788888888",
  "sign": "764AA60F19B6E9228B561A99B07ED28A"
}
```

待签名字符串：

```
currency=VND&merchant_no=100353972&timestamp=1788888888testsecret123456
```

签名结果：

```
764AA60F19B6E9228B561A99B07ED28A
```

#### 示例二：创建支付订单

待签名字符串（按键名升序，`sign` 已排除）：

```
amount=100.00&currency=VND&extra=uid=u1001&identity_no=012345678901&member_name=u1001&merchant_no=100353972&notify_url=https://merchant.example.com/notify&product_no=P1001&return_url=https://merchant.example.com/return&timestamp=1788888888&trade_no=test20260101001&version=v1testsecret123456
```

签名结果（放在 `sign` 字段或 `X-SIGN` 请求头）：

```
D1C6508DE002D01D2B030B218FED5E28
```

#### 示例三：创建代付订单

待签名字符串：

```
account=1234567890&account_type=Bank&amount=500.00&bank=Vietcombank&bank_branch=Hanoi&bank_code=VCB&currency=VND&extra=uid=u1001&merchant_no=100353972&notify_url=https://merchant.example.com/notify&payee=NGUYEN VAN A&product_no=PO1001&remark=salary&timestamp=1788888888&trade_no=test20260101002&version=v1testsecret123456
```

签名结果：

```
3D6C8FCF9C4F8742B3E14C9E332CCA93
```

### 2.5 签名计算代码参考（Java）

```java
public static String sign(Map<String, Object> params, String apiSecret) throws Exception {
    // 1. 排除 sign / sign_type, 按键名 ASCII 升序排序
    TreeMap<String, Object> sorted = new TreeMap<>();
    for (Map.Entry<String, Object> e : params.entrySet()) {
        if ("sign".equals(e.getKey()) || "sign_type".equals(e.getKey())) {
            continue;
        }
        sorted.put(e.getKey(), e.getValue());
    }
    // 2. 拼 k=v&k=v, 跳过 null / 空 / 全空白
    StringBuilder sb = new StringBuilder();
    for (Map.Entry<String, Object> e : sorted.entrySet()) {
        Object v = e.getValue();
        String value = v == null ? "" : String.valueOf(v);
        if (value.trim().isEmpty()) {
            continue;
        }
        if (sb.length() > 0) {
            sb.append('&');
        }
        sb.append(e.getKey()).append('=').append(value);
    }
    // 3. 末尾直接拼 apiSecret, MD5 后转大写
    byte[] digest = MessageDigest.getInstance("MD5")
            .digest((sb + apiSecret).getBytes(StandardCharsets.UTF_8));
    StringBuilder hex = new StringBuilder();
    for (byte b : digest) {
        hex.append(String.format("%02X", b));
    }
    return hex.toString();
}
```

### 2.6 调用示例（curl）

```bash
curl -X POST 'https://fps-gateway.xyzbit7.com/api/open/balance/query' \
  -H 'Content-Type: application/json' \
  -H 'X-API-KEY: your_api_key' \
  -H 'X-SIGN: 764AA60F19B6E9228B561A99B07ED28A' \
  -d '{"merchant_no":"100353972","currency":"VND","timestamp":"1788888888"}'
```

### 2.7 鉴权失败提示（code = 401）

鉴权在进入业务逻辑之前完成，失败时返回英文提示（便于各语言商户接入时定位），完整列表见 [附录 A.1](#a1-鉴权与签名-401)。

---

## 3. 返回结构与错误码

### 3.1 统一返回结构

成功：

```json
{
  "code": 200,
  "message": "success",
  "tm": 1788888888,
  "data": {
    "result": {
      "merchant_no": "100353972",
      "mch_name": "aaa",
      "currency": "VND",
      "balance": "99653.1183",
      "freeze": "1400.0000"
    }
  }
}
```

失败：

```json
{
  "code": 400,
  "message": "订单金额 amount 只支持整数, 当前为 100.5, 请按 100.00 的格式传递",
  "tm": 1788888888
}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `code` | int | 业务状态码，`200` 表示成功，其余为失败 |
| `message` | String | 成功时固定为 `success`；失败时为具体原因（中文业务提示或英文鉴权提示） |
| `tm` | long | 平台服务器时间，秒级时间戳 |
| `data` | Object | 仅成功时返回；业务数据在 `data.result` 中；失败时整个 `data` 不返回 |
| `data.result` | Object | 各接口的具体返回，见第 4 节 |

补充说明：

- `data.result` 中值为 `null` 的字段**不会出现在报文里**（例如通道未返回支付地址时，创建支付订单的返回中没有 `url` 字段）。解析时请按“字段可缺省”处理。
- 查询类接口中未成功的订单，`success_time` 固定返回 `0`（而不是省略字段）。
- 所有金额均为**字符串**，避免浮点精度问题；订单金额 2 位小数，余额与冻结金额 4 位小数，手续费去掉末尾 0。

### 3.2 错误码

| code | message | 含义 | 商户处理建议 |
| --- | --- | --- | --- |
| `200` | `success` | 成功 | 解析 `data.result` |
| `400` | 具体原因 | 参数校验失败或业务校验失败 | 按 `message` 修正后重试；下单类错误修正前重复提交不会产生新订单 |
| `401` | 具体原因 | API Key、IP 白名单或签名校验失败 | 检查 `X-API-KEY`、IP 白名单、签名算法与密钥 |
| `500` | `系统繁忙` | 平台内部异常 | 稍后重试；**下单类接口请先用查询接口确认订单是否存在**，避免重复下单 |

`400` 中包含少量平台内部异常映射，例如：

| message | 触发场景 |
| --- | --- |
| `请求体格式不正确, 请传入合法的 JSON` | 请求体不是合法 JSON（如多传了逗号、使用了单引号） |
| `数据已经存在` | 并发重复提交同一 `trade_no`，被数据库唯一索引拦截；请用查询接口确认订单结果 |
| `请求参数不能为空` | 请求体为空（未携带任何参数） |

---

## 4. 接口详情

### 4.0 公共约定

- 所有接口都需要 `merchant_no`、`timestamp`、`sign`（签名也可放 `X-SIGN` 请求头）。
- 所有字符串字段在业务校验前会**去掉首尾空格**；`merchant_no` 必须与 API Key 所属商户**完全一致**（区分大小写）。
- `currency` 必须与平台币种配置中的代码**完全一致**（区分大小写，例如配置为 `VND` 时传 `vnd` 会提示“币种不存在”）。
- `trade_no` 在同一商户下唯一；**支付与代付共用同一唯一约束**，同一个 `trade_no` 不能既用于支付又用于代付。
- 下单接口的校验顺序为：`merchant_no` → `version` → `trade_no` → `amount` → `currency` → `product_no` → `notify_url` → 其他业务字段 → `timestamp` → 订单号查重 → 产品/通道匹配 → 创建订单 → 请求通道。报错总是**第一个**未通过的校验，修正后可能暴露下一个问题。

### 4.1 创建支付订单

`POST /api/open/payment/create`

#### 请求参数

| 参数 | 必填 | 类型/长度 | 说明 |
| --- | --- | --- | --- |
| `merchant_no` | 是 | String ≤64 | 商户编号（即商户 userID），必须与 API Key 所属商户一致 |
| `trade_no` | 是 | String ≤64 | 商户订单号，同一商户下唯一 |
| `amount` | 是 | String | 订单金额，**只支持整数**（可写成 `100.00`，但小数部分必须为 0），必须大于 0 |
| `currency` | 是 | String ≤16 | 币种代码，必须是**已存在且已启用**的币种，如 `VND` |
| `product_no` | 否 | String ≤32 | 支付产品编号，仅允许字母、数字、下划线、中划线；不传时由平台在匹配的币种、金额下自动选择产品与通道 |
| `notify_url` | 否 | String ≤512 | 异步通知地址，必须以 `http://` 或 `https://` 开头；不传时取本次调用所用 API Key 配置的**代收回调地址**（该地址列宽 255），两者都没有则下单失败 |
| `return_url` | 否 | String ≤512 | 支付成功后的页面跳转地址，以 `http://` 或 `https://` 开头 |
| `extra` | 否 | String ≤255 | 商户透传参数，通知时原样返回 |
| `member_name` | 是 | String ≤64 | 付款用户唯一标识（如用户 ID） |
| `identity_no` | 否 | String ≤64 | 付款人身份证号，部分通道需要 |
| `version` | 否 | String | 接口版本，目前只支持 `v1`，不传按 `v1` 处理 |
| `timestamp` | 是 | String | 10 位秒级或 13 位毫秒级时间戳，与服务器时间相差不超过 5 分钟 |
| `sign` | 否 | String | 签名，也可放请求头 `X-SIGN` |

#### 请求示例

```json
{
  "merchant_no": "100353972",
  "trade_no": "test20260101001",
  "amount": "100.00",
  "currency": "VND",
  "product_no": "P1001",
  "notify_url": "https://merchant.example.com/notify",
  "return_url": "https://merchant.example.com/return",
  "extra": "uid=u1001",
  "version": "v1",
  "member_name": "u1001",
  "identity_no": "012345678901",
  "timestamp": "1788888888",
  "sign": "D1C6508DE002D01D2B030B218FED5E28"
}
```

#### 返回参数（`data.result`）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `platform_trace_no` | String | 平台订单号（全局唯一），后续对账、查单、通知均以此为准 |
| `trade_no` | String | 商户订单号，原样返回 |
| `merchant_no` | String | 商户编号 |
| `product_no` | String | 实际使用的支付产品编号 |
| `currency` | String | 币种代码（平台配置中的代码） |
| `amount` | String | 订单金额，2 位小数 |
| `order_time` | long | 下单时间，**秒级时间戳** |
| `url` | String | 通道返回的支付/收银台地址；通道未返回时该字段不出现 |
| `status` | String | 订单状态，下单成功为 `pending`，见第 5 节 |

#### 返回示例

```json
{
  "code": 200,
  "message": "success",
  "tm": 1788888888,
  "data": {
    "result": {
      "platform_trace_no": "ORD202601011200001234",
      "trade_no": "test20260101001",
      "merchant_no": "100353972",
      "product_no": "P1001",
      "currency": "VND",
      "amount": "100.00",
      "order_time": 1788888888,
      "url": "https://pay.example.com/cashier?token=abc123",
      "status": "pending"
    }
  }
}
```

> 说明：支付订单的手续费不在本接口返回。支付成功后商户入账金额为订单金额扣除平台手续费，费率由平台配置，可在商户后台查看。

### 4.2 查询支付订单

`POST /api/open/payment/query`

只读接口：不改变订单状态，也不触发任何通知。适用于未收到异步通知时主动拉取结果。

#### 请求参数

| 参数 | 必填 | 类型/长度 | 说明 |
| --- | --- | --- | --- |
| `merchant_no` | 是 | String ≤64 | 商户编号，必须与 API Key 所属商户一致 |
| `trade_no` | 是 | String ≤64 | 下单时传入的商户订单号 |
| `currency` | 是 | String ≤16 | 下单时的币种；与订单币种比较时忽略大小写，不一致会直接报错 |
| `timestamp` | 是 | String | 10 位或 13 位时间戳，误差 5 分钟内 |
| `sign` | 否 | String | 签名，也可放请求头 `X-SIGN` |

#### 请求示例

```json
{
  "merchant_no": "100353972",
  "trade_no": "test20260101001",
  "currency": "VND",
  "timestamp": "1788888888",
  "sign": "95A4B9D816599C8B5470CE9AE077A571"
}
```

#### 返回参数（`data.result`）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `platform_trace_no` | String | 平台订单号 |
| `merchant_no` | String | 商户编号 |
| `trade_no` | String | 商户订单号 |
| `amount` | String | 订单金额，2 位小数 |
| `currency` | String | 币种 |
| `status` | String | 订单状态：`pending` / `success` / `failed` |
| `success_time` | long | 订单成功时间，秒级时间戳；**未成功时为 `0`** |

#### 返回示例

```json
{
  "code": 200,
  "message": "success",
  "tm": 1788888888,
  "data": {
    "result": {
      "platform_trace_no": "ORD202601011200001234",
      "merchant_no": "100353972",
      "trade_no": "test20260101001",
      "amount": "100.00",
      "currency": "VND",
      "status": "success",
      "success_time": 1788889999
    }
  }
}
```

### 4.3 创建代付订单

`POST /api/open/payout/create`

#### 请求参数

| 参数 | 必填 | 类型/长度 | 说明 |
| --- | --- | --- | --- |
| `merchant_no` | 是 | String ≤64 | 商户编号，必须与 API Key 所属商户一致 |
| `trade_no` | 是 | String ≤64 | 商户代付订单号，同一商户下唯一（与支付订单共用唯一约束） |
| `amount` | 是 | String | 代付金额，**最多 2 位小数**，必须大于 0 |
| `currency` | 是 | String ≤16 | 币种代码，必须是已存在且已启用的币种 |
| `product_no` | 否 | String ≤32 | 代付产品编号，规则同支付 |
| `notify_url` | 否 | String ≤512 | 异步通知地址；不传时取本次调用所用 API Key 配置的**代付回调地址**（该地址列宽 255），两者都没有则下单失败 |
| `account` | 是 | String ≤64 | 收款银行账户 |
| `account_type` | 是 | String | 账户类型，目前只支持 `Bank`（银行账户） |
| `payee` | 是 | String ≤128 | 收款人姓名 |
| `bank` | 是 | String ≤128 | 银行名称，`account_type=Bank` 时必传 |
| `bank_branch` | 否 | String ≤128 | 银行分行名称 |
| `bank_code` | 条件必填 | String ≤32 | 银行编号；**币种为 VND 时必传**，银行编号请联系运营获取 |
| `extra` | 否 | String ≤255 | 商户透传参数，通知时原样返回 |
| `remark` | 否 | String ≤255 | 备注 |
| `version` | 否 | String | 接口版本，目前只支持 `v1` |
| `timestamp` | 是 | String | 10 位或 13 位时间戳，误差 5 分钟内 |
| `sign` | 否 | String | 签名，也可放请求头 `X-SIGN` |

#### 请求示例

```json
{
  "merchant_no": "100353972",
  "trade_no": "test20260101002",
  "amount": "500.00",
  "currency": "VND",
  "product_no": "PO1001",
  "notify_url": "https://merchant.example.com/notify",
  "account": "1234567890",
  "account_type": "Bank",
  "payee": "NGUYEN VAN A",
  "bank": "Vietcombank",
  "bank_branch": "Hanoi",
  "bank_code": "VCB",
  "extra": "uid=u1001",
  "remark": "salary",
  "version": "v1",
  "timestamp": "1788888888",
  "sign": "3D6C8FCF9C4F8742B3E14C9E332CCA93"
}
```

#### 返回参数（`data.result`）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `platform_trace_no` | String | 平台订单号 |
| `trade_no` | String | 商户订单号，原样返回 |
| `merchant_no` | String | 商户编号 |
| `product_no` | String | 实际使用的代付产品编号 |
| `currency` | String | 币种 |
| `amount` | String | 代付金额，2 位小数 |
| `order_time` | long | 下单时间，秒级时间戳 |
| `fee` | String | 手续费，最多 4 位小数并去掉末尾 0（如 `6`、`0.635`） |
| `status` | String | 订单状态，下单成功为 `pending` |

#### 返回示例

```json
{
  "code": 200,
  "message": "success",
  "tm": 1788888888,
  "data": {
    "result": {
      "platform_trace_no": "ORD202601011200005678",
      "trade_no": "test20260101002",
      "merchant_no": "100353972",
      "product_no": "PO1001",
      "currency": "VND",
      "amount": "500.00",
      "order_time": 1788888888,
      "fee": "0.6",
      "status": "pending"
    }
  }
}
```

> 资金说明：
> - 手续费 = 代付金额 × 费率 ÷ 100，保留 2 位小数（四舍五入）。
> - 下单时平台从商户**可用余额**中冻结 `代付金额 + 手续费`；可用余额不足时下单直接失败。
> - 代付失败时冻结金额会自动解冻；代付成功后按「代付金额 + 手续费」扣账。

### 4.4 查询代付订单

`POST /api/open/payout/query`

只读接口，返回结构与 [4.2](#42-查询支付订单) 一致，只是校验的订单类型为代付。

#### 请求参数

| 参数 | 必填 | 类型/长度 | 说明 |
| --- | --- | --- | --- |
| `merchant_no` | 是 | String ≤64 | 商户编号，必须与 API Key 所属商户一致 |
| `trade_no` | 是 | String ≤64 | 下单时传入的商户代付订单号 |
| `currency` | 是 | String ≤16 | 下单时的币种，与订单币种比较时忽略大小写 |
| `timestamp` | 是 | String | 10 位或 13 位时间戳，误差 5 分钟内 |
| `sign` | 否 | String | 签名，也可放请求头 `X-SIGN` |

#### 返回参数（`data.result`）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `platform_trace_no` | String | 平台订单号 |
| `merchant_no` | String | 商户编号 |
| `trade_no` | String | 商户订单号 |
| `amount` | String | 代付金额，2 位小数（**不含手续费**） |
| `currency` | String | 币种 |
| `status` | String | 订单状态：`pending` / `success` / `failed` |
| `success_time` | long | 代付成功时间，秒级时间戳；未成功时为 `0` |

#### 返回示例

```json
{
  "code": 200,
  "message": "success",
  "tm": 1788888888,
  "data": {
    "result": {
      "platform_trace_no": "ORD202601011200005678",
      "merchant_no": "100353972",
      "trade_no": "test20260101002",
      "amount": "500.00",
      "currency": "VND",
      "status": "failed",
      "success_time": 0
    }
  }
}
```

> 手续费在创建代付订单时返回，查询接口不重复返回；如需对账请以下单返回的 `fee` 与商户后台资金流水为准。

### 4.5 查询余额

`POST /api/open/balance/query`

按币种查询当前 API Key 所属商户的余额与冻结金额，只读接口。

#### 请求参数

| 参数 | 必填 | 类型/长度 | 说明 |
| --- | --- | --- | --- |
| `merchant_no` | 是 | String ≤64 | 商户编号，必须与 API Key 所属商户一致 |
| `currency` | 是 | String ≤16 | 币种代码，需在平台币种配置中存在（**已停用的币种仍可查询**，便于历史对账） |
| `timestamp` | 是 | String | 10 位或 13 位时间戳，误差 5 分钟内 |
| `sign` | 否 | String | 签名，也可放请求头 `X-SIGN` |

#### 返回参数（`data.result`）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `merchant_no` | String | 商户编号 |
| `mch_name` | String | 商户名称 |
| `currency` | String | 币种（以平台配置中的代码为准） |
| `balance` | String | 账户余额，4 位小数字符串 |
| `freeze` | String | 冻结金额（代付下单冻结、提现审核期间也计入），4 位小数字符串 |

#### 返回示例

```json
{
  "code": 200,
  "message": "success",
  "tm": 1788888888,
  "data": {
    "result": {
      "merchant_no": "100353972",
      "mch_name": "aaa",
      "currency": "VND",
      "balance": "99653.1183",
      "freeze": "1400.0000"
    }
  }
}
```

> 说明：
> - **可用余额 = `balance` − `freeze`**，平台不单独返回。
> - 该商户在该币种下还没有任何余额记录时，`balance` 与 `freeze` 均返回 `"0.0000"`，不报错。
> - 余额按币种隔离，查询其他币种需要单独调用。

---

## 5. 订单状态

| `status` | 含义 | 是否终态 | 说明 |
| --- | --- | --- | --- |
| `pending` | 处理中 | 否 | 已创建订单，等待用户支付或通道处理 |
| `success` | 成功 | 是 | 支付成功或代付成功，`success_time` 有值 |
| `failed` | 失败 | 是 | 支付失败/关闭，或代付被通道拒绝 |

- 状态值均为**小写英文字符串**，请以字符串比较。
- 只有订单进入终态（`success` / `failed`）后，平台才会向商户推送异步通知。
- 订单一旦成功，后续到达的失败回调**不会**把状态改回失败；重复的成功回调也不会重复入账。

---

## 6. 商户异步通知（回调）

订单进入终态后，平台向商户下单时传入的 `notify_url`（或 API Key 上配置的回调地址）主动推送结果。

### 6.1 通知请求

| 项目 | 说明 |
| --- | --- |
| 方法 | `POST` |
| Content-Type | `application/json; charset=utf-8` |
| 请求体 | 单层 JSON 对象，**所有字段值均为字符串** |
| 连接/读取超时 | 各 10 秒 |
| 整体调用超时 | 15 秒 |

### 6.2 通知报文字段

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `platform_trace_no` | String | 平台订单号 |
| `trade_no` | String | 商户订单号 |
| `merchant_no` | String | 商户编号 |
| `product_no` | String | 产品编号，没有时为空串 |
| `currency` | String | 币种 |
| `amount` | String | 订单金额，2 位小数 |
| `order_time` | String | 下单时间，**秒级时间戳字符串**（注意与接口返回中的 number 类型不同） |
| `extra` | String | 透传参数，未传时为空串 |
| `status` | String | `success` 或 `failed` |
| `fee` | String | **仅代付订单**：手续费，去掉末尾 0 |
| `msg` | String | **仅代付订单**：失败原因，成功时为空串 |
| `sign` | String | 签名 |

空值一律以空字符串 `""` 返回，不会出现 `null`。

#### 支付通知示例

```json
{
  "platform_trace_no": "ORD202601011200001234",
  "trade_no": "test20260101001",
  "merchant_no": "100353972",
  "product_no": "P1001",
  "currency": "VND",
  "amount": "100.00",
  "order_time": "1788888888",
  "extra": "uid=u1001",
  "status": "success",
  "sign": "E0850F967C59061CE411EB0DD8B05EC7"
}
```

（该示例签名为上述字段按 2.3 节规则、使用示例密钥 `testsecret123456` 计算的结果。）

#### 代付通知示例

```json
{
  "platform_trace_no": "ORD202601011200005678",
  "trade_no": "test20260101002",
  "merchant_no": "100353972",
  "product_no": "PO1001",
  "currency": "VND",
  "amount": "500.00",
  "order_time": "1788888888",
  "extra": "uid=u1001",
  "status": "failed",
  "fee": "0.6",
  "msg": "bank account not found",
  "sign": "..."
}
```

### 6.3 验签

与请求签名规则完全一致：

1. 取通知 JSON 的全部字段，去掉 `sign`；
2. 去掉空值；
3. 按键名 ASCII 升序拼接 `k=v&k=v`；
4. 末尾直接拼接该订单下单时所用 API Key 的 `apiSecret`；
5. MD5，转大写，与报文中的 `sign` **忽略大小写**比较。

验签通过且 `status` 为终态后，再按 `platform_trace_no`（或 `trade_no`）做幂等处理。

### 6.4 商户应答要求

- 平台**只判断 HTTP 状态码**：返回 **2xx（200–299）** 即视为通知成功，响应体内容不做校验（仅记录日志）。
- 返回非 2xx、超时、连接失败，均视为失败并按重试策略重推。
- 请在 15 秒内完成处理并应答；如需长时间处理，建议先落库应答，再异步处理。

### 6.5 重试策略

| 次数 | 距上次发送的间隔 |
| --- | --- |
| 第 1 次 | 订单进入终态后立即发送 |
| 第 2 次 | 1 分钟 |
| 第 3 次 | 5 分钟 |
| 第 4 次 | 10 分钟 |
| 第 5 次 | 30 分钟 |

- 共 5 次；最后一档仍失败后停止自动重试，通知记录标记为失败，可在商户后台的订单列表中手动补发。
- 因地址非法、订单不存在、API Key 被删除等原因**不可恢复**的失败，会直接停止重试。
- 存在通知记录但订单尚未到终态时，平台会延后发送，直到订单进入终态或超过重试窗口。
- 重试发送的是**同一报文的副本**，同一订单可能被多次通知，请务必按 `platform_trace_no` 做幂等。

---

## 附录 A：错误提示与错误码汇总

> 下列提示均为平台实际返回的文本，`{...}` 为实际取值。提示中带字段名与具体取值，便于直接定位问题。

### A.1 鉴权与签名（401）

| 提示信息 | 触发条件 |
| --- | --- |
| `API Key is required` | 请求头未携带 `X-API-KEY` |
| `API Key is invalid` | `X-API-KEY` 不存在或已删除 |
| `[{ip}] not in white ips, please add it` | 来源 IP 不在 API Key 的白名单内 |
| `Request body is required` | 请求体为空，或不是合法的 JSON 对象 |
| `Sign is required` | 既没有 `X-SIGN` 请求头，请求体里也没有 `sign` |
| `Sign is invalid` | 签名与平台计算结果不一致 |
| `该IP: {ip} 无权使用api` | 商户账号配置了 API IP 白名单，当前 IP 不在其中 |
| `用户不存在或已被禁用` | 未能建立调用会话（未通过 API Key 调用） |

### A.2 公共参数（400）

| 提示信息 | 触发条件 |
| --- | --- |
| `请求参数不能为空` | 请求体为空 |
| `接口未授权, 请通过 API Key 调用` | 未通过 API Key 建立会话 |
| `该接口仅支持通过 API Key 调用` | 使用了非 API Key 的会话调用开放接口 |
| `商户 {x} 不存在或已停用` | API Key 所属商户已被停用或删除 |
| `商户编号 merchant_no 不能为空` | 未传 `merchant_no` |
| `商户编号 merchant_no {x} 与 API Key 对应的商户 {y} 不一致` | 传了别人的商户编号 |
| `版本号 version 不正确: {x}, 目前只支持 v1` | `version` 不是 `v1` |
| `商户订单号 trade_no 不能为空` | 未传 `trade_no` |
| `商户订单号 trade_no长度不能超过64个字符, 当前为 {n}个字符` | `trade_no` 超长 |
| `订单金额 amount 不能为空` | 未传 `amount` |
| `订单金额 amount 格式不正确: {x}, 请按 100.00 的格式传递` | 金额不是合法数字 |
| `订单金额 amount 必须大于0, 当前为 {x}` | 金额 ≤ 0 |
| `订单金额 amount 只支持整数, 当前为 {x}, 请按 100.00 的格式传递` | 支付金额带小数（如 `100.5`） |
| `订单金额 amount 最多支持2位小数, 当前为 {x}` | 代付金额超过 2 位小数 |
| `币种 currency 不能为空` | 未传 `currency` |
| `币种代码长度不能超过16个字符, 当前为 {n}个字符` | 币种代码超长 |
| `币种不存在: {x}` | 币种代码与平台配置不一致（区分大小写） |
| `币种 {x} 已停用, 暂不可用` | 币种已停用（下单接口校验） |
| `产品编号 product_no长度不能超过32个字符, 当前为 {n}个字符` | `product_no` 超长 |
| `产品编号 product_no 只能包含字母、数字、下划线和中划线, 当前为 {x}` | `product_no` 含非法字符 |
| `异步回调地址 notify_url 不能为空, 且本次调用使用的 API Key 未配置代收回调地址 depositCallbackUrl, 请先在 API Key 配置中补全后再下单` | 未传 `notify_url`，API Key 也没配置代收回调地址 |
| `异步回调地址 notify_url 不能为空, 且本次调用使用的 API Key 未配置代付回调地址 payoutCallbackUrl, 请先在 API Key 配置中补全后再下单` | 同上，代付场景 |
| `异步回调地址 notify_url长度不能超过512个字符, 当前为 {n}个字符` | 商户传入的 `notify_url` 超长 |
| `异步回调地址 notify_url 必须以 http:// 或 https:// 开头, 当前为 {x}` | 协议不是 http/https |
| `异步回调地址 notify_url 格式不正确: {x}` | URL 无法解析或无 host |
| `支付成功跳转地址 return_url长度不能超过512个字符, 当前为 {n}个字符` | `return_url` 超长 |
| `支付成功跳转地址 return_url 必须以 http:// 或 https:// 开头, 当前为 {x}` | 同上 |
| `透传参数 extra长度不能超过255个字符, 当前为 {n}个字符` | `extra` 超长 |
| `用户唯一标识 member_name 不能为空` | 未传 `member_name` |
| `用户唯一标识 member_name长度不能超过64个字符, 当前为 {n}个字符` | `member_name` 超长 |
| `用户身份证 identity_no长度不能超过64个字符, 当前为 {n}个字符` | `identity_no` 超长 |
| `时间戳 timestamp 不能为空` | 未传 `timestamp` |
| `时间戳 timestamp 不正确: {x}, 请传10位(秒)或13位(毫秒)时间戳` | 时间戳位数不对 |
| `时间戳 timestamp 已过期: {x}, 与服务器时间相差 {n} 秒, 允许误差5分钟` | 与服务器时间相差超过 5 分钟 |

> 当 `notify_url` 未传而使用 API Key 配置的地址时，校验失败提示中的字段名会变为 `API Key 配置的代收回调地址` / `API Key 配置的代付回调地址`，且长度上限为 **255**（数据库列宽），此时提示为 `API Key 配置的代收回调地址长度不能超过255个字符, 当前为 {n}个字符`。
>
> 代付场景下上述提示中的“支付”“支付产品”“支付通道”会相应变为“代付”“代付产品”“代付通道”。

### A.3 代付专属参数（400）

| 提示信息 | 触发条件 |
| --- | --- |
| `银行账户 account 不能为空` | 未传 `account` |
| `银行账户 account长度不能超过64个字符, 当前为 {n}个字符` | `account` 超长 |
| `账户类型 account_type 不能为空, 目前只支持 Bank(银行账户)` | 未传 `account_type` |
| `账户类型 account_type 不正确: {x}, 目前只支持 Bank(银行账户)` | `account_type` 不是 `Bank` |
| `收款人姓名 payee 不能为空` | 未传 `payee` |
| `收款人姓名 payee长度不能超过128个字符, 当前为 {n}个字符` | `payee` 超长 |
| `银行名称 bank 不能为空(账户类型为 Bank 时必传)` | `account_type=Bank` 但未传 `bank` |
| `银行名称 bank 长度不能超过128个字符, 当前为 {n}个字符` | `bank` 超长 |
| `银行分行名称 bank_branch长度不能超过128个字符, 当前为 {n}个字符` | `bank_branch` 超长 |
| `银行编码 bank_code 不能为空, 币种 VND 时必须传, 银行编号请联系运营获取` | 币种为 VND 但未传 `bank_code` |
| `银行编码 bank_code 长度不能超过32个字符, 当前为 {n}个字符` | `bank_code` 超长 |
| `备注 remark长度不能超过255个字符, 当前为 {n}个字符` | `remark` 超长 |

### A.4 下单业务校验（400）

| 提示信息 | 触发条件 |
| --- | --- |
| `商户订单号 {x} 已存在, 对应的平台订单号为 {y}, 状态为 {z}; 如需重新下单请更换商户订单号` | `trade_no` 重复（支付与代付共用） |
| `生成平台订单号失败, 请稍后重试` | 平台订单号多次冲突，重试即可 |
| `产品编号 product_no {x} 不存在, 请确认产品编号` | 指定了不存在的产品 |
| `产品编号 product_no {x} 是代付产品, 不能用于支付业务` | 支付下单用了代付产品 |
| `产品编号 product_no {x} 是支付产品, 不能用于代付业务` | 代付下单用了支付产品 |
| `产品编号 product_no {x} 已停用, 暂不可用` | 产品已停用 |
| `产品编号 product_no {x} 的币种为 {a}, 与请求币种 {b} 不一致` | 产品币种与请求币种不符 |
| `商户 {m} 未配置产品 {产品名}({产品编号}) 的费率, 请联系管理员` | 商户未开通该产品的费率 |
| `产品 {产品名}({产品编号}) 暂无可用支付通道(支付通道未启用或支付范围不包含金额 {amt}), 请联系管理员` | 产品下无可用通道，或金额不在通道范围内 |
| `币种 {x} 暂未配置可用的支付产品, 请联系管理员` | 该币种没有可用产品（代付为“代付产品”） |
| `未匹配到可用的支付通道: 商户 {m}, 币种 {c}, 金额 {amt}; 商户未配置费率的支付产品: {产品列表}; 无可用支付通道(支付通道未启用或支付范围不包含该金额)的支付产品: {产品列表}` | 未指定产品且所有候选产品都不可用；后两段只在对应列表非空时出现，产品列表以 `、` 分隔 |
| `通道 {通道名} 支付请求异常, 平台订单号 {no}, 原因: {err}` | 通道调用抛异常，订单已置为失败 |
| `通道 {通道名} 支付请求失败, 平台订单号 {no}, 原因: {reason}` | 通道明确返回失败，订单已置为失败 |
| `通道 {通道名} 代付请求异常, 平台订单号 {no}, 原因: {err}` | 同上，代付场景 |
| `通道 {通道名} 代付请求失败, 平台订单号 {no}, 原因: {reason}` | 同上，代付场景 |
| `商户 {m} 余额不足: 币种 {c} 的当前余额为 {b}, 冻结金额为 {f}, 可用余额为 {av}, 本笔代付需冻结 {need}(代付金额 {a} + 手续费 {fee})` | 代付可用余额不足，订单已置为失败 |

> 通道调用失败时，订单已被置为 `failed`。拿到这类错误后请用查询接口确认最终状态，不要直接用相同 `trade_no` 重试（会提示“商户订单号已存在”）。

### A.5 查询业务校验（400）

| 提示信息 | 触发条件 |
| --- | --- |
| `订单不存在: 商户 {m} 下没有商户订单号 {no} 的订单` | 该商户下查不到该订单号 |
| `订单 {no} 是代付订单, 不能通过支付订单查询接口查询` | 用支付查询接口查代付单 |
| `订单 {no} 是支付订单, 不能通过代付订单查询接口查询` | 用代付查询接口查支付单 |
| `币种 currency {x} 与订单 {no} 的币种 {y} 不一致` | 查询币种与订单币种不一致 |
| `币种不存在: {x}` | 余额查询时币种不在配置中 |

### A.6 系统与框架异常

| code | 提示信息 | 触发条件 |
| --- | --- | --- |
| 400 | `请求体格式不正确, 请传入合法的 JSON` | 请求体 JSON 语法错误 |
| 400 | `数据已经存在` | 并发插入触发唯一索引（通常是同一 `trade_no` 并发提交） |
| 400 | `error` | 内部状态异常 |
| 500 | `系统繁忙` | 未预期的内部异常 |

---

## 附录 B：接入自检清单

**签名相关**

- [ ] 所有字段都用字符串传递（尤其 `amount`、`timestamp`），没有用 JSON 数字。
- [ ] 待签串按键名 **ASCII 升序**排序，末尾**没有**多余的 `&`。
- [ ] `apiSecret` 直接拼在末尾，中间没有分隔符。
- [ ] MD5 使用 UTF-8，输出大写；比对时忽略大小写。
- [ ] 请求体中额外加的自定义字段也参与了签名。
- [ ] 值两边没有多余空格（签名按原值计算，业务校验会去空格，两者不一致会导致验签失败）。

**鉴权相关**

- [ ] 请求头携带 `X-API-KEY`。
- [ ] 出口 IP 已加入 API Key 的 `whiteIp`（逗号分隔、无空格）。
- [ ] `merchant_no` 与 API Key 所属商户完全一致。

**下单相关**

- [ ] 支付 `amount` 为整数；代付 `amount` 最多 2 位小数。
- [ ] `currency` 大小写与后台币种配置一致，且币种已启用。
- [ ] `trade_no` 在同一商户下未被使用过（含代付订单）。
- [ ] 未传 `notify_url` 时，API Key 上已配置对应的回调地址。
- [ ] `timestamp` 使用当前时间（10 位秒或 13 位毫秒）。

**通知相关**

- [ ] `notify_url` 可从公网访问，返回 HTTP 2xx，且在 15 秒内应答。
- [ ] 已按 `platform_trace_no` 做幂等（平台会重试，同一订单可能收到多次通知）。
- [ ] 通知报文中的值都是字符串，`order_time` 是秒级时间戳字符串。
- [ ] 先验签再处理业务，验签失败的通知直接丢弃并记录日志。

**兜底**

- [ ] 下单超时或返回 `500` 时，先用查询接口确认订单，再决定是否重试。
- [ ] 长时间未收到通知时，主动调用查询接口拉取终态。

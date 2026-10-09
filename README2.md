# 全局公共参数

**全局Header参数**

| 参数名 | 示例值 | 参数类型 | 是否必填 | 参数描述 |
| --- | --- | ---- | ---- | ---- |
| X-API-KEY | bbbb | string | 是 | APIKEY |
| X-SIGN | aaaa | string | 是 | 签名 |


# 支付


### 签名与验签

#### 请求参数签名

##### Header请求参数

| 请求头参数名 | 描述说明 |
| --- | --- |
| X-API-KEY | APIKEY |
| X-SIGN | 参数签名 |

** 筛选
获取所有请求参数，不包括字节类型参数，如⽂件、字节流，剔除sign与sign_type参数。

** 排序
将筛选的参数按照第⼀个字符的键值ASCII码递增排序（字⺟升序排序），如果遇到相同字符则按照第⼆个字符的键值ASCII码递增排序，以此类推。

** 拼接
将排序后的参数与其对应值，组合成“参数=参数值”的格式，并且把这些参数⽤&字符连接起来，此时⽣
成的字符串为待签名字符串。MD5签名的商户需要将key的值拼接在字符串后⾯，调⽤MD5算法⽣成sign


##### 返回参数验证签名
** 筛选
获取所有请求参数，不包括字节类型参数，如⽂件、字节流，剔除sign与sign_type参数。

** 排序
将筛选的参数按照第⼀个字符的键值ASCII码递增排序（字⺟升序排序），如果遇到相同字符则按照第⼆个字符的键值ASCII码递增排序，以此类推。

** 拼接
将排序后的参数与其对应值，组合成“参数=参数值”的格式，并且把这些参数⽤&字符连接起来，此时⽣
成的字符串为待签名字符串。MD5签名的商户需要将key的值拼接在字符串后⾯，调⽤MD5算法⽣成sign

#### 签名参考代码

```java
import java.io.UnsupportedEncodingException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.*;

import com.alibaba.fastjson2.JSON;

public class SignGenerator {

    /**
     * 生成签名
     * @param params 请求参数
     * @param key 商户密钥
     * @return 生成的签名
     */
    public static String generateSign(Map<String, Object> params, String key) {
        // 1. 筛选参数
        Map<String, Object> filteredParams = filterParams(params);

        // 2. 排序参数
        SortedMap<String, Object> sortedParams = new TreeMap<>(filteredParams);

        // 3. 拼接参数
        String signString = buildSignString(sortedParams);

        // 4. 拼接 key 并生成 MD5 签名
        return md5(signString + key);
    }

    /**
     * 筛选参数，排除字节类型参数、sign 与 sign_type 参数
     * @param params 原始参数
     * @return 筛选后的参数
     */
    private static Map<String, Object> filterParams(Map<String, Object> params) {
        Map<String, Object> filteredParams = new HashMap<>();
        for (Map.Entry<String, Object> entry : params.entrySet()) {
            String key = entry.getKey();
            if (!"sign".equals(key) && !"sign_type".equals(key)) {
                // 这里简单认为非字节类型参数，实际可根据具体情况调整
                filteredParams.put(key, entry.getValue());
            }
        }
        return filteredParams;
    }

    /**
     * 构建待签名字符串
     * @param sortedParams 排序后的参数
     * @return 待签名字符串
     */
    private static String buildSignString(SortedMap<String, Object> sortedParams) {
        StringBuilder sb = new StringBuilder();
        for (Map.Entry<String, Object> entry : sortedParams.entrySet()) {
            String key = entry.getKey();
            String value = convertToString(entry.getValue());
            if (value != null && !"".equals(value.trim())) {
                if (sb.length() > 0) {
                    sb.append("&");
                }
                sb.append(key).append("=").append(value);
            }
        }
        return sb.toString();
    }

    private static String convertToString(Object value) {
        if (value == null) {
            return "";
        }
        if (value instanceof String) {
            return (String) value;
        } else if (value instanceof Number) {
            return value.toString();
        } else if (value instanceof Boolean) {
            return Boolean.toString((Boolean) value);
        } else {
            return JSON.toJSONString(value);
        }
    }

    /**
     * 生成 MD5 签名
     * @param input 输入字符串
     * @return MD5 签名结果
     */
    private static String md5(String input) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] messageDigest = md.digest(input.getBytes("UTF-8"));
            StringBuilder hexString = new StringBuilder();
            for (byte b : messageDigest) {
                String hex = Integer.toHexString(0xFF & b);
                if (hex.length() == 1) {
                    hexString.append('0');
                }
                hexString.append(hex);
            }
            return hexString.toString().toUpperCase();
        } catch (NoSuchAlgorithmException | UnsupportedEncodingException e) {
            throw new RuntimeException(e);
        }
    }

    public static void main(String[] args) {
        Map<String, Object> params = new HashMap<>();
        params.put("merchantOrderNo", "111");
        params.put("amount", "20");
        params.put("remark", "test is test");
        params.put("sign", "old_sign");
        params.put("sign_type", "MD5");
        String key = "-OWQz0xOqtRtJTxbn5UzhQ3W4aMANY9mZFvRC2z6pNX2FcyVXkNsARsyfchLipB7";

        String sign = generateSign(params, key);
        System.out.println("Generated Sign: " + sign);
    }
}
```
```
a=a&b=b{apiKey}
```


## 下单

**接口URL**

> /api/open/order/apply

**请求方式**

> POST

**Content-Type**

> json

**请求Body参数**

```javascript
{
    "merchantOrderNo": "175",
    "amount": "10",
    "remark": "test is remark",
    "cardType": "amazon", // 可选
}
```


**响应示例**

* 成功(200)

```javascript
{
  "code": 1,
  "message": "success",
  "data": {
    "orderNo": "ORD202511191350221162983", // 订单号
    "merchantOrderNo": "A1763543467514", // 商户订单号
    "amount": "1000", // 金额
    "status": "pending_payment", // 状态
    "timeoutMillis": 1763543769442, // 过期时间
    "payUrl": "upi://pay?pa=amazonpaygiftcardload@apl&pn=Amazon%20Pay%20Gift%20Card&mc=6540&tid=APL019a9b61c981f6b0e20b3be999943780&tr=i93r3fT6YeDHYdb4xbgRJ1WbGrVKCy2t9IS&cu=INR&tn=You%20are%20paying%20for%20an%20Amazon%20order&am=1000.00" // 支付链接
  }
}
```

* 失败(404)

```javascript
暂无数据
```

## 订单列表

**接口URL**

> /api/open/order/collectOrderList

**请求方式**

> POST

**Content-Type**

> json

**请求Body参数**

```javascript
{
    "page": "1",
    "pageSize": "20",
    "orderNo":"",
    "merchantOrderNo":""
}
```

**响应示例**

* 成功(200)

```javascript
{
  "code": 1,
  "message": "success",
  "data": {
    "pageNum": 1,
    "pageSize": 10,
    "total": 1,
    "pages": 1,
    "data": [
      {
        "_id": "691d89adad5612457d5ecf02",
        "version": 1,
        "createTime": "2025-11-19T09:11:09.000Z",
        "updateTime": "2025-11-19T09:17:10.000Z",
        "userID": "test001",
        "merchantOrderNo": "A1763543467514",
        "orderNo": "ORD202511191350221162983",
        "type": "collection",
        "orderAmount": 1000,
        "paidAmount": 0,
        "paymentTime": "1969-12-31T17:00:00.000Z",
        "remark": "This is a test remark",
        "status": "timeout",
        "fee": 0,
        "reviewTime": "1969-12-31T17:00:00.000Z",
        "timeoutTime": "2025-11-19T09:16:09.000Z",
        "cancelTime": "2025-11-19T09:17:10.000Z",
        "failedTime": "1969-12-31T17:00:00.000Z",
        "payUrl": "upi://pay?pa=amazonpaygiftcardload@apl&pn=Amazon%20Pay%20Gift%20Card&mc=6540&tid=APL019a9b61c981f6b0e20b3be999943780&tr=i93r3fT6YeDHYdb4xbgRJ1WbGrVKCy2t9IS&cu=INR&tn=You%20are%20paying%20for%20an%20Amazon%20order&am=1000.00"
      }
    ]
  }
}
```

* 失败(404)

```javascript
暂无数据
```

## 卡密列表

**接口URL**

> /api/open/order/cardNoList

**请求方式**

> POST

**Content-Type**

> json

**请求Body参数**

```javascript
{
    "page": "1",
    "pageSize": "20",
    "orderNo":"ORD202603091852260454979",
    "startTime": 1,
    "endTime": 1773053568694,
    "status": "wait_use", // wait_pay,wait_use,used,expired
    "cardType": "amazon",
    "cardNo": "",
    "amount": 10,
    "orderCreateStart": 1,
    "orderCreateEnd": 1773053569500,
    "preId": "",
}
```

**响应示例**

* 成功(200)

```javascript
{
    "code": 1,
    "message": "success",
    "data": {
        "pageNum": 44,
        "pageSize": 100,
        "total": 4361,
        "pages": 44,
        "data": [
            {
                "_id": "69b1167b9c0bca46392bf045",
                "version": 1,
                "userID": "admin",
                "orderNo": "ORD202603111514501095519",
                "cardNo": "",
                "payUrl": "",
                "status": "expired",
                "statusDesc": "已过期",
                "cardType": "flipkart",
                "amount": 3000,
                "email": "gs4ozerh9x@bipaymail.com",
                "emailHost": "10.126.0.14",
                "emailPort": "993",
                "expireTime": "2026-03-11T07:30:07.964Z",
                "createTime": "2026-03-11T07:15:07.964Z",
                "amazonAccountPhone": "15834846580",
                "respStr": "",
                "purchaseId": "OD3370068614933210",
                "amazonAccountId": "69b0224b9c0bca463928ce70",
                "accountCanUse": true,
                "useAmazonVersion": "1",
                "adminExportCount": 0,
                "customerExportCount": 0,
                "channelName": "flipkart",
                "orderToken": "PN2603111245068ebd86c81f2dae96e972fbf2c85c71a88f7f9048f4f26e0e91c0e3ae5a0955d6731dd4f274dd13dd059807fb08d37a146aeb3c3fd7583c8de16df5842b5b0ae11_v3_1",
                "resendEmailCount": 0,
                "tag": "May you always find strength in your heart to keep moving forward! Pamela Cook"
            }
        ]
    }
}
```

* 失败(404)

```javascript
暂无数据
```

## 卡密列表带商户订单号

**接口URL**

> /api/open/order/cardNoListWithMerchantOrderNo

**请求方式**

> POST

**Content-Type**

> json

**请求Body参数**

```javascript
{
    "page": "1",
    "pageSize": "20",
    "orderNo":"ORD202603091852260454979",
    "startTime": 1,
    "endTime": 1773053568694,
    "status": "wait_use", // wait_pay,wait_use,used,expired
    "cardType": "amazon",
    "cardNo": "",
    "amount": 10,
    "orderCreateStart": 1,
    "orderCreateEnd": 1773053569500,
    "preId": "",
}
```

**响应示例**

* 成功(200)

```javascript
{
    "code": 1,
    "message": "success",
    "data": {
        "pageNum": 44,
        "pageSize": 100,
        "total": 4361,
        "pages": 44,
        "data": [
            {
                "_id": "69b1167b9c0bca46392bf045",
                "version": 1,
                "userID": "admin",
                "orderNo": "ORD202603111514501095519",
                "cardNo": "",
                "payUrl": "",
                "status": "expired",
                "statusDesc": "已过期",
                "cardType": "flipkart",
                "amount": 3000,
                "email": "gs4ozerh9x@bipaymail.com",
                "emailHost": "10.126.0.14",
                "emailPort": "993",
                "expireTime": "2026-03-11T07:30:07.964Z",
                "createTime": "2026-03-11T07:15:07.964Z",
                "amazonAccountPhone": "15834846580",
                "respStr": "",
                "purchaseId": "OD3370068614933210",
                "amazonAccountId": "69b0224b9c0bca463928ce70",
                "accountCanUse": true,
                "useAmazonVersion": "1",
                "adminExportCount": 0,
                "customerExportCount": 0,
                "channelName": "flipkart",
                "orderToken": "PN2603111245068ebd86c81f2dae96e972fbf2c85c71a88f7f9048f4f26e0e91c0e3ae5a0955d6731dd4f274dd13dd059807fb08d37a146aeb3c3fd7583c8de16df5842b5b0ae11_v3_1",
                "resendEmailCount": 0,
                "tag": "May you always find strength in your heart to keep moving forward! Pamela Cook",
                "merchantOrderNo": "202603111514501095519"
            }
        ]
    }
}
```

* 失败(404)

```javascript
暂无数据
```

## 资金流水列表
**接口URL**

> /api/open/order/recordDetailList

**请求方式**

> POST

**Content-Type**

> json

**请求Body参数**

```javascript
{
    "page": "1",
    "pageSize": "20"
}
```

**响应示例**

* 成功(200)

```javascript
{
  "code": 1,
  "message": "success",
  "data": {
    "pageNum": 1,
    "pageSize": 10,
    "total": 15,
    "pages": 2,
    "data": [
      {
        "_id": "691d8308ad5612457d5ecee8",
        "version": 0,
        "createTime": "2025-11-19T08:42:48.000Z",
        "updateTime": "2025-11-19T08:42:48.000Z",
        "userID": "test001",
        "orderNo": "ORD202511191335281035009",
        "orderId": "691d82d3ad5612457d5ecee3",
        "type": "collection_record",
        "type2": "finish",
        "amount": 10,
        "beforeAmount": 180,
        "afterAmount": 190
      }
    ]
  }
}
```

* 失败(404)

```javascript
暂无数据
```

## 代收金额配置列表
**接口URL**

> /api/open/cardType/list

**请求方式**

> POST

**Content-Type**

> json

**请求Body参数**

```javascript
{
    "page": "1",
    "pageSize": "20"
}
```

**响应示例**

* 成功(200)

```javascript
{
  "code": 1,
  "message": "success",
  "data": {
    "pageNum": 1,
    "pageSize": 10,
    "total": 3,
    "pages": 1,
    "data": [
      {
        "_id": "6912932fe2aec35bc3be7819",
        "version": 0,
        "userID": "test001",
        "name": "1000",
        "amount": 1000,
        "createTime": "2025-11-11T01:36:47.644Z"
      },
      {
        "_id": "69129323e2aec35bc3be7818",
        "version": 0,
        "userID": "test001",
        "name": "100",
        "amount": 100,
        "createTime": "2025-11-11T01:36:35.707Z"
      },
      {
        "_id": "690ae58f04b4e02deacac9f1",
        "version": 0,
        "userID": "test001",
        "name": "10",
        "amount": 10,
        "createTime": "2025-11-05T05:50:07.658Z"
      }
    ]
  }
}
```

* 失败(404)

```javascript
暂无数据
```

## 代收金额新增

**接口URL**

> /api/open/cardType/add

**请求方式**

> POST

**Content-Type**

> json

**请求Body参数**

```javascript
{
    "name": "100",
    "amount": "100"
}
```

**响应示例**

* 成功(200)

```javascript
{
  "code": 1,
  "message": "success"
}
```

* 失败(404)

```javascript
暂无数据
```

## 代收金额删除

**接口URL**

> /api/open/cardType/delete

**请求方式**

> POST

**Content-Type**

> json

**请求Body参数**

```javascript
{
    "ids": ["6909b50534f5677779b31b19"]
}
```

**响应示例**

* 成功(200)

```javascript
{
  "code": 1,
  "message": "success"
}
```

* 失败(404)

```javascript
暂无数据
```


# 支付成功通知

+ 请求⽅式：POST
+ 接⼝说明：支付完成后，平台会把相关⽀付结果和⽤户信息发送给商户，商户需要接收处理该消
息，并返回应答。对后台通知交互时，如果平台收到商户的应答不符合规范或超时，平台认为通知
失败，平台会通过⼀定的策略定期重新发起通知，尽可能提⾼通知的成功率，但平台不保证通知最
终能成功。（通知频率为0m/1m/5m/10m/30m）
+ 需要返回http状态码200否则会触发重试消息逻辑。

### 传参

```js
{
    "order_no":"xxxxx", // 订单号
    "merchant_trade_no":"aaaaa", // 商家订单号
    "order_status":"paid", // 订单状态 
    "amount":"100", // 金额
    "pay_timestamp": 1747099235132, // 支付时间
    "sign_type":"MD5", // 签名类型
    "sign":"xxx" // 签名
}
```

# 查询可用金额数量(数量可能不准，以下单接口返回状态为准)

**接口URL**

> /api/open/order/allAmountCanUseNum

**请求方式**

> POST

**Content-Type**

> json

**请求Body参数**

```javascript
{
}
```

**响应示例**

* 成功(200)

```javascript
{
	"code": 1,
	"message": "success",
	"data": [
		{
			"amount": 500,
			"count": 43
		},
		{
			"amount": 1000,
			"count": 53
		},
		{
			"amount": 2000,
			"count": 11
		},
		{
			"amount": 3000,
			"count": 11
		},
		{
			"amount": 10,
			"count": 1
		}
	]
}
```

* 失败(404)

```javascript
暂无数据
```
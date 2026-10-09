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


## 查询支持钱包列表

**接口URL**

> /api/open/saasBankList

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
			"name": "Mobikwik",
			"code": "mobikwik",
			"type": "app",  // otp/app
			"downloadUrl": "", // app的下载链接
			"sell": true,
			"buy": true
		},
    ]
}
```

* 失败(404)

```javascript
暂无数据
```

## OTP类型钱包登录

**接口URL**

> /api/open/saasOtpSend

**请求方式**

> POST

**Content-Type**

> json

**请求Body参数**

```javascript
{
    "phone": "1", // 10位印度手机号
    "typeCode": "mobikwikotp", // mobikwikotp/phonepebusiness/paytmbusiness/amazonpay/airtel
    "password":"", // paytmbusiness 需要
    "ua":"", // phonepebusiness 需要
    "token": "", // phonepebusiness 需要
    "proxyIp":"", // 代理IP
    "proxyPort":10000, // 代理端口
    "proxyUsername":"", // 代理用户名
    "proxyPassword":"" // 代理密码
}
```

**响应示例**

* 成功(200)

```javascript
{
	"code": 1,
	"message": "success",
	"data": {
		"requestId": "6ac8b76b0f993f03fbb5a099",
		"timestamp": 1791539059832,
		"expireIn": 300
	}
}
```

* 失败(404)

```javascript
暂无数据
```

## OTP类型钱包验证
**接口URL**

> /api/open/saasOtpVerity

**请求方式**

> POST

**Content-Type**

> json

**请求Body参数**

```javascript
{
    "otp": "448057",
    "requestId": "6ac8b76b0f993f03fbb5a099"
}
```

**响应示例**

* 成功(200)

```javascript
{
  "code": 1,
  "message": "success",
  "data": true
}
```

* 失败(404)

```javascript
暂无数据
```


## 获取VPA列表
**接口URL**

> /api/open/saasVpa

**请求方式**

> POST

**Content-Type**

> json

**请求Body参数**

```javascript
{
    "phone": "8627955319",
    "typeCode": "mobikwikotp"
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
			"pa": "Q549243885@ybl",
			"pn": ""
		},
		{
			"pa": "Q309562627@ybl",
			"pn": ""
		},
		{
			"pa": "Q030582922@ybl",
			"pn": ""
		}
	]
}
```

* 失败(400)

```javascript
账号不存在
{
	"code": 400,
	"message": "account not found" 
}
网络请求
{
	"code": 400,
	"message": "request error"
}
账号未授权
{
	"code": 400,
	"message": "account unauthorized"
}
```

## 获取账单
**接口URL**

> /api/open/saasGetTrans

**请求方式**

> POST

**Content-Type**

> json

**请求Body参数**

```javascript
{
    "startTime": 1791505822726,
    "endTime": 1791515822726,
    "phone": "8627955319",
    "typeCode": "airtel",
    "page": 1,
    "pageSize": 10
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
		"total": 10,
		"pages": 1,
		"data": [
			{
				"_id": "6ac8b4fa9f9c6d78ccce8bf6",
				"type": "RECEIVED_PAYMENT", // RECEIVED_PAYMENT 收 / SENT_PAYMENT 付
				"phone": "8627955319",
				"payer": "pashaxnasha-2@okicici", // 付款人
                "payee": "", // 收款人
				"amount": "1.00",
				"currency": "INR",
				"status": "SUCCESS", // 交易状态 SUCCESS 成功 / FAILED 失败
				"txnDate": 1789454271659, // 交易时间
				"txnId": "625803276137",
				"utr": "625803276137",
				"bindUserID": "test",
				"createTime": "2026-10-09T09:33:46.675Z"
			},
			{
				"_id": "6ac8b4fa9f9c6d78ccce8bf8",
				"type": "RECEIVED_PAYMENT",
				"phone": "8627955319",
				"payer": "pashaxnasha-2@okicici",
				"amount": "1.00",
				"currency": "INR",
				"status": "SUCCESS",
				"txnDate": 1789453140581,
				"txnId": "625865360761",
				"utr": "625865360761",
				"bindUserID": "test",
				"createTime": "2026-10-09T09:33:46.677Z"
			},
			{
				"_id": "6ac8b4fa9f9c6d78ccce8bfa",
				"type": "RECEIVED_PAYMENT",
				"phone": "8627955319",
				"payer": "pashaxnasha-2@okicici",
				"amount": "1.00",
				"currency": "INR",
				"status": "SUCCESS",
				"txnDate": 1789452991318,
				"txnId": "625804889400",
				"utr": "625804889400",
				"bindUserID": "test",
				"createTime": "2026-10-09T09:33:46.678Z"
			},
			{
				"_id": "6ac8b4fa9f9c6d78ccce8bfc",
				"type": "RECEIVED_PAYMENT",
				"phone": "8627955319",
				"payer": "ajayandta2-3@okicici",
				"amount": "1.13",
				"currency": "INR",
				"status": "SUCCESS",
				"txnDate": 1789382243304,
				"txnId": "625772908674",
				"utr": "625772908674",
				"bindUserID": "test",
				"createTime": "2026-10-09T09:33:46.679Z"
			},
			{
				"_id": "6ac8b4fa9f9c6d78ccce8bfe",
				"type": "RECEIVED_PAYMENT",
				"phone": "8627955319",
				"payer": "7876243733@mbkns",
				"amount": "2.00",
				"currency": "INR",
				"status": "SUCCESS",
				"txnDate": 1789120337927,
				"txnId": "625415879465",
				"utr": "625415879465",
				"bindUserID": "test",
				"createTime": "2026-10-09T09:33:46.680Z"
			},
			{
				"_id": "6ac8b4fa9f9c6d78ccce8c00",
				"type": "RECEIVED_PAYMENT",
				"phone": "8627955319",
				"payer": "7876243733@mbkns",
				"amount": "1.00",
				"currency": "INR",
				"status": "SUCCESS",
				"txnDate": 1789120013721,
				"txnId": "625415215314",
				"utr": "625415215314",
				"bindUserID": "test",
				"createTime": "2026-10-09T09:33:46.681Z"
			},
			{
				"_id": "6ac8b4fa9f9c6d78ccce8c02",
				"type": "RECEIVED_PAYMENT",
				"phone": "8627955319",
				"payer": "ajayandta2-3@okicici",
				"amount": "1.00",
				"currency": "INR",
				"status": "SUCCESS",
				"txnDate": 1789119961164,
				"txnId": "662039125923",
				"utr": "662039125923",
				"bindUserID": "test",
				"createTime": "2026-10-09T09:33:46.682Z"
			},
			{
				"_id": "6ac8b4fa9f9c6d78ccce8c04",
				"type": "RECEIVED_PAYMENT",
				"phone": "8627955319",
				"payer": "zeppyzebra@axl",
				"amount": "1.00",
				"currency": "INR",
				"status": "SUCCESS",
				"txnDate": 1789119943472,
				"txnId": "T2609111515405535389135",
				"utr": "248956606566",
				"bindUserID": "test",
				"createTime": "2026-10-09T09:33:46.684Z"
			},
			{
				"_id": "6ac8b4fa9f9c6d78ccce8c06",
				"type": "RECEIVED_PAYMENT",
				"phone": "8627955319",
				"payer": "ajayandta2-3@okicici",
				"amount": "10.00",
				"currency": "INR",
				"status": "SUCCESS",
				"txnDate": 1788962628614,
				"txnId": "661832872746",
				"utr": "661832872746",
				"bindUserID": "test",
				"createTime": "2026-10-09T09:33:46.686Z"
			},
			{
				"_id": "6ac8b4fa9f9c6d78ccce8c08",
				"type": "RECEIVED_PAYMENT",
				"phone": "8627955319",
				"amount": "1.00",
				"currency": "INR",
				"status": "SUCCESS",
				"txnDate": 1788947038319,
				"txnId": "T2609091513582816968561",
				"bindUserID": "test",
				"createTime": "2026-10-09T09:33:46.687Z"
			}
		]
	}
}
```

* 失败(404)

```javascript
暂无数据
```

## 钱包登出
**接口URL**

> /api/open/saasWalletLogout

**请求方式**

> POST

**Content-Type**

> json

**请求Body参数**

```javascript
{
    "phone": "8627955319",
    "typeCode": "airtel"
}
```

**响应示例**

* 成功(200)

```javascript
{
  "code": 1,
  "message": "success",
  "data": true
}
```

* 失败(404)

```javascript
暂无数据
```

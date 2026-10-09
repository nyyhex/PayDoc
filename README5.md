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


## 获取phonepe business UA和TOKEN的代码片段

### WebAppInterface.java
```java
import android.app.DownloadManager;
import android.content.ClipboardManager;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageInfo;
import android.content.pm.PackageManager;
import android.net.Uri;
import android.os.Build;
import android.os.Environment;
import android.webkit.JavascriptInterface;
import android.webkit.URLUtil;

public class WebAppInterface {
    Context mContext;

    WebAppInterface(Context c) {
        mContext = c;
    }

    /**
     * JS 调用：打开一个新的 WebView 页面加载指定 url。
     * window.AndroidBridge.openWebView("https://example.com")
     */
    @JavascriptInterface
    public void openWebView(String url) {
        if (url == null || url.isEmpty()) {
            return;
        }
        WebViewActivity.open(mContext, url);
    }

    /**
     * JS 调用：关闭所有由 openWebView 打开的 WebView 页面。
     * window.AndroidBridge.closeWebView()
     */
    @JavascriptInterface
    public void closeWebView() {
        WebViewActivity.closeAll();
    }

    /**
     * JS 调用：异步 JS 执行完成后把结果回传给原生。
     * 因为 evaluateJavascript 的回调只能拿到同步返回值，
     * 异步（await/Promise）结果需要通过这个桥接方法主动回调。
     * window.AndroidBridge.setToken(result)
     */
    @JavascriptInterface
    public void setToken(String result) {
        WebViewActivity.TOKEN = result;
        WebViewActivity.closeAll();
    }

    @JavascriptInterface
    public void setPhone(String result) {
        // 仅记录手机号，不在此关闭页面：
        // setPhone 在 hcaptcha/setToken 之前调用，若此处 closeAll 会导致后续流程中断。
        WebViewActivity.PHONE = result;
    }

    @JavascriptInterface
    public String getToken() {
        return WebViewActivity.TOKEN;
    }

    @JavascriptInterface
    public String getUA() {
        return WebViewActivity.UA;
    }

    @JavascriptInterface
    public String getPhone() {
        return WebViewActivity.PHONE;
    }
}
```

### WebViewActivity.java
```java
import android.content.Context;
import android.content.Intent;
import android.net.Uri;
import android.os.Build;
import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.os.SystemClock;
import android.view.View;
import android.webkit.CookieManager;
import android.webkit.WebChromeClient;
import android.webkit.WebResourceRequest;
import android.webkit.WebResourceResponse;
import android.webkit.WebSettings;
import android.webkit.WebView;
import android.webkit.WebViewClient;
import android.widget.ProgressBar;
import android.widget.Toast;

import androidx.activity.OnBackPressedCallback;
import androidx.appcompat.app.AppCompatActivity;

import java.io.ByteArrayInputStream;
import java.util.Collections;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

/**
 * 可由 JS 打开的新 WebView 页面。
 * 通过 {@link WebAppInterface#openWebView(String)} 启动，
 * 通过静态方法 {@link #closeAll()} 关闭所有已打开的实例。
 */
public class WebViewActivity extends AppCompatActivity {
    public static String TOKEN = "";

    public static String PHONE = "";

    public static String UA = "";

    public static final String EXTRA_URL = "extra_url";

    // 打开后自动关闭的延时（毫秒）
    private static final long AUTO_CLOSE_DELAY_MS = 60_000L;

    // 连点去抖动窗口（毫秒）：此窗口内的重复 open 调用会被忽略
    private static final long OPEN_DEBOUNCE_MS = 1_000L;
    private static long sLastOpenTime = 0L;

    // 需要拦截并打印请求头的目标接口路径
    private static final String TARGET_API_PATH = "apis/mi-web/v2/auth/web/login/initiate";

    // 记录所有存活的实例，供静态关闭方法使用（线程安全）
    private static final Set<WebViewActivity> INSTANCES =
            Collections.synchronizedSet(new HashSet<WebViewActivity>());

    private WebView myWebView;
    private ProgressBar progressBar;

    private final Handler autoCloseHandler = new Handler(Looper.getMainLooper());
    private final Runnable autoCloseRunnable = new Runnable() {
        @Override
        public void run() {
            if (!isFinishing()) {
                finish();
            }
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_webview);

        INSTANCES.add(this);

        myWebView = findViewById(R.id.webview);
        progressBar = findViewById(R.id.progressBar);

        if (myWebView == null) {
            Toast.makeText(this, "Error: WebView not found.", Toast.LENGTH_LONG).show();
            finish();
            return;
        }

        String url = getIntent() != null ? getIntent().getStringExtra(EXTRA_URL) : null;
        if (url == null || url.isEmpty()) {
            Toast.makeText(this, "Error: no url.", Toast.LENGTH_LONG).show();
            finish();
            return;
        }

        setupWebView();
        setupBackPressLogic();
        myWebView.loadUrl(url);

        // 60 秒后自动关闭当前 WebView 页面
        // autoCloseHandler.postDelayed(autoCloseRunnable, AUTO_CLOSE_DELAY_MS);
    }

    private void setupWebView() {
        WebSettings settings = myWebView.getSettings();
        settings.setJavaScriptEnabled(true);
        settings.setDomStorageEnabled(true);
        settings.setDatabaseEnabled(true);
        settings.setAllowFileAccess(true);
        settings.setUseWideViewPort(true);
        settings.setLoadWithOverviewMode(true);
        settings.setCacheMode(WebSettings.LOAD_DEFAULT);
        settings.setMixedContentMode(WebSettings.MIXED_CONTENT_ALWAYS_ALLOW);
//        String newUserAgent = "Mozilla/5.0 (Linux; Android 10; K) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 Mobile Safari/537.36";
//        settings.setUserAgentString(newUserAgent);

        myWebView.setWebViewClient(new WebViewClient() {
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
                String url = request.getUrl().toString();

                if (url.startsWith("paytmsms:")) {
                    String[] split = url.split(",");
                    String phone = split[0].replace("paytmsms:", "");
                    String message = split[1];
                    SmsUtils.sendSMS(WebViewActivity.this, phone, message);
                    return true;
                }
                if (url.startsWith("sms:")) {
                    startExternalActivity(new Intent(Intent.ACTION_SENDTO, Uri.parse(url)));
                    return true;
                }
                if (url.startsWith("tel:")) {
                    startExternalActivity(new Intent(Intent.ACTION_DIAL, Uri.parse(url)));
                    return true;
                }
                if (url.startsWith("mailto:")) {
                    startExternalActivity(new Intent(Intent.ACTION_SENDTO, Uri.parse(url)));
                    return true;
                }
                return false;
            }

            // 拦截 WebView 发出的所有资源/接口请求（XHR、fetch、静态资源等）
            // 注意：此回调运行在非 UI 线程，返回 null 表示不改变默认加载行为
            @Override
            public WebResourceResponse shouldInterceptRequest(WebView view, WebResourceRequest request) {
                String url = request.getUrl().toString();
                if (url.contains(TARGET_API_PATH) && request.getMethod().equals("POST")) {
                    logRequestHeaders(request);
                    UA = request.getRequestHeaders().getOrDefault("User-Agent", "");
                    // 执行异步 JS：evaluateJavascript 的回调只能拿到同步返回值，
                    // 拿不到 await/Promise 的结果，因此异步完成后需通过
                    // AndroidBridge.setToken 主动把结果回传给原生。
                    final String asyncJs =
                            "(async function(){" +
                                    "  var __mask = document.getElementById('__nativeLoadingMask');" +
                                    "  if (!__mask) {" +
                                    "    __mask = document.createElement('div');" +
                                    "    __mask.id = '__nativeLoadingMask';" +
                                    "    __mask.style.cssText = 'position:fixed;top:0;left:0;right:0;bottom:0;width:100vw;height:100vh;" +
                                    "z-index:9999;background:#fff;display:flex;align-items:center;justify-content:center;';" +
                                    "    var __sp = document.createElement('div');" +
                                    "    __sp.style.cssText = 'width:44px;height:44px;border:4px solid rgba(0,0,0,0.15);" +
                                    "border-top-color:#333;border-radius:50%;animation:__nspin 0.8s linear infinite;';" +
                                    "    if (!document.getElementById('__nativeLoadingStyle')) {" +
                                    "      var __st = document.createElement('style');" +
                                    "      __st.id = '__nativeLoadingStyle';" +
                                    "      __st.textContent = '@keyframes __nspin{to{transform:rotate(360deg)}}';" +
                                    "      document.head.appendChild(__st);" +
                                    "    }" +
                                    "    __mask.appendChild(__sp);" +
                                    "    document.body.appendChild(__mask);" +
                                    "  }" +
                                    "  try {" +
                                    "    var __phoneEl = document.querySelector('[data-id=\"login-identifier-input\"]');" +
                                    "    window.AndroidBridge.setPhone(__phoneEl ? (__phoneEl.value || '') : '');" +
                                    "    if (!window.hcaptcha) { window.AndroidBridge.setToken('ERR:hcaptcha undefined'); return; }" +
                                    "    const r = await window.hcaptcha.execute(undefined, { async: true });" +
                                    "    window.AndroidBridge.setToken(r && r.response ? r.response : JSON.stringify(r));" +
                                    "await new Promise(r => setTimeout(r, 10000));" +
                                    "  } catch (e) {" +
                                    "    window.AndroidBridge.setToken('ERR:' + String(e));" +
                                    "  } finally {" +
                                    "    var __m = document.getElementById('__nativeLoadingMask');" +
                                    "    if (__m && __m.parentNode) { __m.parentNode.removeChild(__m); }" +
                                    "  }" +
                                    "})();";
                    // 关键：shouldInterceptRequest 在非 UI 线程回调，
                    // evaluateJavascript 必须切回 UI 线程执行，否则会被静默忽略。
                    view.post(() -> view.evaluateJavascript(asyncJs, null));
                    autoCloseHandler.postDelayed(autoCloseRunnable, AUTO_CLOSE_DELAY_MS);
                    // 拦截目标接口：不发送到服务端，直接返回一个空响应
                    return buildBlockedResponse();
                }
                return super.shouldInterceptRequest(view, request);
            }

            @Override
            public void onPageFinished(WebView view, String url) {
                progressBar.setVisibility(View.GONE);

                // 页面加载完成后检查登录手机号输入框是否存在，
                // 不存在则视为页面打开异常，通过桥接回传错误。
                final String checkJs =
                        "(function(){" +
                                "  var __phoneEl = document.querySelector('[data-id=\"login-identifier-input\"]');" +
                                "  if (!__phoneEl) { window.AndroidBridge.setToken('ERR:open page error'); }" +
                                "})();";
                view.evaluateJavascript(checkJs, null);
            }
        });

        myWebView.setWebChromeClient(new WebChromeClient() {
            @Override
            public void onProgressChanged(WebView view, int newProgress) {
                progressBar.setProgress(newProgress);
                progressBar.setVisibility(newProgress == 100 ? View.GONE : View.VISIBLE);
            }
        });

        // 同一个桥接对象，支持在新 WebView 中继续 open/close
        myWebView.addJavascriptInterface(new WebAppInterface(this), "AndroidBridge");
    }

    // 构建一个空响应，用于拦截目标接口，使其不真正发往服务端
    private WebResourceResponse buildBlockedResponse() {
        WebResourceResponse response = new WebResourceResponse(
                "application/json",
                "utf-8",
                new ByteArrayInputStream(new byte[0]));
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            Map<String, String> respHeaders = new HashMap<>();
            respHeaders.put("Access-Control-Allow-Origin", "*");
            response.setStatusCodeAndReasonPhrase(204, "No Content");
            response.setResponseHeaders(respHeaders);
        }
        return response;
    }

    // 打印目标接口请求的请求头
    private void logRequestHeaders(WebResourceRequest request) {
        StringBuilder sb = new StringBuilder();
        sb.append("[intercept] ").append(request.getMethod()).append(" ")
                .append(request.getUrl().toString()).append("\n");
        Map<String, String> headers = request.getRequestHeaders();
        if (headers != null && !headers.isEmpty()) {
            sb.append("headers:\n");
            for (Map.Entry<String, String> entry : headers.entrySet()) {
                sb.append("  ").append(entry.getKey()).append(": ")
                        .append(entry.getValue()).append("\n");
            }
        } else {
            sb.append("headers: <empty>\n");
        }

        // WebView 不会把 Cookie 放进 getRequestHeaders()（它在此回调之后由内核附加），
        // 因此直接从 CookieManager 读取该 URL 对应的 Cookie 补充打印。
        String cookie = CookieManager.getInstance().getCookie(request.getUrl().toString());
        sb.append("cookie: ").append(cookie == null ? "<none>" : cookie).append("\n");

        LogUtils.printLog(sb.toString());
    }

    private void startExternalActivity(Intent intent) {
        try {
            startActivity(intent);
        } catch (Exception e) {
            Toast.makeText(this, "Unable to open app", Toast.LENGTH_SHORT).show();
        }
    }

    private void setupBackPressLogic() {
        getOnBackPressedDispatcher().addCallback(this, new OnBackPressedCallback(true) {
            @Override
            public void handleOnBackPressed() {
                if (myWebView.canGoBack()) {
                    myWebView.goBack();
                } else {
                    finish();
                }
            }
        });
    }

    @Override
    protected void onDestroy() {
        autoCloseHandler.removeCallbacks(autoCloseRunnable);
        INSTANCES.remove(this);
        if (myWebView != null) {
            myWebView.destroy();
            myWebView = null;
        }
        super.onDestroy();
    }

    /**
     * 启动一个新的 WebView 页面。
     * 通过时间去抖动防止按钮连点导致重复打开：
     * startActivity 是异步的，短时间内的第二次调用此时实例可能还没进入 INSTANCES，
     * 因此这里用一个同步的时间窗口来拦截重复触发。
     */
    public static synchronized void open(Context context, String url) {
        long now = SystemClock.elapsedRealtime();
        if (now - sLastOpenTime < OPEN_DEBOUNCE_MS) {
            return; // 去抖动：忽略过快的重复调用
        }
        sLastOpenTime = now;

        Intent intent = new Intent(context, WebViewActivity.class);
        intent.putExtra(EXTRA_URL, url);
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        context.startActivity(intent);
    }

    /**
     * 静态方法：关闭所有已由 JS 打开的 WebView 页面。
     */
    public static void closeAll() {
        WebViewActivity[] snapshot;
        synchronized (INSTANCES) {
            snapshot = INSTANCES.toArray(new WebViewActivity[0]);
        }
        for (WebViewActivity activity : snapshot) {
            if (activity != null && !activity.isFinishing()) {
                activity.runOnUiThread(activity::finish);
            }
        }
    }
}

```


### h5 中打开webview, 获取 UA和TOKEN
```javascript
async function handlePhonePeBusiness() {
  const bridge = (window as any).AndroidBridge
  // No native bridge: this flow is only available inside the App
  if (!bridge?.openWebView) {
    showToast({ message: 'This operation is only available in the App', duration: 3000 })
    return
  }

  stopTokenPoll()
  bridge.setToken?.('')
  bridge.openWebView('https://business.phonepe.com/login')

  showLoadingToast({ message: 'Waiting for login...', forbidClick: true, duration: 0 })

  const POLL_INTERVAL = 1000
  const TIMEOUT = 120000
  const startedAt = Date.now()

  tokenPollTimer = setInterval(async () => {
    if (Date.now() - startedAt >= TIMEOUT) {
      stopTokenPoll()
      closeToast()
      showToast({ message: 'Login timed out. Please try again.', duration: 3000 })
      return
    }

    const token = bridge.getToken?.() || ''
    if (!token) return
    console.log(token)

    // Got a token: stop polling and proceed
    stopTokenPoll()

    if (!token.startsWith('P1_')) {
      closeToast()
      showToast({ message: 'Send OTP failed', duration: 3000 })
      return
    }
    const ua = bridge.getUA?.() || ''
    console.log('UA:' + ua + ' token:' + token)
    closeToast()
  }, POLL_INTERVAL)
}
```
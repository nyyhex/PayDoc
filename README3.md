# 全局公共参数

**全局Header参数**

| 参数名 | 示例值 | 参数类型 | 是否必填 | 参数描述 |
| --- | --- | ---- | ---- | ---- |
| X-API-KEY | bbbb | string | 是 | APIKEY |
| X-SIGN | aaaa | string | 是 | 签名 |


# 浏览器相关接口


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


---

## 一、浏览器管理

### 1.1 创建浏览器

- **端点**：`POST /api/open/browser/create`
- **说明**：创建一个浏览器实例并分配给当前用户

**请求体**（可选）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `type` | String | 浏览器类型，如 `yuntu` / `browserless` |
| `proxyMode` | String | 代理模式：`system`（系统代理）/ `custom`（自定义代理） |
| `proxyUrl` | String | 代理地址，如 `socks5://user:pass@host:port`（proxyMode=custom 时使用） |
| `env` | List\<String\> | 环境变量列表，支持：`SYSTEM`（Windows/Mac/Linux/Android/iOS）、`WINDOW_SIZE`、`EXTENSION`、`DISABLE_WEBRTC`（True/False）、`COOKIES`、`TZ`（时区）、`COUNTRY_CODE`（国家代码） |
| `desc` | String | 浏览器实例描述 |
| `enableCdp` | Boolean | 是否开启 CDP 代理。为 `true` 时，系统根据浏览器所在服务器的网域（`server.gateway`）自动匹配网关并创建 APISIX 路由 |

**返回 `data`**：`BrowserInstance` 对象

| 字段 | 类型 | 说明 |
|---|---|---|
| `_id` | String | 浏览器实例 ID |
| `serverId` | String | 所属服务器 ID |
| `serverName` | String | 服务器名称 |
| `dockerId` | String | Docker 容器 ID |
| `deviceId` | String | 设备 ID |
| `ipAddress` | String | 局域网 IP |
| `port` | String | CDP 端口 |
| `name` | String | 容器名称 |
| `useUserID` | String | 使用者用户 ID |
| `type` | String | 类型：`yuntu` / `browserless` |
| `image` | String | 镜像名称 |
| `status` | String | 状态：`idle` / `used` / `stop` |
| `status2` | String | 是否在线 |
| `jumpServerId` | String | 跳板服务器 ID |
| `vncPort` | String | VNC 端口 |
| `proxyUrl` | String | 代理地址 |
| `proxyMode` | String | 代理模式：`system`（系统代理）/ `custom`（自定义代理） |
| `cdpProxyId` | String | CDP 代理记录 ID（开启 CDP 时返回） |
| `cdpUrl` | String | CDP 代理访问地址，如 `http://gateway:9080/{instanceId}`（开启 CDP 时返回） |
| `authorizedUserIDs` | List\<String\> | 被授权访问的用户 ID 列表 |
| `createTime` | String | 创建时间（ISO 8601） |
| `updateTime` | String | 更新时间 |
| `usedTime` | String | 开始使用时间 |
| `stopTime` | String | 停止时间 |

---

### 1.2 更新浏览器

- **端点**：`POST /api/open/browser/update`
- **说明**：更新已有的浏览器实例配置

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | String | 是 | 浏览器实例 ID |
| `type` | String | 否 | 浏览器类型，如 `yuntu` / `browserless` |
| `proxyUrl` | String | 否 | 代理地址，如 `socks5://user:pass@host:port` |
| `env` | List\<String\> | 否 | 环境变量列表 |
| `desc` | String | 否 | 浏览器实例描述 |

**返回 `data`**：`BrowserInstance` 对象（字段同 1.1）

---

### 1.3 浏览器列表

- **端点**：`POST /api/open/browser/list`
- **说明**：查询当前用户的浏览器实例列表

**请求体**（可选）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `filters` | Object | 过滤条件（key-value，会追加 `useUserID` 限制） |
| `sorter` | Object | 排序，如 `{"createTime": -1}` |
| `pageSize` | Integer | 每页条数（默认 20） |
| `page` | Integer | 页码（默认 1） |

**返回 `data`**：分页结构，`data` 数组为 `BrowserInstance` 对象列表（字段同 1.1）

---

### 1.4 释放浏览器

- **端点**：`POST /api/open/browser/release/{id}`
- **说明**：释放指定浏览器实例，恢复为 `idle` 状态

**路径参数**：

| 参数 | 类型 | 说明 |
|---|---|---|
| `id` | String | 浏览器实例 ID |

**请求体**：无

**返回 `data`**：`null`

---

### 1.5 暂停浏览器

- **端点**：`POST /api/open/browser/pause/{id}`
- **说明**：暂停指定浏览器实例

**路径参数**：

| 参数 | 类型 | 说明 |
|---|---|---|
| `id` | String | 浏览器实例 ID |

**请求体**：无

**返回 `data`**：`null`

---

### 1.6 恢复浏览器

- **端点**：`POST /api/open/browser/resume/{id}`
- **说明**：恢复已暂停的浏览器实例

**路径参数**：

| 参数 | 类型 | 说明 |
|---|---|---|
| `id` | String | 浏览器实例 ID |

**请求体**：无

**返回 `data`**：`null`

---

### 1.7 获取浏览器打开 URL

- **端点**：`POST /api/open/getOpenBrowserUrl`
- **说明**：获取指定浏览器的可远程打开 URL

**请求体**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `userID` | String | 目标用户 ID |
| `browserId` | String | 浏览器实例 ID |

**返回 `data`**：`String`，浏览器远程打开 URL

---

### 1.8 浏览器新增CDP代理

- **端点**：`POST /api/open/browser/cdp/create`
- **说明**：为指定浏览器实例创建 CDP 代理路由。网关根据浏览器所在服务器的网域（`server.gateway`）与网关名称（`apisixGateway.name`）自动匹配，无需手动指定。

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | String | 是 | 浏览器实例 ID |

**返回 `data`**：`BrowserInstance` 对象（含 `cdpProxyId` 和 `cdpUrl` 字段）

| 字段 | 类型 | 说明 |
|---|---|---|
| `cdpProxyId` | String | 创建的 CDP 代理记录 ID |
| `cdpUrl` | String | CDP 代理访问地址，如 `http://gateway:9080/{instanceId}` |

**错误情况**：
- 浏览器实例不存在
- 该实例已开启CDP代理
- 未找到匹配的网关（服务器网域未配置或无对应网关）

---

### 1.9 浏览器删除CDP代理

- **端点**：`POST /api/open/browser/cdp/delete`
- **说明**：删除指定浏览器实例的 CDP 代理路由，同时从 APISIX 网关移除对应路由。

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | String | 是 | 浏览器实例 ID |

**返回 `data`**：`null`

**错误情况**：
- 浏览器实例不存在
- 该实例未开启CDP代理

---

### 1.10 重新创建浏览器

- **端点**：`POST /api/open/recreate/{id}`
- **说明**：重新创建指定浏览器实例的 Docker 容器。删除旧容器后使用相同配置（代理、环境变量等）创建并启动新容器，仅对 `yuntu` 类型且配置了代理或环境变量的实例生效。

**路径参数**：

| 参数 | 类型 | 说明 |
|---|---|---|
| `id` | String | 浏览器实例 ID |

**请求体**：无

**返回 `data`**：`null`

**错误情况**：
- 浏览器实例不存在或无权操作
- 容器创建失败（镜像拉取失败、Docker API 异常等）

---

### 1.11 转移浏览器

- **端点**：`POST /api/open/browser/transfer`
- **说明**：将指定浏览器实例的使用权转移给目标用户（更换 `useUserID`）

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | String | 是 | 浏览器实例 ID |
| `targetUserID` | String | 是 | 目标用户 ID |

**返回 `data`**：`null`

**错误情况**：
- 浏览器实例不存在或无权操作

---

### 1.12 授权用户访问浏览器

- **端点**：`POST /api/open/browser/authorize`
- **说明**：将指定浏览器实例授权给目标用户，被授权用户可打开该浏览器页面

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | String | 是 | 浏览器实例 ID |
| `targetUserID` | String | 是 | 要授权的目标用户 ID |

**返回 `data`**：`null`

**错误情况**：
- 浏览器实例不存在或无权操作

---

### 1.13 取消用户授权

- **端点**：`POST /api/open/browser/revoke`
- **说明**：取消指定用户对浏览器实例的访问授权

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | String | 是 | 浏览器实例 ID |
| `targetUserID` | String | 是 | 要取消授权的目标用户 ID |

**返回 `data`**：`null`

**错误情况**：
- 浏览器实例不存在或无权操作

---

### 1.14 被授权浏览器列表

- **端点**：`POST /api/open/browser/authorized/list`
- **说明**：查询当前用户被授权访问的浏览器实例列表

**请求体**（可选）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `filters` | Object | 过滤条件 |
| `sorter` | Object | 排序 |
| `pageSize` | Integer | 每页条数（默认 10） |
| `page` | Integer | 页码（默认 1） |

**返回 `data`**：分页结构，`data` 数组为 `BrowserInstance` 对象列表（字段同 1.1）

---

## 二、代理管理

### 2.1 创建代理

- **端点**：`POST /api/open/proxy/create`
- **说明**：创建一个代理实例

**请求体**（可选）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `browserId` | String | 关联的浏览器实例 ID |
| `proxyUrl` | String | 代理地址，如 `socks5://user:pass@host:port` |
| `imageType` | String | 镜像类型 |

**返回 `data`**：`ProxyInstance` 对象

| 字段 | 类型 | 说明 |
|---|---|---|
| `_id` | String | 代理实例 ID |
| `serverId` | String | 所属服务器 ID |
| `serverName` | String | 服务器名称 |
| `dockerId` | String | Docker 容器 ID |
| `port` | String | 代理端口 |
| `name` | String | 容器名称 |
| `useUserID` | String | 使用者用户 ID |
| `type` | String | 类型，如 `gost` |
| `image` | String | 镜像名称 |
| `status` | String | 状态：`idle` / `used` / `stop` |
| `browserId` | String | 关联浏览器实例 ID |
| `browserName` | String | 关联浏览器名称 |
| `proxyUrl` | String | 完整代理地址 |
| `createTime` | String | 创建时间 |
| `updateTime` | String | 更新时间 |
| `stopTime` | String | 停止时间 |

---

### 2.2 代理列表

- **端点**：`POST /api/open/proxy/list`
- **说明**：查询当前用户的代理实例列表

**请求体**（可选）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `filters` | Object | 过滤条件（会追加 `useUserID` 限制） |
| `sorter` | Object | 排序 |
| `pageSize` | Integer | 每页条数（默认 20） |
| `page` | Integer | 页码（默认 1） |

**返回 `data`**：分页结构，`data` 数组为 `ProxyInstance` 对象列表（字段同 2.1）

---

### 2.3 释放代理

- **端点**：`POST /api/open/proxy/release/{id}`
- **说明**：释放指定代理实例

**路径参数**：

| 参数 | 类型 | 说明 |
|---|---|---|
| `id` | String | 代理实例 ID |

**请求体**：无

**返回 `data`**：`null`

---

## 三、用户管理

### 3.1 创建子用户

- **端点**：`POST /api/open/createUser`
- **说明**：在当前用户下创建一个子用户

**请求体**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `userID` | String | 子用户 ID |

**返回 `data`**：`null`

---

### 3.2 登出用户

- **端点**：`POST /api/open/logout`
- **说明**：登出指定用户并释放其浏览器资源

**请求体**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `userID` | String | 要登出的用户 ID |

**返回 `data`**：`null`

---

### 3.3 创建 API Key

- **端点**：`POST /api/open/createApiKey`
- **说明**：为当前用户创建一个新的 API Key（含 Key 和 Secret）

**请求体**：无

**返回 `data`**：`ApiKey` 对象

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | Long | API Key 记录 ID |
| `userID` | String | 所属用户 ID |
| `apiKey` | String | API Key（24字节 Base64） |
| `apiSecret` | String | API Secret（48字节 Base64） |
| `whiteIp` | String | IP 白名单（逗号分隔） |
| `status` | Integer | 状态：1=启用 |
| `createdTime` | String | 创建时间 |

---

## 四、数据仓库

### 4.1 查询数据仓库列表

- **端点**：`POST /api/open/dataRepo/list`
- **说明**：查询当前用户的数据仓库列表（admin 可查全部）

**请求体**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `filters` | Object | 过滤条件 |
| `sorter` | Object | 排序 |
| `pageSize` | Integer | 每页条数 |
| `page` | Integer | 页码 |

**返回 `data`**：分页结构，`data` 数组为 `DataRepo` 对象

| 字段 | 类型 | 说明 |
|---|---|---|
| `_id` | String | 仓库 ID |
| `userID` | String | 所属用户 ID |
| `name` | String | 仓库名称 |
| `dataSize` | Long | 数据总大小（字节） |
| `textNum` | Long | 文本条数 |
| `fileNum` | Long | 文件条数 |
| `imageNum` | Long | 图片条数 |
| `tableNum` | Long | 表格条数 |
| `createTime` | String | 创建时间 |
| `updateTime` | String | 更新时间 |

---

### 4.2 查询仓库数据列表（text / file / image）

- **端点**：`POST /api/open/dataRepo/{id}/{type}`
- **说明**：查询指定仓库中指定类型的数据列表

**路径参数**：

| 参数 | 类型 | 说明 |
|---|---|---|
| `id` | String | 数据仓库 ID |
| `type` | String | 数据类型：`text` / `file` / `image` / `table` |

**请求体**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `filters` | Object | 追加过滤条件 |
| `pageSize` | Integer | 每页条数 |
| `page` | Integer | 页码 |

**返回 `data`**：分页结构，`data` 数组为 `SubTaskData` 对象

| 字段 | 类型 | 说明 |
|---|---|---|
| `_id` | String | 数据 ID |
| `userID` | String | 所属用户 ID |
| `taskId` | String | 来源任务 ID |
| `taskExecuteNo` | String | 任务执行编号 |
| `repoId` | String | 所属仓库 ID |
| `repoName` | String | 仓库名称 |
| `name` | String | 数据名称 |
| `dataType` | String | 数据类型：`text`/`file`/`image`/`table` |
| `textContent` | String | 文本内容（type=text 时有值） |
| `filePath` | String | 文件路径（type=file 时有值） |
| `imgPath` | String | 图片路径（type=image 时有值） |
| `rowsHead` | String | 表头（CSV 格式，type=table 时有值） |
| `dataSize` | Long | 数据大小（字节） |
| `tableRowsNum` | Long | 表格行数（type=table 时有值） |
| `createTime` | String | 创建时间 |

---

### 4.3 查询表格明细数据

- **端点**：`POST /api/open/dataRepo/table/{id}`
- **说明**：查询指定 `SubTaskData`（type=table）的行级明细数据

**路径参数**：

| 参数 | 类型 | 说明 |
|---|---|---|
| `id` | String | SubTaskData 的 `_id` |

**请求体**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `filters` | Object | 追加过滤条件 |
| `pageSize` | Integer | 每页条数 |
| `page` | Integer | 页码 |

**返回 `data`**：分页结构，`data` 数组为 `SubTaskDataTable` 对象

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | String | 行数据 ID |
| `userID` | String | 所属用户 ID |
| `taskId` | String | 来源任务 ID |
| `taskExecuteNo` | String | 任务执行编号 |
| `dataId` | String | 关联的 SubTaskData ID |
| `rows` | String | 行数据内容（JSON 或 CSV） |
| `dataSize` | Long | 数据大小（字节） |
| `createTime` | String | 创建时间 |

---

### 4.4 新增数据到仓库

- **端点**：`POST /api/open/dataRepo/save`
- **说明**：向指定数据仓库写入一条数据

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `repoName` | String | 是 | 目标仓库名称 |
| `dataType` | String | 是 | 数据类型：`text` / `file` / `image` / `table` |
| `name` | String | 否 | 数据名称 |
| `taskId` | String | 否 | 关联任务 ID |
| `content` | String | type=text | 文本内容 |
| `base64` | String | type=file/image | 文件/图片 Base64 编码内容 |
| `filepath` | String | type=file/image | 文件路径（与 base64 二选一） |
| `rowsHead` | String | type=table | 表头（逗号分隔，如 `列1,列2,列3`） |
| `rows` | List\<String\> | type=table | 每行数据（逗号分隔，与 rowsHead 列数对应） |

**返回 `data`**：`null`

---

## 五、任务管理

### 5.1 新增/更新任务

- **端点**：`POST /api/open/subTask/save`
- **说明**：创建或更新一个任务（有 `_id` 则更新）

**请求体**（`SubTask` 对象）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `_id` | String | 任务 ID（更新时传入） |
| `name` | String | 任务名称 |
| `dockerId` | String | 任务镜像 |
| `type` | String | 任务类型 |
| `browserIds` | List\<String\> | 指定浏览器 ID 列表 |
| `browserType` | String | 浏览器类型（非指定时使用） |
| `taskDockerId` | String | 任务镜像 ID |
| `status` | String | 状态：`待执行`/`执行中`/`执行完成` |
| `executeType` | String | 执行方式：`执行1次`/`循环执行` |
| `executeTime` | String | 执行时间（ISO 8601） |
| `loopExecuteTime` | String | 循环执行规则（Cron 表达式） |
| `params` | Object | 执行参数（任意 key-value） |
| `timeout` | Long | 超时时间（秒） |
| `relationDataRepoId` | List\<String\> | 关联的数据仓库 ID 列表 |

**返回 `data`**：`SubTaskSaveVO`

| 字段 | 类型 | 说明 |
|---|---|---|
| `_id` | String | 保存后的任务 ID |

---

### 5.2 暂停任务

- **端点**：`POST /api/open/subTask/pause`
- **说明**：将状态为 `待执行` 的任务批量暂停（改为 `暂停`）

**请求体**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `ids` | List\<String\> | 任务 ID 列表 |

**返回 `data`**：`null`

---

### 5.3 重启任务

- **端点**：`POST /api/open/subTask/restart`
- **说明**：将状态为 `暂停` 的任务批量恢复为 `待执行`

**请求体**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `ids` | List\<String\> | 任务 ID 列表 |

**返回 `data`**：`null`

---

### 5.4 结束任务

- **端点**：`POST /api/open/subTask/finish`
- **说明**：批量强制结束任务

**请求体**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `ids` | List\<String\> | 任务 ID 列表 |

**返回 `data`**：`null`

---

### 5.5 查询任务列表

- **端点**：`POST /api/open/subTask/list`
- **说明**：查询任务列表（admin 可查全部，普通用户只查自己的）

**请求体**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `filters` | Object | 过滤条件，支持按 `name`/`status`/`executeType` 等字段过滤 |
| `sorter` | Object | 排序 |
| `pageSize` | Integer | 每页条数 |
| `page` | Integer | 页码 |

**返回 `data`**：分页结构，`data` 数组为 `SubTask` 对象（字段同 5.1 请求体，另含以下只读字段）

| 字段 | 类型 | 说明 |
|---|---|---|
| `hasExecuteNum` | Long | 已执行次数 |
| `finishTime` | String | 任务结束时间 |
| `containerId` | String | 运行容器 ID |
| `deviceId` | String | 设备 ID |
| `createTime` | String | 创建时间 |

---

## 六、任务日志

### 6.1 查询日志列表

- **端点**：`POST /api/open/subTaskLog/list`
- **说明**：查询任务日志列表（admin 可查全部）

**请求体**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `filters` | Object | 过滤条件，支持 `taskId`/`taskExecuteNo`/`logLevel`/`logTag` |
| `sorter` | Object | 排序 |
| `pageSize` | Integer | 每页条数 |
| `page` | Integer | 页码 |

**返回 `data`**：分页结构，`data` 数组为 `SubTaskLog` 对象

| 字段 | 类型 | 说明 |
|---|---|---|
| `_id` | String | 日志 ID |
| `userID` | String | 所属用户 ID |
| `taskId` | String | 任务 ID |
| `taskExecuteNo` | String | 任务执行编号 |
| `logLevel` | String | 日志级别：`DEBUG`/`INFO`/`WARN`/`ERROR` |
| `logContent` | String | 日志内容 |
| `logTag` | String | 日志标签（如 `browser_init`/`data_processing`） |
| `createTime` | String | 创建时间 |

---

### 6.2 新增日志

- **端点**：`POST /api/open/subTaskLog/save`
- **说明**：为指定任务写入一条日志（任务必须归属当前用户）

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `taskId` | String | 是 | 关联任务 ID |
| `taskExecuteNo` | String | 否 | 任务执行编号 |
| `logLevel` | String | 否 | 日志级别：`DEBUG`/`INFO`/`WARN`/`ERROR` |
| `logTag` | String | 否 | 日志标签 |
| `logContent` | String | 否 | 日志内容 |

**返回 `data`**：`null`

---

## 七、镜像管理

### 7.1 保存镜像（新增/更新）

- **端点**：`POST /api/open/dockerImage/save`
- **说明**：创建或更新一个 Docker 镜像配置（有 `_id` 则更新）。新建时会对 `imageType` 做唯一性校验（`taskjs`/`taskpython` 类型除外，每种类型每个用户只能绑定一个镜像）。更新时 `imageType` 不可通过此接口修改。系统会根据 `registryURL`/`repositoryName`/`tag` 自动生成 `imgId`。

**请求体**（`DockerImage` 对象）：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `_id` | String | 否 | 镜像 ID（更新时传入） |
| `name` | String | 否 | 镜像名称 |
| `registryURL` | String | 否 | 镜像仓库地址，如 `registry.example.com` |
| `repositoryName` | String | 否 | 仓库名称，如 `myapp/browser` |
| `tag` | String | 否 | 镜像标签（默认 `latest`） |
| `imageType` | String | 否 | 镜像类型：`yuntu`/`browserless`/`gost`/`taskjs`/`taskpython`（仅新建时有效） |
| `description` | String | 否 | 镜像描述 |

**返回 `data`**：`DockerImageSaveVO`

| 字段 | 类型 | 说明 |
|---|---|---|
| `_id` | String | 保存后的镜像 ID |

---

### 7.2 查询单个镜像

- **端点**：`GET /api/open/dockerImage/{_id}`
- **说明**：根据 ID 查询镜像详情

**路径参数**：

| 参数 | 类型 | 说明 |
|---|---|---|
| `_id` | String | 镜像 ID |

**返回 `data`**：`DockerImage` 对象

| 字段 | 类型 | 说明 |
|---|---|---|
| `_id` | String | 镜像 ID |
| `imgId` | String | 镜像唯一标识（自动生成，格式 `registryURL/repositoryName:tag`） |
| `name` | String | 镜像名称 |
| `registryURL` | String | 镜像仓库地址 |
| `repositoryName` | String | 仓库名称 |
| `tag` | String | 镜像标签 |
| `imageType` | String | 镜像类型 |
| `description` | String | 镜像描述 |
| `userID` | String | 所属用户 ID |
| `createdTime` | String | 创建时间（ISO 8601） |
| `updatedTime` | String | 更新时间 |

---

### 7.3 镜像列表

- **端点**：`POST /api/open/dockerImage/list`
- **说明**：查询镜像列表（admin 可查全部，普通用户只查自己的）

**请求体**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `filters` | Object | 过滤条件，支持 `name`/`imageType`/`registryURL` 等字段 |
| `sorter` | Object | 排序 |
| `pageSize` | Integer | 每页条数（默认 20） |
| `page` | Integer | 页码（默认 1） |

**返回 `data`**：分页结构，`data` 数组为 `DockerImage` 对象（字段同 7.2）

---

### 7.4 删除镜像

- **端点**：`POST /api/open/dockerImage/delete`
- **说明**：批量删除镜像

**请求体**：`List<String>`，镜像 ID 列表

```json
["id1", "id2", "id3"]
```

**返回 `data`**：`null`

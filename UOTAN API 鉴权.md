
# UOTAN API 鉴权

## 概念说明

### Token Keystore

> 用于验证 Token 是否有效的随机字符串，存储在服务端数据库

|属性|说明|
|---|---|
|类型|String|
|长度|64位随机字符串|
|存储|服务端数据库, 客户端不可见|

**更新时机：**

- 用户首次设置密码时

- 用户修改密码时

- 客户端使用 Refresh Token 换取新 Access Token 时

> 服务端会存储 Access Token 和 Refresh Token 的 Keystore

使用 Refresh Token 换取新 Access Token 时：

- 服务端更新 Keystore

- 同时签发新的 Access Token 和 Refresh Token (并不一致)

- 旧 Access Token 和 Refresh Token 立即失效

Refresh Token 仅用于换取 Access Token, 过期后必须重新进行完整鉴权

### Access Token

> 短期有效的请求凭证，每次调用 API 时携带

|属性|说明|
|:---:|:---:|
|有效期|默认 2 小时|
|存储位置|非持久化存储|
|携带方式|`Authorization: Bearer <token>`|
|到期操作|客户端必须使用 Refresh Token 换取新的 Access Token|

**Payload 结构：**

|键|值类型|说明|
|:---:|:---:|:---:|
|user_id|Int|用户唯一标识|
|keystore|String (Token Keystore)|与服务端比对, 验证有效性|
|expires|Long|过期时间戳|


### Refresh Token

> 长期有效的刷新凭证，用于在 Access Token 过期后静默换取新的 Access Token

|属性|说明|
|:---:|:---:|
|有效期|30 天|
|存储位置|客户端加密持久化）|
|携带方式|请求 Body|
|一次性|每次使用后立即作废, 同时签发新的 Refresh Token|

**Payload 结构：**

|键|值类型|说明|
|:---:|:---:|:---:|
|user_id|Int|用户唯一标识|
|keystore|String|与服务端比对，验证有效性|
|expires|Long|过期时间戳|

## 服务端存储

```sql
CREATE TABLE xf_app_tokens (
    user_id        INT PRIMARY KEY,
    access_key     VARCHAR(64),
    refresh_key    VARCHAR(64),
    updated_at     INT
)
```

## 鉴权流程

### 完整鉴权

**POST** -- /api/app/auth/

**Body 结构：**

|键|值类型|说明|
|:---:|:---:|:---:|
|account|String|用户名|
|password|String|密码|

**Return** -- Access Token + Refresh Token

### 正常请求 API

![正常请求](\src\鉴权\正常请求.png)

### 刷新 Token

**POST** -- /api/app/auth/refresh

**Body 结构：**

|键|值类型|说明|
|:---:|:---:|:---:|
|refresh_token|String|Refresh Token|

![刷新 Token](<\src\鉴权\刷新 Token.png>)

## 强制登出场景

|场景|处理方式|
|:---:|:---:|
|用户修改密码|Keystore 全部更新, 所有 Token 立即失效|
|Refresh Token 过期|强制重新登录|

## 安全说明

|威胁|对策|
|:---:|:---:|
|Access Token 泄露|短时间自动失效|
|Refresh Token 泄露|依赖传输和存储层保证|
|用户改密码|Keystore 全部更新, 全端踢出|
|长期不使用|Refresh Token 过期, 重新登录|
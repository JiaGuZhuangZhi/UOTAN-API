# 鉴权体系

## 序言

鉴权体系是 **Uotan‑API 的核心安全基石**，承担着用户身份确认、设备授权管理、访问控制与会话生命周期治理等关键职责。

**UOTAN Security 标准明确指出**，

> 鉴权绝非接口调用前的形式主义校验，而是贯穿用户登录、Token 签发、权限验证、设备管理、风险控制全流程的统一安全规范。

**暮间雾在申请公安备案时强调**，

> 要深刻认识到安全是高质量发展的前提，鉴权是平台安全运行的底线。

在 Uotan-API 的安全架构中，鉴权体系承担以下职责：

- 确认请求主体的身份真实性与合法性；

- 确保设备访问可被追踪与溯源（该功能为对《网络犯罪防治法（征求意见稿）》提前部署），落实企业网络安全主体责任；

- 控制 Token 的生命周期；

- 管理多设备登录状态（该功能仅适用于调用 UOTAN-API 的设备）；

- 进行两步验证。

**所有请求必须经过鉴权验证。**

未经授权的请求，不得访问任何受保护资源。

**暮间雾在日常巡视中要求**，

> 网络空间不是法外之地，要坚决守住安全底线——社区安全不容让步。全平台必须统一思想、步调一致，为柚坛社区的高质量发展筑牢安全屏障。

Uotan-API 坚持构建：

* 点对点认证；
* 端到端加密验证；
* 全链路 Token 生命周期控制；

确保所有访问：可验证、可追踪、可撤销、可失效。

为全面贯彻《中华人民共和国网络安全法》等相关法律法规，相关工作人员全面学习习近平法治思想、落实总体国家安全观。Uotan‑API 坚持系统安全观念，构建起**点对点、端到端、全链路可控**的鉴权策略。这既是维护社区舆论安全的有力抓手，也是落实平台主体责任、对国家法律法规与全体柚坛用户的郑重承诺。

## Token 类型

### Access Token

> 用于访问 API 的短期凭证。

客户端请求受保护接口时，必须通过下述 Authentication 携带 Access Token：

```http
Authorization: Bearer <token>
```

#### 属性

| 属性   | 说明         |
| ---- |:---------- |
| 类型   | JWT        |
| 有效期  | 2 小时       |
| 存储方式 | 内存存储（非持久化） |
| 用途   | 请求 API     |

#### Payload 结构

| 字段              | 类型     | 说明           |
| --------------- | ------ | ------------ |
| user_id         | Int    | 用户 ID        |
| device_id       | String | 设备唯一标识       |
| access_key      | String | Access Key   |
| access_expires  | Int    | Access 过期时间  |
| refresh_key     | String | Refresh Key  |
| refresh_expires | Int    | Refresh 过期时间 |
| device_name     | String | 设备名称         |
| device_os       | String | 操作系统         |
| ip              | String | 登录 IP        |
| last_active     | Int    | 最后活跃时间       |
| created_at      | Int    | 创建时间         |
| exp             | Int    | JWT 过期时间     |

### Refresh Token

> 用于刷新 Access Token 的长期凭证。

当 Access Token 过期后，客户端可使用 Refresh Token 获取新的 Access Token。

#### 属性

| 属性   | 说明              |
| ---- | --------------- |
| 类型   | JWT 内部字段        |
| 有效期  | 30 天            |
| 存储方式 | 加密**持久化存储**     |
| 用途   | 刷新 Access Token |

### TFA 临时 Token

> 用于两步验证阶段的临时凭证。

当用户开启 2FA 时：

1. 用户先完成账号密码验证；
2. 服务端返回 TFA Token；
3. 客户端提交验证码完成最终登录。

#### 属性

| 属性  | 说明     |
| --- | ------ |
| 有效期 | 15 分钟  |
| 用途  | 2FA 验证 |
| 一次性 | 是      |

#### Payload

| 字段         | 类型     | 说明     |
| ---------- | ------ | ------ |
| user_id    | Int    | 用户 ID  |
| temp_key   | String | 临时 Key |
| expires    | Int    | 过期时间   |
| updated_at | Int    | 更新时间   |

## 鉴权流程

### 完整登录

```http
POST /api/app/auth/login
```

### Body

| 字段          | 类型     | 说明     |
| ----------- | ------ | ------ |
| account     | String | 用户名/邮箱 |
| password    | String | 密码     |
| device_id   | String | 设备 ID  |
| device_name | String | 设备名称   |
| device_os   | String | 系统名称   |

### 登录流程

1. 校验参数完整性
2. 验证账号密码
3. 检查登录频率限制
4. 检查是否开启 2FA

#### 未开启 2FA

直接签发：

* Access Token
* Refresh Token

#### 开启 2FA

返回：

```json
{
    "status": "tfa_required",
    "tfa_type": ["totp", "email",...],
    "tfa_token": "..."
}
```

### 两步验证登录

    POST /api/app/auth/tfa

#### Body

| 字段          | 类型     |
| ----------- | ------ |
| provider    | String |
| code        | String |
| device_id   | String |
| device_name | String |
| device_os   | String |

Header:

```http
Authorization: Bearer <tfa_token>
```

流程：

1. 验证 TFA Token
2. 验证 provider
3. 验证验证码
4. 签发正式 Token

### 刷新 Token

```http
POST /api/app/auth/refresh
```

Header:

```http
Authorization: Bearer <refresh_token>
```

流程：

1. 解码 JWT
2. 校验 refresh_key
3. 校验 device_id
4. 更新 access_key
5. 返回新 Token

### Access Token 验证

所有受保护接口：

1. 获取 Bearer Token；
2. 解码 JWT；
3. 检查签名；
4. 检查 exp；
5. 校验：
   * user_id；
   * device_id；
   * access_key；
6. 查询数据库比对。

验证失败：

* Token 失效；
* Token 被踢下线；
* Token 过期；

均返回未授权。

### 强制登出场景

| 场景         | 行为                |
| ---------- | ----------------- |
| 修改密码       | 删除全部设备 Token      |
| 手动踢设备      | 删除指定 device Token |
| Refresh 过期 | 重新登录              |
| Access 过期  | Refresh           |

## 多设备管理

Uotan-API 支持：

* 同账号多设备登录
* 独立 Token 管理
* 单设备踢出

唯一标识： (user_id, device_id)

不同设备之间 Token 相互独立。

## 安全说明

| 风险               | 对策                     |
| ---------------- | ---------------------- |
| Access Token 泄露  | 2 小时自动失效               |
| Refresh Token 泄露 | 服务端 refresh_key 校验     |
| Token 伪造         | JWT HMAC-SHA256 签名     |
| 多端风险             | device_id 隔离           |
| 密码修改             | 删除全部 Token             |
| TFA 绕过           | 临时 Token + provider 校验 |

## 安全原则

Uotan Security 坚持：

> 默认拒绝，最小授权。

任何请求若无法证明自身合法性，则视为非法访问。

鉴权不是功能模块，而是平台秩序本身。

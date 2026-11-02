# 后端工程化每日训练 Day 45：JWT 登录认证、Token 刷新与登录态安全设计复盘

## 一、今天学习什么

今天主要学习：

- Authentication（认证）与 Authorization（授权）的区别
- JWT 的 Payload、签名与篡改校验
- Access Token 为什么必须设置有效期
- Refresh Token 为什么存在，以及它和 Access Token 的职责区别
- 退出登录后旧 JWT 为什么不一定真正失效
- Token Version 如何让旧登录态批量失效
- 用户权限变化后，为什么不能完全相信 JWT 中的旧角色
- 资源级权限校验与接口越权
- JWT 黑名单与 Redis 带来的“半有状态”设计
- JWT、Refresh Token、Redis、Token Version、权限校验如何组合成完整登录系统

今天最核心的判断：

> JWT 真正的工程难点，不是“如何生成一个 Token”，而是如何管理一个身份凭证从签发、使用、刷新、权限变化到最终撤销的完整生命周期。

---

# 二、认证和授权不是一回事

首先需要区分两个概念。

认证：

```text
Authentication
=
确认你是谁
```

例如：

```text
用户名 + 密码
↓
后端验证
↓
确认当前用户是 user1001
```

授权：

```text
Authorization
=
确认你能做什么
```

例如：

```text
普通用户
→ 查看自己的订单

管理员
→ 查看全部用户
→ 删除用户
```

所以：

```text
已经登录
≠
拥有所有接口权限
```

登录只解决身份问题，后端仍然必须继续做权限判断。

---

# 三、JWT Payload 可以读取，但不能随便篡改

今天第一个问题是：

> JWT 的 Payload 能不能被客户端读取？

答案是：

```text
可以
```

因为普通 JWT 的 Payload 通常只是：

```text
Base64URL 编码
```

而不是：

```text
加密
```

例如 JWT 中可能有：

```json
{
  "userId": 1001,
  "role": "USER",
  "exp": 1780000000
}
```

客户端可以把 Payload 解码出来看到这些内容。

但如果客户端把：

```text
role = USER
```

修改成：

```text
role = ADMIN
```

原来的 Signature 就不再匹配。

后端验签：

```text
Header + Payload
↓
重新计算签名
↓
与 Token 中 Signature 比较
↓
不一致
↓
拒绝请求
```

因此 JWT 的重点不是：

```text
让别人看不到数据
```

而是：

```text
让别人修改数据以后能够被发现
```

所以不能把下面这些敏感信息直接塞进普通 JWT：

```text
密码
身份证号
银行卡号
其他敏感隐私数据
```

---

# 四、Access Token 为什么不能永久有效

如果 Access Token 永不过期：

```text
Token 被盗
↓
攻击者今天能用
↓
一个月后还能用
↓
一年后仍然能用
```

例如：

```text
电脑丢失
浏览器环境被窃取
Token 被恶意程序复制
```

攻击者拿到 Token 后，就可能长期冒充用户。

所以 Access Token 通常设计为：

```text
短生命周期
```

例如：

```text
15 分钟
30 分钟
1 小时
```

这样即使 Token 泄漏，也能够限制攻击窗口。

核心思想：

> Token 有效期本质上是在安全风险和用户体验之间做平衡。

---

# 五、Refresh Token 不是用来调用业务接口的

今天这里纠正了一个容易出现的误解。

假设：

```text
Access Token = 30 分钟
Refresh Token = 7 天
```

错误理解可能是：

```text
Access Token 过期
↓
以后 7 天直接拿 Refresh Token 调业务接口
```

这是错误的。

正确职责应该是：

```text
Access Token
→ 调用业务接口

Refresh Token
→ 只负责换新的 Access Token
```

正常流程：

```text
用户登录
↓
获得 Access Token + Refresh Token
↓
使用 Access Token 调用业务接口
```

30 分钟后：

```text
Access Token 过期
↓
客户端拿 Refresh Token
↓
POST /auth/refresh
↓
服务器验证 Refresh Token
↓
签发新的 Access Token
↓
继续使用新的 Access Token 调业务接口
```

所以所谓：

```text
7 天免登录
```

真正表示的是：

```text
7 天内不用重新输入账号密码
```

而不是：

```text
Refresh Token 直接拥有 7 天业务接口访问权限
```

Refresh Token 生命周期更长，因此一旦泄漏，风险甚至可能比短期 Access Token 更高，所以通常需要更严格的保护和撤销能力。

---

# 六、前端删除 JWT 不等于后端 Token 已经失效

用户点击：

```text
退出登录
```

前端可能只是：

```text
删除浏览器保存的 JWT
```

这只能说明：

```text
当前客户端以后不再主动携带这个 Token
```

不能说明：

```text
这个 Token 已经从整个系统彻底失效
```

例如攻击者提前复制了 JWT：

```text
用户退出登录
↓
浏览器删除 Token
↓
攻击者仍然保存着旧 Token
↓
Token 签名正确
↓
Token 仍未过期
↓
后端可能继续接受
```

因此真正安全要求较高的系统，需要服务端具备撤销能力。

常见方案包括：

```text
Redis Token 黑名单
Token Version
Refresh Token 撤销
按设备 / Session 管理登录态
```

今天形成的一个关键认识：

> “前端退出”是客户端行为，“凭证失效”是服务端安全问题，两者不能混为一谈。

---

# 七、修改密码后，不能只修改 password

假设账号已经被盗。

攻击者手里已经拿到了：

```text
Access Token
Refresh Token
```

用户发现以后修改密码。

如果系统只做：

```sql
UPDATE user
SET password = ?
WHERE id = ?;
```

那么已经签发出去的 Token 可能完全不受影响。

攻击者甚至还可能继续使用 Refresh Token 换取新的 Access Token。

因此更合理的设计是：

```text
修改 password
↓
Token Version + 1
↓
撤销旧 Refresh Token
↓
旧登录态全部失效
```

例如当前用户：

```text
tokenVersion = 8
```

旧 Token 中：

```text
version = 8
```

修改密码以后：

```text
tokenVersion
8 → 9
```

旧 Token 再请求：

```text
Token version = 8
当前 version = 9
↓
不一致
↓
拒绝
```

这样就可以一次性让该用户的一批旧登录态失效。

---

# 八、Token Version 解决的是“一批旧 Token 失效”

Token Version 可以简单理解为：

```text
用户登录凭证的版本号
```

例如用户表：

```text
user_id = 1001
token_version = 5
```

JWT 中：

```json
{
  "userId": 1001,
  "tokenVersion": 5
}
```

用户发生高风险操作：

```text
修改密码
账号封禁
踢出全部设备
```

服务端：

```text
tokenVersion
5 → 6
```

所有版本为 5 的旧 Token：

```text
全部失效
```

它适合解决：

```text
一次性让某个用户所有旧登录态失效
```

但代价是：

```text
后端需要知道当前 tokenVersion
```

所以通常仍然需要：

```text
Redis / DB
```

保存或查询最新状态。

---

# 九、JWT 中的旧角色不能永远相信

假设管理员在 10:00 登录。

JWT：

```text
role = ADMIN
```

10:05，数据库把这个用户降级：

```text
role = USER
```

但旧 JWT 还有 55 分钟才过期。

这时候可能出现：

```text
数据库真实权限 = USER
JWT 中旧权限     = ADMIN
```

如果后端之后完全相信 JWT：

```text
role = ADMIN
```

那么这个用户可能继续：

```text
访问管理员接口
执行管理员操作
```

直到 Token 自然过期。

所以高风险系统需要考虑：

```text
权限变化如何快速生效
```

方案可能包括：

```text
短生命周期 Access Token
Redis 权限缓存
权限版本号
关键接口实时权限校验
```

这里的重点不是“JWT 不能放 role”，而是：

> JWT 中的数据是一份签发时快照，业务状态发生变化后，这份快照可能已经过期。

---

# 十、必须登录仍然可能发生严重越权

接口：

```http
GET /orders/{orderId}
```

系统已经配置：

```text
必须登录
```

仍然不能认为接口就是安全的。

例如：

```text
用户 A 正常登录
自己的订单 ID = 10001
```

然后用户 A 手动访问：

```http
GET /orders/10002
```

而：

```text
订单 10002 属于用户 B
```

如果后端只执行：

```sql
SELECT *
FROM orders
WHERE id = ?;
```

那么用户 A 就可能看到用户 B 的订单。

正确逻辑至少应该把资源归属一起检查：

```sql
SELECT *
FROM orders
WHERE id = ?
AND user_id = ?;
```

因此：

```text
Token 合法
≠
有权访问这个资源
```

这里不是“伪造登录”，而是：

> 用户已经正常登录，但访问了本来不属于自己的资源，这属于登录后的越权访问。

---

# 十一、前端隐藏按钮不能代替后端权限校验

错误设计：

```text
普通用户看不到管理员删除按钮
↓
所以后端 DELETE 接口不用检查权限
```

这是错误的。

因为前端只是界面。

攻击者可以完全绕过页面，直接使用：

```text
Apifox
Postman
curl
浏览器开发者工具
```

直接构造请求：

```http
DELETE /api/users/1001
```

如果后端没有真正做角色 / 权限判断，那么普通用户仍然可能成功调用管理员接口。

正确流程应该是：

```text
请求到达后端
↓
识别当前用户
↓
读取 / 判断当前权限
↓
检查是否拥有接口所需权限
↓
有权限 → 执行
无权限 → 403
```

所以：

> 前端权限控制主要改善用户体验，后端权限控制才是真正的安全边界。

---

# 十二、权限校验不是“管理员”本身

今天还纠正了一个概念：

```text
权限校验
```

不是某一种角色。

权限校验是一段后端判断逻辑。

例如：

```text
当前用户角色：USER
请求：DELETE /users/1001
```

接口要求：

```text
USER_DELETE
```

后端判断：

```text
当前用户没有 USER_DELETE
↓
403 Forbidden
```

管理员只是通常拥有更多权限。

真正的逻辑是：

```text
用户
↓
角色 / 权限
↓
接口要求
↓
权限校验
↓
允许 / 拒绝
```

---

# 十三、JWT 黑名单为什么削弱“无状态”优势

纯 JWT 的一个主要特点是：

```text
服务端不需要保存每个用户的 Session
```

请求：

```text
携带 JWT
↓
验证签名
↓
检查 exp
↓
解析 userId
↓
放行
```

后端不需要额外查询：

```text
“这个 Token 当前有没有被注销？”
```

这就是 JWT 的无状态优势之一。

但是加入 Redis 黑名单以后：

```text
用户退出
↓
Token jti = abc123
↓
Redis：blacklist:abc123 = 1
↓
TTL = Token 剩余有效期
```

每次请求都需要：

```text
验证 JWT
↓
再查询 Redis
↓
检查是否在黑名单
↓
存在 → 拒绝
不存在 → 放行
```

于是系统又开始依赖服务端状态。

因此可以理解为：

```text
JWT 本身仍然无状态
+
Redis 保存撤销状态
=
整个认证系统变成半有状态
```

这种设计不是错误。

它本质上是在做取舍：

```text
纯 JWT
优点：简单、少一次远程查询、扩展方便
缺点：签发以后很难立即撤销

JWT + Redis 黑名单
优点：可以立即注销、立即踢人
缺点：增加 Redis 状态和查询成本
```

所以：

> 为了获得“立即撤销能力”，系统牺牲了一部分纯 JWT 的无状态优势。

---

# 十四、核心支付后台如何组合这些组件

今天最后用一个支付后台场景进行了组合设计。

需求：

```text
Access Token：30 分钟
7 天免登录
修改密码后全部设备退出
管理员可以强制踢人
权限修改快速生效
多实例 Java 服务
```

不能指望一个 JWT 解决所有问题。

应该让不同组件承担不同职责。

## 1. JWT Access Token：30 分钟访问凭证

```text
登录成功
↓
签发 JWT Access Token
↓
有效期 30 分钟
↓
正常业务接口携带 Access Token
```

主要解决：

```text
后端识别当前请求是谁发出的
```

---

## 2. Refresh Token：7 天免登录

```text
Refresh Token
有效期 7 天
```

Access Token 过期：

```text
Refresh Token
↓
/auth/refresh
↓
换新的 30 分钟 Access Token
```

主要解决：

```text
安全使用短 Access Token
+
又不要求用户每 30 分钟重新输入密码
```

---

## 3. Token Version：修改密码后全部设备退出

修改密码：

```text
password 更新
↓
tokenVersion + 1
↓
旧 Token 版本全部不匹配
↓
全部设备旧登录态失效
```

主要解决：

```text
一批旧 Token 快速统一失效
```

---

## 4. Redis：管理员踢人和共享动态状态

多实例 Java 服务：

```text
Java Instance A
Java Instance B
Java Instance C
```

如果管理员踢掉某个会话：

```text
Redis 记录该 Session / Token 已撤销
```

那么所有实例都可以读取同一份状态：

```text
A 能看到
B 能看到
C 能看到
```

主要解决：

```text
多实例之间共享需要快速变化的登录状态
```

---

## 5. Redis + 权限校验：权限修改快速生效

例如：

```text
原权限：ADMIN
新权限：USER
```

不能继续只相信 JWT 中：

```text
role = ADMIN
```

可以设计：

```text
JWT 获取 userId
↓
查询 Redis 中当前权限
↓
执行权限校验
↓
允许 / 拒绝
```

Redis：

```text
保存 / 缓存最新权限状态
```

权限校验：

```text
真正决定当前接口能不能执行
```

---

# 十五、完整登录态流程

可以把今天的知识组合成下面这张流程图：

```text
用户名 + 密码
↓
登录成功
↓
Access Token（JWT，30 分钟）
+
Refresh Token（7 天）
```

正常访问：

```text
Access Token
↓
业务接口
↓
认证
↓
权限校验
↓
执行请求
```

Access Token 过期：

```text
Refresh Token
↓
/auth/refresh
↓
新的 Access Token
```

修改密码：

```text
tokenVersion + 1
↓
全部旧登录态失效
```

管理员踢某台设备：

```text
Redis 撤销对应会话
↓
该设备立即失效
```

修改用户权限：

```text
更新真实权限
↓
Redis 更新 / 失效缓存
↓
后端重新权限校验
↓
新权限快速生效
```

---

# 十六、今天纠正的几个错误理解

## 1. Refresh Token 不是业务 Token

错误：

```text
Access Token 过期以后
直接拿 Refresh Token 调接口 7 天
```

正确：

```text
Refresh Token
只负责换新的 Access Token
```

---

## 2. 不能简单说“后端把旧 JWT 删掉”

纯无状态 JWT 场景中：

```text
服务端可能根本没有保存完整 JWT
```

所以真正应该描述为：

```text
让旧登录态失效
```

实现方式可能是：

```text
黑名单
Token Version
Refresh Token 撤销
Session 撤销
```

---

## 3. 订单接口问题不是“伪造登录”

用户已经正常登录。

真正的问题是：

```text
用户 A
访问用户 B 的资源
```

属于：

```text
资源级越权
```

---

## 4. 权限校验不是管理员角色

```text
管理员
```

是一种角色。

```text
权限校验
```

是后端根据当前身份、角色、权限和目标接口做出的判断过程。

---

# 十七、今天形成的工程认知

以前可能会把登录系统理解成：

```text
登录
↓
生成 JWT
↓
接口解析 JWT
```

今天理解进一步变成：

```text
登录只是入口
```

真正完整的认证系统还需要考虑：

```text
Token 多久过期
如何续签
如何退出
如何踢人
修改密码后旧 Token 怎么办
权限修改后旧 Token 怎么办
Token 泄漏怎么办
多设备如何管理
多实例如何共享登录状态
认证以后如何继续做资源级授权
```

JWT、Refresh Token、Redis、Token Version、权限校验并不是互相替代，而是在解决不同的问题：

```text
JWT
→ 身份凭证

Refresh Token
→ 短 Token 的续签

Redis
→ 动态状态、撤销状态、多实例共享

Token Version
→ 批量让旧登录态失效

权限校验
→ 决定当前用户是否真的有权执行操作
```

最后的核心思想：

> 一个可靠的登录系统，不应该只关注 Token 能不能生成和解析，而应该关注身份凭证在整个生命周期内能不能被安全地使用、刷新、限制和撤销。

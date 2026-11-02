# 后端工程化每日训练 Day 26：对象存储、文件一致性与云厂商迁移设计

## 一、今天学习的知识点

今天学习的是：

**对象存储（OSS / OBS / S3）、文件上传架构、文件元数据管理、数据一致性补偿，以及云厂商迁移设计。**

一句话理解：

> 图片、PDF、视频、Excel、AI 生成文件等大对象保存在对象存储中，数据库只保存文件元数据和 Object Key；系统还必须处理上传、入库、删除、权限和迁移过程中可能出现的数据不一致问题。

今天重点解决以下问题：

```text
为什么文件不适合长期保存在应用服务器本地
为什么数据库通常不直接保存文件 BLOB
数据库有文件记录，但下载返回 404，可能是什么原因
OBS 上传成功、数据库写入失败，应该如何补偿
为什么数据库建议保存 Object Key，而不是完整 URL
系统迁移到另一家云厂商时，如何减少业务代码改动
```

---

## 二、对象存储解决什么问题

最简单的文件上传设计可能是：

```text
Client
↓
Spring Boot
↓
服务器本地磁盘
```

这种方案在单机开发环境中可以运行，但进入生产环境后会出现明显问题。

### 1. 本地磁盘容量有限

图片、PDF、视频和 AI 生成文件会持续增长。

```text
上传文件不断增加
↓
服务器磁盘逐渐被占满
↓
日志、数据库或应用无法继续写入
↓
服务异常甚至宕机
```

### 2. 多实例之间无法共享文件

假设系统部署了两个 Spring Boot 实例：

```text
Server A
Server B
```

用户把文件上传到了 A 的本地磁盘，但下载请求经过负载均衡后可能被转发到 B。

```text
文件实际在 A
下载请求进入 B
↓
B 找不到文件
↓
返回 404
```

### 3. 服务器迁移或重装时容易丢失文件

应用服务器应该尽量保持无状态。

如果业务文件和应用程序绑定在同一台机器上：

```text
服务器重装
容器重新创建
磁盘损坏
扩容迁移
```

都可能影响文件安全。

### 4. 不方便使用 CDN 和临时授权

对象存储通常原生支持：

```text
CDN 加速
私有 Bucket
预签名 URL
生命周期管理
版本控制
跨区域复制
分片上传
```

这些能力如果全部由业务系统自行实现，成本很高。

因此企业中更常见的架构是：

```text
Client
↓
Spring Boot
↓
OSS / OBS / S3

MySQL 只保存文件元数据
```

---

## 三、为什么不建议把大文件直接存入 MySQL

理论上可以使用 BLOB 字段保存文件，但通常不适合大量业务文件。

```text
MySQL
↓
BLOB
```

可能带来以下问题：

```text
数据库体积快速增长
数据库备份和恢复变慢
主从复制压力增加
查询和网络传输压力增加
冷热数据难以分离
数据库成本被大量文件占用
```

数据库更适合管理结构化业务数据，而不是承担大规模文件存储。

核心原则是：

> 对象存储保存文件本体，数据库保存文件的身份、归属、状态和定位信息。

---

## 四、标准文件上传架构

典型上传流程如下：

```text
用户提交文件
↓
Spring Boot 校验文件
↓
生成 Object Key
↓
上传对象存储
↓
获得上传结果
↓
数据库保存文件元数据
↓
返回 fileId
```

数据库可以保存：

```text
id
storage_type
bucket
object_key
file_name
file_size
content_type
file_hash
status
owner_id
business_type
business_id
create_time
update_time
```

其中最重要的字段包括：

```text
fileId       业务系统内部文件编号
objectKey    对象在 Bucket 中的唯一位置
fileName     用户上传时的原始文件名
fileSize     文件大小
contentType  文件类型
status       文件当前状态
ownerId      文件所属用户
```

Object Key 可以设计为：

```text
pdf/2026/08/03/550e8400-e29b-41d4-a716-446655440000.pdf
```

不建议直接使用用户上传的原始文件名作为 Object Key。

例如：

```text
合同.pdf
```

可能出现：

```text
文件重名覆盖
特殊字符和编码问题
路径注入风险
泄露用户信息
不同业务文件难以分类
```

更合理的方式是：

```text
业务目录 + 日期 + UUID + 扩展名
```

原始文件名仅作为元数据保存，用于前端展示和下载时设置文件名。

---

## 五、数据库有文件记录，但下载返回 404

今天练习中的核心故障是：

```text
MySQL 中存在 file_info 记录
↓
用户点击下载
↓
返回 404
```

404 通常表示当前请求对应的资源没有被找到，但真正原因可能发生在多个环节。

### 1. 对象已被删除，数据库记录没有删除

这是最直接的一种情况。

```text
MySQL：记录存在
OBS：对象不存在
```

可能原因包括：

```text
运维人员在 OBS 控制台手工删除文件
业务代码删除了对象，但数据库删除失败
清理任务误删文件
Bucket 生命周期规则自动删除了过期对象
迁移过程中漏掉了部分文件
测试脚本误操作生产 Bucket
```

本质是：

> 数据库记录和对象存储对象的生命周期没有保持一致。

### 2. 数据库写入成功，但对象上传失败

错误流程可能是：

```text
先插入数据库记录
↓
再上传 OBS
↓
OBS 上传失败
↓
数据库记录没有回滚或标记失败
```

最终数据库中存在一条看似正常的文件记录，但对应对象从未成功创建。

因此文件记录最好具有明确状态：

```text
UPLOADING
AVAILABLE
FAILED
DELETING
DELETED
```

只有状态为 `AVAILABLE` 的文件才允许生成下载地址。

### 3. 数据库保存了旧的完整 URL

例如数据库保存：

```text
https://old.example.com/pdf/a.pdf
```

后来系统更换了：

```text
访问域名
CDN 域名
Bucket
Region
云厂商
```

实际地址已经变成：

```text
https://new.example.com/pdf/a.pdf
```

历史记录中的旧 URL 仍然存在，访问时可能返回 404。

这个问题说明：

> 完整访问地址属于运行环境配置，不应该轻易固化到业务数据中。

### 4. Object Key 保存错误

例如实际上传对象为：

```text
pdf/2026/a.pdf
```

数据库保存成：

```text
pdf/2025/a.pdf
```

或者出现：

```text
大小写不一致
多了一个斜杠
少了目录前缀
URL 编码错误
中文和空格处理错误
上传使用临时 Key，入库使用正式 Key
```

对象存储通常把 Object Key 当作精确字符串处理，任何字符不同都可能定位不到对象。

### 5. Bucket、Region 或 Endpoint 配置错误

数据库中的 Object Key 可能正确，但应用访问了错误的存储位置。

例如：

```text
文件实际位于 Bucket A
应用配置却访问 Bucket B
```

或者：

```text
文件位于华北 Region
应用使用华南 Endpoint
```

此时也可能表现为对象不存在。

### 6. CDN 或反向代理路径配置错误

系统可能通过 CDN 域名访问对象：

```text
Client
↓
CDN
↓
OBS
```

如果 CDN 回源目录、域名绑定、缓存规则或路径重写错误，即使 OBS 中对象存在，也可能返回 404。

### 7. 文件迁移不完整

迁移系统时可能只迁移了 MySQL，没有迁移对象存储文件。

```text
数据库记录全部导入新环境
↓
部分对象没有复制到新 Bucket
↓
历史文件下载 404
```

因此云存储迁移不能只校验数据库行数，还要校验对象数量、文件大小和文件 Hash。

### 8. 权限问题不一定是 404

一般无权限访问更常见的是：

```text
403 Forbidden
```

但部分网关、CDN 或存储服务为了避免泄露对象是否存在，也可能对无权限请求返回类似资源不存在的结果。

因此排查时不能只根据 HTTP 状态码下结论，还要查看：

```text
对象存储错误码
响应头
应用日志
CDN 日志
预签名 URL 参数
```

---

## 六、数据库有记录但下载 404 的排查顺序

遇到这种问题，不应该直接猜测，而应该按链路逐层确认。

### 第一步：查询数据库记录

先根据 `fileId` 查询：

```text
storage_type
bucket
object_key
status
owner_id
file_size
```

确认：

```text
记录是否存在
状态是否为 AVAILABLE
Object Key 是否为空
Bucket 信息是否正确
```

### 第二步：直接检查对象是否存在

使用对象存储控制台或 SDK 检查：

```text
bucket + objectKey
```

如果对象不存在，问题位于上传、删除、清理或迁移环节。

如果对象存在，继续排查访问地址和权限。

### 第三步：对比真实 Key 和数据库 Key

重点检查：

```text
目录前缀
大小写
斜杠
中文和空格
文件扩展名
临时目录和正式目录
```

### 第四步：检查当前存储配置

检查：

```text
Bucket
Region
Endpoint
访问域名
CDN 域名
存储类型
```

确认应用没有访问错误环境。

### 第五步：检查下载链接生成逻辑

如果使用私有 Bucket，下载链接通常不是简单字符串拼接，而是由 SDK 生成预签名 URL。

需要检查：

```text
Object Key 是否正确
签名是否过期
签名使用的 Endpoint 是否正确
服务器时间是否准确
URL 是否被二次编码
```

### 第六步：查看删除、上传和迁移日志

通过 `fileId`、`objectKey`、`requestId` 或 `traceId` 查询日志：

```text
文件何时上传
上传是否成功
数据库何时写入
是否执行过删除
哪个任务执行了清理
文件是否经历迁移
```

标准排查路径可以总结为：

```text
数据库记录
↓
对象是否存在
↓
Key 是否一致
↓
Bucket / Region / Endpoint
↓
下载 URL 和权限
↓
上传、删除、清理、迁移日志
```

---

## 七、OBS 上传成功但数据库写入失败，如何补偿

典型流程是：

```text
上传 OBS 成功
↓
获得 objectKey
↓
写入 MySQL 失败
```

此时对象存储中出现了没有数据库归属的文件，也就是孤儿对象。

### 1. 为什么数据库事务无法自动回滚 OBS

MySQL 事务只能控制数据库操作：

```text
INSERT
UPDATE
DELETE
```

OBS 上传属于外部系统调用，不受本地数据库事务控制。

```text
MySQL ROLLBACK
≠
OBS 自动删除对象
```

因此必须主动执行补偿操作。

### 2. 首选方案：立即删除刚上传的对象

流程：

```text
OBS 上传成功
↓
数据库写入失败
↓
立即调用 OBS 删除 objectKey
↓
抛出上传失败异常
```

示例代码：

```java
public Long upload(MultipartFile file) {
    String objectKey = objectKeyGenerator.generate(file);

    storageService.upload(
        objectKey,
        file.getInputStream(),
        file.getContentType()
    );

    try {
        FileInfo fileInfo = buildFileInfo(file, objectKey);
        fileInfoMapper.insert(fileInfo);
        return fileInfo.getId();
    } catch (Exception databaseException) {
        try {
            storageService.delete(objectKey);
        } catch (Exception deleteException) {
            cleanupTaskService.create(objectKey, deleteException.getMessage());
        }

        throw databaseException;
    }
}
```

这里的核心是：

> 数据库失败后立即尝试反向撤销已经完成的 OBS 上传。

### 3. 删除补偿也可能失败

例如：

```text
网络异常
OBS 临时不可用
权限配置错误
删除请求超时
```

如果删除失败，不能只打印一条日志然后放弃。

应该记录一条持久化补偿任务：

```text
file_cleanup_task
```

可包含：

```text
id
storage_type
bucket
object_key
status
retry_count
next_retry_time
last_error
create_time
update_time
```

状态可以设计为：

```text
PENDING
PROCESSING
SUCCESS
FAILED
```

后台任务按照退避策略重试删除：

```text
第一次失败：1 分钟后重试
第二次失败：5 分钟后重试
第三次失败：30 分钟后重试
超过最大次数：告警并人工处理
```

### 4. XXL-JOB 适合作为兜底，而不是第一补偿手段

今天的初步思路是使用 XXL-JOB 定时扫描 OBS 和数据库的不一致数据。

这个方向可以保留，但不能代替即时补偿。

只依赖定时任务存在以下问题：

```text
孤儿对象会保留到下一次扫描
对象数量很大时全量扫描成本高
列举对象需要分页并产生 API 调用
临时文件可能被误判为孤儿对象
缺少 ownerId、businessId 等信息时无法安全补数据库
```

更合理的组合是：

```text
立即补偿删除
+
持久化失败重试任务
+
XXL-JOB 定时对账兜底
+
监控告警
```

### 5. 为什么通常不能自动补一条数据库记录

发现 OBS 中有文件、数据库中没有记录时，不建议直接自动插入数据库。

因为仅凭 Object Key 通常无法确定：

```text
文件属于哪个用户
属于哪个订单或任务
原始文件名是什么
文件是否上传完整
文件是否已经被业务取消
文件是否只是临时文件
文件是否本来就应该删除
```

因此孤儿对象通常更适合：

```text
确认超过安全时间窗口
↓
删除对象
```

而不是盲目恢复数据库记录。

---

## 八、为什么数据库保存 Object Key，而不是完整 URL

完整 URL 可能是：

```text
https://bucket.obs.example.com/pdf/2026/08/a.pdf
```

其中混合了两类信息：

```text
运行环境信息：域名、协议、CDN、Endpoint
业务定位信息：pdf/2026/08/a.pdf
```

数据库更适合只保存稳定的业务定位信息：

```text
pdf/2026/08/a.pdf
```

也就是 Object Key。

### 1. 更换域名时不需要修改历史数据

例如：

```text
旧域名：https://old.example.com
新域名：https://cdn.example.com
```

只要 Object Key 不变，运行时使用新配置即可生成新地址。

### 2. 测试和生产环境可以使用不同配置

```text
测试环境：test-bucket
生产环境：prod-bucket
```

业务表中的 Object Key 可以保持相同格式，不需要因为环境变化修改代码。

### 3. 可以切换公开访问和私有访问

公开文件可能直接拼接 CDN 地址：

```text
cdnDomain + objectKey
```

私有文件则需要：

```text
根据 objectKey 生成 10 分钟有效的预签名 URL
```

如果数据库保存的是永久完整 URL，就很难灵活切换访问策略。

### 4. 降低云厂商耦合

数据库不保存某家云厂商的固定域名，可以减少历史业务数据与供应商绑定。

但需要注意：

> 保存 Object Key 只能减少地址层面的耦合，迁移云厂商时仍然必须把真实文件复制到新的对象存储中。

修改域名不会让旧文件自动出现在新 Bucket。

---

## 九、迁移到其他云厂商，如何减少业务代码改动

仅仅做到“不写硬编码”还不够。

例如把：

```java
String endpoint = "https://obs.example.com";
```

改成：

```java
String endpoint = StorageConstants.ENDPOINT;
```

虽然减少了字符串散落，但业务代码仍然直接依赖 OBS。

真正需要解决的是：

> 业务层不能直接依赖某家云厂商的 SDK 和专属类型。

### 1. 将存储配置放入配置文件或配置中心

```yaml
storage:
  type: obs
  endpoint: https://obs.example.com
  bucket: production-file
  region: cn-north-4
  domain: https://file.example.com
```

不同环境使用不同配置：

```text
application-dev.yml
application-test.yml
application-prod.yml
Apollo / Nacos
```

Endpoint、Bucket、Region、域名等不应散落在业务代码中。

### 2. 定义统一的对象存储接口

```java
public interface ObjectStorageService {

    void upload(
        String objectKey,
        InputStream inputStream,
        String contentType
    );

    void delete(String objectKey);

    boolean exists(String objectKey);

    String generateDownloadUrl(
        String objectKey,
        Duration expiration
    );
}
```

业务层只依赖自己的接口：

```java
objectStorageService.upload(objectKey, inputStream, contentType);
```

不直接调用：

```java
obsClient.putObject(...);
```

### 3. 每家云厂商提供独立适配器

```java
public class ObsObjectStorageService
        implements ObjectStorageService {
}
```

```java
public class OssObjectStorageService
        implements ObjectStorageService {
}
```

```java
public class S3ObjectStorageService
        implements ObjectStorageService {
}
```

这些实现类负责处理：

```text
华为 OBS SDK
阿里云 OSS SDK
AWS S3 SDK
各自异常类型
各自签名方式
各自请求参数
```

业务层不需要理解厂商差异。

### 4. 通过配置选择实现类

Spring Boot 可以根据配置装配不同实现：

```java
@Configuration
public class StorageConfiguration {

    @Bean
    @ConditionalOnProperty(
        prefix = "storage",
        name = "type",
        havingValue = "obs"
    )
    public ObjectStorageService obsStorageService() {
        return new ObsObjectStorageService();
    }
}
```

以后从 OBS 切换到 OSS，主要变化为：

```text
新增或启用 OSS 适配器
修改 storage.type
修改 Bucket、Endpoint、Region 配置
迁移真实对象文件
```

业务 Service 和 Controller 不需要大规模修改。

### 5. 数据库模型保持厂商中立

建议保存：

```text
storage_type
bucket
object_key
file_name
file_size
content_type
status
```

避免把大量云厂商专属字段直接写入核心业务表。

如果确实需要专属属性，可以放入：

```text
扩展表
JSON 扩展字段
存储适配层内部
```

### 6. 云厂商迁移不仅是改配置

完整迁移流程通常包括：

```text
1. 实现并测试新的存储适配器
2. 建立新 Bucket 和权限策略
3. 全量复制旧对象
4. 校验文件数量、大小和 Hash
5. 增量同步迁移期间产生的新文件
6. 灰度切换下载流量
7. 新上传流量切换到新存储
8. 监控 404、403、下载耗时和失败率
9. 稳定运行一段时间后下线旧存储
```

重要系统可能在迁移期间短期双写：

```text
上传文件
↓
同时写旧存储和新存储
```

但双写本身会产生新的数据一致性问题：

```text
旧存储成功，新存储失败
新存储成功，旧存储失败
```

所以双写只能作为迁移手段，需要明确：

```text
主存储
失败重试
对账任务
切换时间
结束条件
```

不能长期无边界保留。

---

## 十、私有文件的正确下载流程

合同、支付凭证、用户上传 PDF 等文件不应该使用永久公开 URL。

更合理的流程是：

```text
用户请求下载 fileId
↓
后端查询 file_info
↓
校验登录状态
↓
校验 ownerId、角色和业务权限
↓
确认文件状态为 AVAILABLE
↓
根据 objectKey 生成短期预签名 URL
↓
返回客户端
↓
客户端直接从对象存储下载
```

预签名 URL 可以设置：

```text
5 分钟有效
10 分钟有效
一次业务操作所需的最短时间
```

需要注意：

> 预签名 URL 只是临时访问凭证，不能代替业务权限校验。

后端必须先判断用户是否有权访问该文件，再生成下载地址。

这样既避免所有文件流量经过 Spring Boot，也能控制文件访问权限。

---

## 十一、文件状态机设计

文件上传和处理通常不是一个瞬间完成的操作。

可以设计状态：

```text
UPLOADING
↓
AVAILABLE
↓
PROCESSING
↓
SUCCESS / FAILED
↓
DELETING
↓
DELETED
```

不同业务可以简化或扩展。

例如 PDF 转换平台：

```text
UPLOADING      正在上传原始 PDF
AVAILABLE      原始文件已上传，可进入转换
PROCESSING     正在转换
SUCCESS        新 PDF 已生成并上传
FAILED         转换失败
DELETING       正在删除原始文件和结果文件
DELETED        文件已删除
```

状态机可以解决以下问题：

```text
数据库记录存在，但文件还没上传完成
重复提交转换任务
上传失败却仍然允许下载
删除过程中用户继续访问
任务失败后无法判断是否需要补偿
```

---

## 十二、文件上传还需要关注的安全问题

对象存储架构不仅是把文件上传成功，还需要控制文件风险。

### 1. 限制文件大小

```text
普通图片最大 10 MB
PDF 最大 100 MB
视频根据业务单独限制
```

否则可能导致：

```text
带宽耗尽
内存占用过高
磁盘临时空间被占满
上传接口被恶意滥用
```

### 2. 校验扩展名和真实类型

不能只相信：

```text
前端传来的 content-type
用户上传的文件后缀
```

攻击者可能把可执行文件伪装成图片或 PDF。

需要结合：

```text
扩展名白名单
MIME 类型检测
文件头 Magic Number
病毒或恶意内容扫描
```

### 3. Object Key 不暴露敏感信息

不建议使用：

```text
身份证号/用户姓名/合同名称.pdf
```

作为公开可见路径。

更适合使用随机 ID，并把原始名称保存在数据库中。

### 4. 上传和下载需要限流

大文件上传、批量下载和 AI 批量生成文件都可能消耗大量带宽。

应结合：

```text
用户额度
上传频率限制
并发限制
单文件大小限制
每日总容量限制
异常流量告警
```

---

## 十三、今天练习题的完整回答

### 问题一：为什么数据库有记录但对象不存在

可能原因包括：

```text
对象被人工、业务代码或生命周期规则删除，但数据库未同步
数据库先写入成功，随后 OBS 上传失败
保存了旧完整 URL，域名或 CDN 已经变化
数据库 Object Key 与实际上传 Key 不一致
Bucket、Region 或 Endpoint 配置错误
CDN 回源路径配置错误
迁移时只迁移数据库，遗漏对象文件
```

排查时应先拿到数据库中的 Bucket、Object Key 和状态，再直接确认对象是否真实存在。

### 问题二：OBS 上传成功、数据库写入失败如何补偿

正确处理优先级是：

```text
立即删除刚上传的对象
↓
删除失败则记录持久化清理任务
↓
后台重试删除
↓
XXL-JOB 定时对账兜底
↓
超过重试次数进行告警
```

不建议只依赖定时任务，也不建议仅凭孤儿对象自动补数据库记录。

### 问题三：为什么保存 Object Key 而不是完整 URL

因为域名、CDN、Bucket、Endpoint、访问方式和云厂商都可能变化。

数据库只保存稳定的 Object Key，运行时根据配置拼接公开地址，或者通过 SDK 生成预签名 URL，可以减少历史数据修改和环境耦合。

### 问题四：如何降低云厂商迁移的代码改动

需要同时做到：

```text
Endpoint、Bucket、Region 和域名配置化
定义统一 ObjectStorageService 接口
OBS、OSS、S3 分别实现适配器
业务层只依赖统一接口
数据库模型保持厂商中立
迁移真实对象并进行数量、大小、Hash 校验
```

关键不是把硬编码放进常量类，而是通过接口和适配器隔离厂商 SDK。

---

## 十四、面试表达

可以这样回答：

> 在文件类业务中，我会使用对象存储保存图片、PDF、视频等大文件，数据库只保存文件元数据和 Object Key，而不是直接保存 BLOB 或固定完整 URL。上传时先校验文件并上传对象存储，再写入数据库；如果数据库写入失败，会立即删除刚上传的对象进行补偿，删除仍失败时记录持久化清理任务，由后台重试，定时对账只作为兜底。
>
> 如果数据库存在记录但下载返回 404，我会先查询文件状态、Bucket 和 Object Key，再通过控制台或 SDK 确认对象是否存在，然后检查 Key、Region、Endpoint、域名、预签名 URL、删除日志和迁移记录。常见原因包括对象被删除但数据库未同步、上传失败却留下数据库记录、Object Key 错误、旧完整 URL 失效以及迁移遗漏。
>
> 为了降低云厂商迁移成本，我会定义统一的 ObjectStorageService 接口，将 OBS、OSS、S3 的 SDK 封装在各自的实现类中，业务层只依赖统一接口，同时将 Endpoint、Bucket、Region 和域名放入配置中心。迁移时主要替换实现、修改配置并搬迁对象，避免大规模修改业务代码。

---

## 十五、今日总结

今天真正需要掌握的不是某家对象存储 SDK 的上传方法，而是完整的文件生命周期设计：

```text
文件校验
Object Key 生成
对象上传
元数据入库
状态管理
权限校验
预签名下载
失败补偿
异步删除
定时对账
监控告警
跨云迁移
```

核心原则可以总结为：

```text
对象存储保存文件本体
数据库保存元数据和 Object Key
业务层不直接依赖云厂商 SDK
跨系统操作必须设计补偿机制
私有文件必须先校验权限再生成临时下载地址
定时扫描是兜底，不是主要一致性方案
```

对象存储广泛存在于 PDF 工具平台、广告素材平台、AI 图片生成、OCR、报表导出和附件管理等业务中。真正的中级后端能力，不是会调用一次上传 SDK，而是能够处理文件一致性、权限、故障、清理和迁移问题。

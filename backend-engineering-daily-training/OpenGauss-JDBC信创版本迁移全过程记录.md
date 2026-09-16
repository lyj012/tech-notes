# OpenGauss JDBC 信创版本迁移全过程记录

## 1. 背景

目标：在信创版本代码分支中增加 OpenGauss JDBC 数据源支持，并验证 Zabbix HTTP Agent / 后端接口 / JDBC 执行链路能够正常工作。

本次记录目标：

- 记录实际调查过程
- 记录遇到的问题和错误判断
- 记录代码迁移思路
- 保留所有基于事实确认的信息
- 作为后续开发、排障、交接参考

---

## 2. 初始问题

当前现象：

- Zabbix HTTP Agent 调用接口返回：

```json
{"code":500,"msg":"No static resource monitor/datasource/execute.","data":null,"success":false}
```

- 最开始怀疑接口、部署、版本问题。
- 后续确认需要先明确实际使用的代码分支和部署版本。

---

## 3. Git 分支确认

重要结论：

当前开发目标不是普通 dev 分支，而是信创版本对应分支。

之前 dev 分支存在 OpenGauss 修改，但不能直接认为目标分支也是同一套代码结构。

需要先确认：

- 负责人指定的信创版本分支
- 当前部署环境对应版本
- 当前代码链路

最终确认：

应基于信创版本分支迁移 OpenGauss 修改。

---

## 4. dev 分支已有 OpenGauss 修改

已存在修改：

### 4.1 DataSourceTypeEnum

新增：

```java
OPEN_GAUSS("opengauss", "org.opengauss.Driver", "jdbc:opengauss://{}:{}/{}")
```

作用：

- 支持识别 opengauss 类型
- 提供 JDBC Driver
- 提供 JDBC URL 模板

---

### 4.2 DataBaseExecuteFactory

增加 OpenGauss 路由。

作用：

让 OpenGauss 类型进入对应 Handler。

---

### 4.3 OpenGaussExecuteHandler

已有实现：

- 创建 OpenGauss 数据源
- 使用 org.opengauss.Driver
- 拼接 jdbc:opengauss URL
- 执行 SQL

但是该实现属于 dev 分支原有 jdbcproxy 体系。

---

### 4.4 Maven 依赖

增加：

```xml
<groupId>org.opengauss</groupId>
<artifactId>opengauss-jdbc</artifactId>
<version>6.0.1-og</version>
```

作用：

提供：

```java
org.opengauss.Driver
```

---

## 5. 为什么不能直接复制代码

调查发现：

不同分支 JDBC 体系不同。

不能简单复制：

- DataSourceTypeEnum
- DataBaseExecuteFactory
- Handler

原因：

### dev 分支

结构：

```
monitor.jdbcproxy.*
```

特点：

- 自己维护 Handler
- Factory 使用 switch / instanceof
- 参数对象是 JdbcExecute

---

### 信创版本分支

结构不同：

```
monitor.collector.modular.jdbc.*
```

特点：

- 使用 collector JDBC 链路
- Handler 通过 getType() 注册
- 参数对象不同

因此迁移不是复制文件，而是迁移功能。

---

## 6. collector-server 调查记录

发现信创版本依赖：

```xml
com.rusong:collector-server:${revision}
```

revision 对应版本。

本地检查发现：

- 有旧版本 sources 可参考
- 没有确认的目标版本源码
- Hope 仓库本身不是 collector-server 源码仓库

因此不能直接修改一个不存在的源码目录。

当前结论：

如果 OpenGauss Handler 位于 collector-server，则需要先确认对应源码来源。

---

## 7. 当前正确迁移原则

不要：

- cherry-pick 整个 dev 提交
- 全量 merge dev 到信创分支
- 复制 monitor-proxy 文件

原因：

两个分支代码结构不同。

应该：

1. 保留 dev OpenGauss 修改作为参考
2. 在信创版本实际 JDBC 链路中实现同等能力
3. 保持信创版本已有架构
4. 再进行编译测试

---

## 8. 当前验证顺序

推荐顺序：

### 第一阶段：代码确认

确认：

- OpenGauss 类型在哪里定义
- JDBC Handler 在哪里实现
- Factory 如何加载 Handler
- Driver 依赖在哪里声明

---

### 第二阶段：功能实现

实现：

- OpenGauss enum
- OpenGauss Handler
- JDBC Driver 依赖

---

### 第三阶段：编译验证

验证：

- Maven 编译通过
- 依赖正常
- Spring 加载正常

---

### 第四阶段：接口验证

验证：

- 数据源连接
- SQL 执行
- 返回数据格式

---

## 9. 本次过程中的问题复盘

问题：

一开始把 dev 分支 OpenGauss 代码和信创分支代码体系混在一起分析。

正确认识：

- 分支名称相似不代表代码结构相同
- 迁移功能不等于复制文件
- 首先需要确认目标分支架构

---

## 10. 当前状态

已完成：

- 确认目标方向
- 确认 dev 已有 OpenGauss 修改
- 确认不能直接复制
- 梳理迁移风险

未完成：

- 信创版本最终 OpenGauss 修改位置确认
- 代码迁移
- 编译验证
- 部署测试

---

## 11. 后续执行原则

所有修改必须基于事实：

- 当前代码仓库
- 当前分支
- 当前依赖
- 当前部署环境

不要根据文件名、jar 名称、旧版本经验直接推断。

# Day69 OpenGauss JDBC 架构迁移问题排查记录

## 主题

从 `/monitor/datasource/execute` 接口失败，到确认 v8.2 正式 JDBC 采集链路的完整排查过程。

## 一、问题背景

目标：将 OpenGauss JDBC 能力迁移到信创版本模板中。

原计划：

1. 导入 OpenGauss 模板
2. 增加 OpenGauss JDBC 执行能力
3. 验证采集数据准确性

在代码修改完成并重新打包部署后，测试接口：

```
POST /monitor/datasource/execute
```

返回：

```
No static resource monitor/datasource/execute.
```

初始判断需要确认：

- jar 是否包含 OpenGauss JDBC 驱动
- OpenGauss Handler 是否存在
- Controller 是否存在
- 当前项目真实 JDBC 采集架构是什么

## 二、第一次错误方向

之前新增 OpenGauss JDBC 支持时，在 Hope 侧增加：

```
/monitor/datasource/execute
```

并增加：

```
MonitorDataSourceController
MonitorDataSourceService
JdbcProxyController
OpenGaussExecuteHandler
```

但是负责人反馈：

> 代码写错地方，需要撤回，迁移到信创版本测试。

随后执行 revert。

事实：

这不是 OpenGauss JDBC 能力错误，而是代码归属位置错误。

## 三、重新验证 jar

服务器检查：

```bash
ls -lh hope-web-app-8.2.0.jar
```

确认 jar 上传成功。

检查依赖：

```
BOOT-INF/lib/opengauss-jdbc-6.0.1-og.jar
BOOT-INF/lib/monitor-core-8.2.0.jar
BOOT-INF/lib/collector-server-8.2.0.jar
```

确认：

- JDBC 驱动存在
- collector-server 存在
- OpenGauss Handler 存在

## 四、发现接口不存在

测试：

```bash
curl -X POST http://127.0.0.1:8082/monitor/datasource/execute
```

返回：

```
No static resource monitor/datasource/execute
```

进一步检查：

```
MonitorDataSourceController
```

发现当前版本没有 execute 方法。

因此结论：

不是 jar 打包失败。
不是驱动缺失。
不是 Spring 扫描失败。

而是当前版本没有这个 HTTP 接口。

## 五、追查历史架构

通过 Git 历史确认：

旧版本存在：

```
Zabbix HTTP Agent
        ↓
/monitor/datasource/execute
        ↓
Hope Service
        ↓
/monitor/jdbcproxy/execute
        ↓
JDBC Handler
```

后续架构调整：

```
jdbc全面移动到采集器包中
```

旧接口被删除。

因此：

`/monitor/datasource/execute` 属于旧架构，不是当前 v8.2 正式链路。

## 六、确认 v8.2 正式采集链路

当前真实流程：

```
模板
 ↓
collect_item
 ↓
collect_type = 1(DB)
 ↓
Hope生成 CollectJobParam
 ↓
collector-server 调度
 ↓
ItemCollectJob
 ↓
ItemCollectServiceImpl.collectData()
 ↓
collectDataByJdbc()
 ↓
DataBaseExecuteFactory
 ↓
OpenGaussExecuteHandler
 ↓
数据库
```

关键点：

数据库 SQL 不通过 HTTP Controller 暴露。

SQL 来源：

```
collect_item.param.sql
```

数据库连接来源：

```
hostInterfaceConfig.db
```

## 七、为什么 Oracle/MySQL 模板可以工作

之前误认为：

```
Oracle JDBC模板
→ /monitor/datasource/execute
```

实际 v8.2：

```
Oracle数据库模板
collect_type=1
```

走：

```
collectDataByJdbc()
```

不是 HTTP Agent。

## 八、当前 OpenGauss 改造思路

错误思路：

```
新增 HTTP Controller
恢复 /monitor/datasource/execute
```

正确思路：

```
迁移现有数据库模板
 ↓
保持 collect_type=1
 ↓
配置 OpenGauss 数据库连接
 ↓
验证 DataBaseExecuteFactory
 ↓
验证 OpenGaussExecuteHandler
 ↓
验证数据准确性
```

## 九、当前进度

三个阶段：

### 第一阶段：模板面板迁移

状态：进行中。

需要：

- 基于 OpenGauss ODBC 模板迁移指标
- 保留 SQL、触发器、展示逻辑
- 调整为 v8.2 DB 采集模型

### 第二阶段：代码实现

状态：已完成基础能力。

已有：

- OpenGauss JDBC Driver
- DataSourceTypeEnum
- DataBaseExecuteFactory
- OpenGaussExecuteHandler

无需新增 Hope Controller。

### 第三阶段：数据准确性

状态：未开始。

下一步：

先用最简单 SQL 验证：

```sql
select 1 as value
```

跑通完整采集链路。

## 十、问题解决方法总结

遇到接口不存在问题时，不应该直接补接口。

正确排查顺序：

1. 验证部署产物
2. 验证接口是否属于当前版本
3. 查看 Git 历史确认架构变化
4. 查看模板实际使用方式
5. 还原完整数据链路
6. 再决定是否修改代码

本次问题最终不是代码缺失，而是对旧架构和新架构理解错误。

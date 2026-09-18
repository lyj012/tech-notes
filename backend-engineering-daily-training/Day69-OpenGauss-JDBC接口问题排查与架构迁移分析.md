# Day69 OpenGauss JDBC 接口问题排查与架构迁移分析

## 一、问题背景

本次任务目标：将 OpenGauss JDBC 能力迁移到信创版本进行验证。

原计划：

1. 模板面板迁移
2. 代码实现
3. 数据准确性验证

当前阶段主要处于第 1 阶段向第 2 阶段过渡时的问题定位。

---

## 二、最初现象

服务器部署新的 `hope-web-app-8.2.0.jar` 后，验证 OpenGauss JDBC 执行接口：

```bash
curl -X POST http://127.0.0.1:8082/monitor/datasource/execute
```

返回：

```
No static resource monitor/datasource/execute.
```

初步怀疑：

- jar 包没有打入新代码
- Controller 没有加载
- Maven 打包遗漏模块
- 部署的 jar 不是最新版本

---

## 三、第一轮验证：确认 jar 内容

检查：

```bash
jar tf hope-web-app-8.2.0.jar
```

确认：

存在：

```
BOOT-INF/lib/opengauss-jdbc-6.0.1-og.jar
BOOT-INF/lib/monitor-core-8.2.0.jar
BOOT-INF/lib/collector-server-8.2.0.jar
```

并确认 OpenGauss 相关类存在：

```
OpenGaussExecuteHandler.class
DataSourceTypeEnum.class
DataBaseExecuteFactory.class
```

结论：

不是 jar 打包问题。

---

## 四、第二轮验证：为什么接口不存在？

检查当前代码发现：

当前分支不存在：

```
POST /monitor/datasource/execute
```

存在的是：

```
POST /monitor/datasource/checkDataSource
```

作用：

- 检查数据源连接
- 不执行模板 SQL
- 不返回 SQL 查询结果

因此：

`/monitor/datasource/execute` 调用失败是代码事实，不是部署异常。

---

## 五、关键问题：之前系统怎么实现？

没有直接认为代码缺失，而是回溯 Git 历史和旧版本架构。

发现旧版本存在：

```
Zabbix HTTP Agent
        |
        v
/monitor/datasource/execute
        |
        v
/monitor/jdbcproxy/execute
        |
        v
JDBC Handler
```

旧实现中：

Hope 负责暴露 HTTP 接口。

后来发生架构调整：

```
jdbc 全面移动到采集器包中
```

旧接口和旧 proxy 实现被删除。

因此：

当前不是简单少了一个 Controller。

而是架构已经发生变化。

---

## 六、负责人要求撤回的原因

之前新增 OpenGauss JDBC 支持时，把部分 JDBC 执行能力放到了 Hope 侧。

Git 提交显示：

```
feat: 新增 OpenGauss JDBC 执行支持
```

随后：

```
revert: 撤回 Hope 侧 OpenGauss JDBC 错误实现
```

撤回原因：

代码实现位置错误。

不是 OpenGauss JDBC 方向错误。

不是 JDBC 模板需求不存在。

核心问题：

JDBC 能力应该属于 collector 侧，而不是 Hope 侧重新实现。

---

## 七、当前 v8.2 正式 JDBC 链路

当前 Oracle/MySQL 在线模板验证后发现：

不是：

```
Zabbix
 -> HTTP Agent
 -> /monitor/datasource/execute
```

而是：

```
采集任务
 -> CollectJobParam
 -> ItemCollectJob
 -> ItemCollectServiceImpl.collectData()
 -> collectType = DB
 -> collectDataByJdbc()
 -> DataBaseExecuteFactory
 -> OpenGaussExecuteHandler
 -> BaseDataBaseExecuteHandler.execute()
```

关键入口：

```
POST /itemCollect/getItemData
```

这是 collector 内部采集入口。

---

## 八、模板问题如何理解

历史模板中存在：

```
{$MONITOR.SERVER.URL}/monitor/datasource/execute
```

原因：

模板来源于旧架构。

当前代码已经迁移到新的采集模型。

所以出现：

```
模板设计
       !=
当前代码架构
```

不能直接认为模板错，也不能直接恢复旧接口。

需要先确认当前版本标准接入方式。

---

## 九、当前正确推进思路

不要立即写 Controller。

正确顺序：

### 第一步：模板迁移

确认：

- 信创数据库模板
- 采集类型
- SQL
- 宏配置
- 指标定义

是否符合当前 v8.2 架构。

### 第二步：代码实现

基于当前架构补充缺失能力。

优先复用：

```
DataBaseExecuteFactory
BaseDataBaseExecuteHandler
OpenGaussExecuteHandler
```

而不是恢复旧 Hope JDBC 代码。

### 第三步：数据准确性

验证：

- SQL 执行结果
- 指标值
- Zabbix 展示
- 触发器判断

---

## 十、本次排查方法总结

遇到接口不存在问题，不应该直接补接口。

正确思路：

1. 看现象
2. 验证部署产物
3. 确认代码是否存在
4. 查看 Git 历史
5. 还原架构演进
6. 找当前版本真实链路
7. 再决定是否开发

最终结论：

当前问题不是 jar 包问题。

不是 OpenGauss 驱动问题。

不是数据库连接问题。

核心问题是：

旧 JDBC HTTP 接口与当前 v8.2 采集架构不一致。

下一步应该围绕当前采集架构完成信创模板迁移和验证，而不是恢复旧接口。

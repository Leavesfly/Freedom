# SQL执行

<cite>
**本文档引用文件**  
- [SqlExecutor.java](file://src/main/java/alchemystar/freedom/sql/SqlExecutor.java)
- [CreateExecutor.java](file://src/main/java/alchemystar/freedom/sql/CreateExecutor.java)
- [InsertExecutor.java](file://src/main/java/alchemystar/freedom/sql/InsertExecutor.java)
- [SelectExecutor.java](file://src/main/java/alchemystar/freedom/sql/SelectExecutor.java)
- [DeleteExecutor.java](file://src/main/java/alchemystar/freedom/sql/DeleteExecutor.java)
- [FrontendConnection.java](file://src/main/java/alchemystar/freedom/engine/net/handler/frontend/FrontendConnection.java)
- [Session.java](file://src/main/java/alchemystar/freedom/engine/session/Session.java)
- [Table.java](file://src/main/java/alchemystar/freedom/meta/Table.java)
- [BPTree.java](file://src/main/java/alchemystar/freedom/index/bp/BPTree.java)
- [InsertVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/InsertVisitor.java)
- [SelectVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/SelectVisitor.java)
- [TableFilter.java](file://src/main/java/alchemystar/freedom/sql/select/TableFilter.java)
- [Trx.java](file://src/main/java/alchemystar/freedom/transaction/Trx.java)
- [TrxManager.java](file://src/main/java/alchemystar/freedom/transaction/TrxManager.java)
</cite>

## 目录
1. [引言](#引言)
2. [核心调度机制](#核心调度机制)
3. [执行器实现分析](#执行器实现分析)
   - [CreateExecutor](#createexecutor)
   - [InsertExecutor](#insertexecutor)
   - [SelectExecutor](#selectexecutor)
   - [DeleteExecutor](#deleteexecutor)
4. [事务与会话管理](#事务与会话管理)
5. [数据检索与索引机制](#数据检索与索引机制)
6. [不支持的SQL类型与扩展方法](#不支持的sql类型与扩展方法)

## 引言
本文档深入分析Freedom数据库系统的SQL执行引擎实现，重点阐述`SqlExecutor`作为核心调度中枢如何根据Druid解析出的SQL语句类型实例化相应的执行器，并详细说明各执行器与会话、连接及存储引擎的交互机制。

## 核心调度机制

`SqlExecutor`作为SQL执行的调度中枢，负责解析SQL语句并分发到相应的执行器。其核心流程如下：

```mermaid
flowchart TD
Start([开始执行SQL]) --> Parse["解析SQL语句\nMySqlStatementParser.parseStatement()"]
Parse --> CheckType["判断SQLStatement类型"]
CheckType --> |CREATE TABLE| Create["实例化CreateExecutor\n执行execute()"]
CheckType --> |INSERT| Insert["实例化InsertExecutor\n执行execute(session)"]
CheckType --> |SELECT| Select["实例化SelectExecutor\n执行execute()"]
CheckType --> |DELETE| Delete["实例化DeleteExecutor\n执行execute(session)"]
CheckType --> |其他类型| Error["抛出异常：不支持的语句类型"]
Create --> Response["响应客户端\nOkResponse.response()"]
Insert --> ResponseWithRows["响应客户端\nOkResponse.responseWithAffectedRows()"]
Select --> End
Delete --> ResponseWithRows
Error --> End
Response --> End
ResponseWithRows --> End
End([结束])
```

**图示来源**
- [SqlExecutor.java](file://src/main/java/alchemystar/freedom/sql/SqlExecutor.java#L1-L50)

**本节来源**
- [SqlExecutor.java](file://src/main/java/alchemystar/freedom/sql/SqlExecutor.java#L1-L50)

## 执行器实现分析

### CreateExecutor
`CreateExecutor`负责处理CREATE TABLE语句，通过`CreateVisitor`解析表结构信息，并将新表注册到`TableManager`中。

```mermaid
sequenceDiagram
participant SqlExecutor
participant CreateExecutor
participant CreateVisitor
participant TableManager
SqlExecutor->>CreateExecutor : execute()
CreateExecutor->>CreateExecutor : init()
CreateExecutor->>CreateVisitor : accept(sqlStatement)
CreateVisitor-->>CreateExecutor : 构建Table对象
CreateExecutor->>TableManager : addTable(table, true)
TableManager-->>CreateExecutor : 完成表创建
CreateExecutor-->>SqlExecutor : 返回
```

**图示来源**
- [CreateExecutor.java](file://src/main/java/alchemystar/freedom/sql/CreateExecutor.java#L1-L31)
- [CreateVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/CreateVisitor.java)

**本节来源**
- [CreateExecutor.java](file://src/main/java/alchemystar/freedom/sql/CreateExecutor.java#L1-L31)

### InsertExecutor
`InsertExecutor`处理INSERT语句，解析值列表并调用存储层写入数据。执行过程包含事务日志记录和数据持久化。

```mermaid
sequenceDiagram
participant SqlExecutor
participant InsertExecutor
participant InsertVisitor
participant Session
participant Table
SqlExecutor->>InsertExecutor : execute(session)
InsertExecutor->>InsertExecutor : init()
InsertExecutor->>InsertVisitor : accept(sqlStatement)
InsertVisitor-->>InsertExecutor : 构建IndexEntry
InsertExecutor->>Session : addLog(table, OpType.insert, null, indexEntry)
InsertExecutor->>Table : insert(indexEntry)
Table->>BPTree : insert(indexEntry, true)
BPTree-->>Table : 完成插入
Table-->>InsertExecutor : 返回
InsertExecutor-->>SqlExecutor : 返回
```

**图示来源**
- [InsertExecutor.java](file://src/main/java/alchemystar/freedom/sql/InsertExecutor.java#L1-L39)
- [InsertVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/InsertVisitor.java)
- [Table.java](file://src/main/java/alchemystar/freedom/meta/Table.java)

**本节来源**
- [InsertExecutor.java](file://src/main/java/alchemystar/freedom/sql/InsertExecutor.java#L1-L39)

### SelectExecutor
`SelectExecutor`处理SELECT语句，结合B+树索引进行数据检索，并通过`FrontendConnection`返回结果集。

```mermaid
sequenceDiagram
participant SqlExecutor
participant SelectExecutor
participant SelectVisitor
participant TableFilter
participant Cursor
participant FrontendConnection
SqlExecutor->>SelectExecutor : execute()
SelectExecutor->>SelectExecutor : init()
SelectExecutor->>SelectVisitor : accept(sqlStatement)
SelectVisitor-->>SelectExecutor : 构建TableFilter和SelectItems
loop 遍历数据
SelectExecutor->>TableFilter : next()
TableFilter->>Cursor : next()
Cursor-->>TableFilter : 返回IndexEntry
TableFilter-->>SelectExecutor : 设置当前行
alt 存在WHERE条件
SelectExecutor->>SelectExecutor : checkWhereCondition()
SelectExecutor-->>SelectExecutor : 条件不匹配则跳过
end
SelectExecutor->>SelectResponse : writeRow()
SelectResponse->>FrontendConnection : 发送数据行
end
SelectExecutor->>SelectResponse : writeLastEof()
SelectResponse->>FrontendConnection : 发送结束包
SelectExecutor-->>SqlExecutor : 返回
```

**图示来源**
- [SelectExecutor.java](file://src/main/java/alchemystar/freedom/sql/SelectExecutor.java#L1-L122)
- [SelectVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/SelectVisitor.java)
- [TableFilter.java](file://src/main/java/alchemystar/freedom/sql/select/TableFilter.java)

**本节来源**
- [SelectExecutor.java](file://src/main/java/alchemystar/freedom/sql/SelectExecutor.java#L1-L122)

### DeleteExecutor
`DeleteExecutor`处理DELETE语句，先收集待删除的记录，然后逐个删除并记录事务日志。

```mermaid
sequenceDiagram
participant SqlExecutor
participant DeleteExecutor
participant DeleteVisitor
participant TableFilter
participant Cursor
participant Session
participant Table
SqlExecutor->>DeleteExecutor : execute(session)
DeleteExecutor->>DeleteExecutor : init()
DeleteExecutor->>DeleteVisitor : accept(sqlStatement)
DeleteVisitor-->>DeleteExecutor : 构建TableFilter
loop 收集待删除记录
DeleteExecutor->>TableFilter : next()
TableFilter->>Cursor : next()
Cursor-->>TableFilter : 返回IndexEntry
TableFilter-->>DeleteExecutor : 检查WHERE条件
alt 条件匹配
DeleteExecutor->>toDelete : 添加记录
end
end
loop 删除记录
DeleteExecutor->>Session : addLog(table, OpType.delete, delItem, null)
DeleteExecutor->>Table : delete(delItem)
Table->>BPTree : delete(indexEntry)
BPTree-->>Table : 完成删除
end
DeleteExecutor->>OkResponse : responseWithAffectedRows()
DeleteExecutor-->>SqlExecutor : 返回
```

**图示来源**
- [DeleteExecutor.java](file://src/main/java/alchemystar/freedom/sql/DeleteExecutor.java#L1-L74)
- [DeleteVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/DeleteVisitor.java)
- [TableFilter.java](file://src/main/java/alchemystar/freedom/sql/select/TableFilter.java)

**本节来源**
- [DeleteExecutor.java](file://src/main/java/alchemystar/freedom/sql/DeleteExecutor.java#L1-L74)

## 事务与会话管理
SQL执行过程中，`Session`对象管理事务状态，`Trx`类实现事务的ACID特性。每个执行器通过`Session`与事务系统交互。

```mermaid
classDiagram
class Session {
+FrontendConnection con
+Trx trx
+execute(sql, con)
+begin()
+commit()
+rollback()
+addLog(table, opType, before, after)
}
class Trx {
+int state
+int trxId
+Log[] logs
+begin()
+commit()
+rollback()
+addLog(table, opType, before, after)
+redo()
+undo()
}
class TrxManager {
+AtomicInteger trxIdCount
+newTrx()
+newEmptyTrx()
}
class Log {
+long lsn
+int trxId
+int logType
+int opType
+String tableName
+IndexEntry before
+IndexEntry after
}
Session --> Trx : "拥有"
TrxManager --> Trx : "创建"
Session --> FrontendConnection : "关联"
InsertExecutor --> Session : "调用addLog"
DeleteExecutor --> Session : "调用addLog"
```

**图示来源**
- [Session.java](file://src/main/java/alchemystar/freedom/engine/session/Session.java)
- [Trx.java](file://src/main/java/alchemystar/freedom/transaction/Trx.java)
- [TrxManager.java](file://src/main/java/alchemystar/freedom/transaction/TrxManager.java)
- [InsertExecutor.java](file://src/main/java/alchemystar/freedom/sql/InsertExecutor.java)
- [DeleteExecutor.java](file://src/main/java/alchemystar/freedom/sql/DeleteExecutor.java)

**本节来源**
- [Session.java](file://src/main/java/alchemystar/freedom/engine/session/Session.java)
- [Trx.java](file://src/main/java/alchemystar/freedom/transaction/Trx.java)
- [TrxManager.java](file://src/main/java/alchemystar/freedom/transaction/TrxManager.java)

## 数据检索与索引机制
系统采用B+树索引进行高效数据检索。`TableFilter`结合`WhereVisitor`优化查询条件，`BPTree`实现索引的搜索操作。

```mermaid
flowchart TD
Start([开始查询]) --> BuildKey["构建查询键\nTableFilter.buildLowUpEntry()"]
BuildKey --> CheckConflict["检查条件冲突\nisConflict"]
CheckConflict --> |冲突| ReturnEmpty["返回空结果集"]
CheckConflict --> |无冲突| Search["执行索引搜索\nTable.searchRange()"]
Search --> BPTree["BPTree.searchRange()"]
BPTree --> GetRoot["获取根节点"]
GetRoot --> Traverse["遍历B+树"]
Traverse --> Leaf["到达叶子节点"]
Leaf --> GetPosition["获取起始位置\ngetFirst()"]
GetPosition --> Cursor["创建Cursor"]
Cursor --> ReturnCursor["返回游标"]
ReturnCursor --> SelectExecutor["SelectExecutor遍历结果"]
ReturnEmpty --> SelectExecutor
SelectExecutor --> End([结束])
```

**图示来源**
- [TableFilter.java](file://src/main/java/alchemystar/freedom/sql/select/TableFilter.java#L1-L278)
- [BPTree.java](file://src/main/java/alchemystar/freedom/index/bp/BPTree.java#L1-L277)
- [Table.java](file://src/main/java/alchemystar/freedom/meta/Table.java)

**本节来源**
- [TableFilter.java](file://src/main/java/alchemystar/freedom/sql/select/TableFilter.java#L1-L278)
- [BPTree.java](file://src/main/java/alchemystar/freedom/index/bp/BPTree.java#L1-L277)

## 不支持的SQL类型与扩展方法
当前系统仅支持基本的CRUD操作，以下SQL类型尚未支持：

- **UPDATE语句**：需要实现`UpdateExecutor`和`UpdateVisitor`
- **复杂JOIN**：目前仅支持INNER JOIN，需扩展`SelectVisitor`以支持其他JOIN类型
- **子查询**：需要增强`SelectVisitor`的解析能力
- **聚合函数**：需在`SelectExecutor`中实现SUM、COUNT等函数
- **事务控制语句**：如SAVEPOINT等，需扩展`FrontendConnection`的处理能力

扩展方法建议：
1. 实现`UpdateExecutor`类，类似`InsertExecutor`但包含旧值和新值
2. 增强`SelectVisitor`以支持更多SQL语法结构
3. 在`SqlExecutor`中添加对新语句类型的判断和分发
4. 实现相应的`Visitor`类来解析特定SQL结构
5. 更新`Table`类以支持更多操作类型

**本节来源**
- [SqlExecutor.java](file://src/main/java/alchemystar/freedom/sql/SqlExecutor.java#L45-L50)
- [SelectVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/SelectVisitor.java#L70-L75)
# INSERT执行

<cite>
**本文档中引用的文件**   
- [InsertExecutor.java](file://src/main/java/alchemystar/freedom/sql/InsertExecutor.java)
- [InsertVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/InsertVisitor.java)
- [IndexEntry.java](file://src/main/java/alchemystar/freedom/meta/IndexEntry.java)
- [Table.java](file://src/main/java/alchemystar/freedom/meta/Table.java)
- [Trx.java](file://src/main/java/alchemystar/freedom/transaction/Trx.java)
- [Log.java](file://src/main/java/alchemystar/freedom/transaction/log/Log.java)
- [Session.java](file://src/main/java/alchemystar/freedom/engine/session/Session.java)
</cite>

## 目录
1. [简介](#简介)
2. [核心组件分析](#核心组件分析)
3. [INSERT执行流程](#insert执行流程)
4. [InsertVisitor值提取机制](#insertvisitor值提取机制)
5. [IndexEntry构建过程](#indexentry构建过程)
6. [WAL协议实现](#wal协议实现)
7. [存储层写入流程](#存储层写入流程)
8. [主键约束检查](#主键约束检查)
9. [多值插入处理](#多值插入处理)
10. [错误处理策略](#错误处理策略)
11. [性能优化建议](#性能优化建议)

## 简介
本文档深入解析Freedom数据库系统中INSERT语句的执行流程，重点阐述`InsertExecutor`如何通过`InsertVisitor`提取值列表并构建`IndexEntry`数据条目。详细说明`execute`方法中事务日志记录与存储层写入的执行顺序，强调WAL（预写日志）协议的实现机制。文档还将分析主键约束检查机制和错误处理策略，并提供性能优化建议。

## 核心组件分析

**Section sources**
- [InsertExecutor.java](file://src/main/java/alchemystar/freedom/sql/InsertExecutor.java#L1-L38)
- [InsertVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/InsertVisitor.java#L1-L98)
- [IndexEntry.java](file://src/main/java/alchemystar/freedom/meta/IndexEntry.java#L1-L181)
- [Table.java](file://src/main/java/alchemystar/freedom/meta/Table.java#L1-L172)

## INSERT执行流程

```mermaid
sequenceDiagram
participant Client as "客户端"
participant SqlExecutor as "SqlExecutor"
participant InsertExecutor as "InsertExecutor"
participant InsertVisitor as "InsertVisitor"
participant Session as "Session"
participant Trx as "Trx"
participant LogStore as "LogStore"
participant Table as "Table"
participant BPTree as "BPTree"
Client->>SqlExecutor : execute(sql, con, session)
SqlExecutor->>InsertExecutor : new InsertExecutor(sqlStatement)
InsertExecutor->>InsertExecutor : execute(session)
InsertExecutor->>InsertExecutor : init()
InsertExecutor->>InsertVisitor : sqlStatement.accept(insertVisitor)
InsertVisitor->>InsertVisitor : visit(SQLInsertStatement)
InsertVisitor->>InsertVisitor : mapColumnIndex(x)
InsertExecutor->>InsertVisitor : buildInsertEntry()
InsertExecutor->>Session : addLog(table, OpType.insert, null, indexEntry)
Session->>Trx : addLog(table, opType, before, after)
Trx->>LogStore : appendLog(log)
InsertExecutor->>Table : insert(indexEntry)
Table->>BPTree : clusterIndex.insert(entry, true)
BPTree->>BPNode : root.insert(entry, tree, isUnique)
BPNode->>BPNode : innerCheckExist(entry)
BPNode->>BPNode : innerInsert(entry)
BPTree-->>Table : 插入完成
Table->>BPTree : secondIndex.insert(entry, false)
BPTree-->>Table : 二级索引插入完成
InsertExecutor-->>SqlExecutor : 执行完成
SqlExecutor-->>Client : OkResponse
```

**Diagram sources**
- [InsertExecutor.java](file://src/main/java/alchemystar/freedom/sql/InsertExecutor.java#L12-L38)
- [InsertVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/InsertVisitor.java#L26-L98)
- [Table.java](file://src/main/java/alchemystar/freedom/meta/Table.java#L64-L71)
- [Trx.java](file://src/main/java/alchemystar/freedom/transaction/Trx.java#L44-L62)
- [Log.java](file://src/main/java/alchemystar/freedom/transaction/log/Log.java#L74-L118)

## InsertVisitor值提取机制

```mermaid
flowchart TD
Start([开始visit]) --> CheckTableSource["检查tableSource类型"]
CheckTableSource --> ValidateType{"是否为SQLExprTableSource?"}
ValidateType --> |否| ThrowException["抛出运行时异常"]
ValidateType --> |是| GetTable["通过TableManager获取Table"]
GetTable --> MapColumnIndex["调用mapColumnIndex建立列名与索引映射"]
MapColumnIndex --> GetValuesClause["获取ValuesClause"]
GetValuesClause --> AssignValueExpr["将values赋值给valueExpr"]
AssignValueExpr --> End([visit完成])
subgraph mapColumnIndex流程
MapStart([开始mapColumnIndex]) --> LoopColumns["遍历SQL列列表"]
LoopColumns --> GetColumnExpr["获取SQLExpr"]
GetColumnExpr --> ExtractColumnName["提取列名"]
ExtractColumnName --> UpdateMaps["更新attributeIndexMap和indexAttributeMap"]
UpdateMaps --> CheckLoop{"是否遍历完成?"}
CheckLoop --> |否| LoopColumns
CheckLoop --> |是| MapEnd([映射完成])
end
```

**Diagram sources**
- [InsertVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/InsertVisitor.java#L33-L67)
- [InsertVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/InsertVisitor.java#L73-L98)

**Section sources**
- [InsertVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/InsertVisitor.java#L26-L98)

## IndexEntry构建过程

```mermaid
classDiagram
class IndexEntry {
+Value[] values
+IndexDesc indexDesc
+buildInsertEntry() IndexEntry
+getCompareEntry() IndexEntry
+getDeleteCompareEntry() IndexEntry
+getBytes() byte[]
+read(byte[]) void
+compareIndex(IndexEntry) int
+getLength() int
}
class ClusterIndexEntry {
+ClusterIndexEntry(Value[] values)
}
class NotLeafEntry {
+NotLeafEntry(Value[] values)
}
class Value {
+int type
+getLength() int
+getBytes() byte[]
+read(byte[]) void
+compare(Value) int
}
class ValueString {
+ValueString()
+ValueString(String value)
}
class ValueLong {
+ValueLong()
+ValueLong(long value)
}
class ValueInt {
+ValueInt()
+ValueInt(int value)
}
class ValueBoolean {
+ValueBoolean()
+ValueBoolean(boolean value)
}
IndexEntry <|-- ClusterIndexEntry
IndexEntry <|-- NotLeafEntry
Value <|-- ValueString
Value <|-- ValueLong
Value <|-- ValueInt
Value <|-- ValueBoolean
InsertVisitor --> IndexEntry : "构建"
InsertVisitor --> Value : "创建"
Table --> IndexEntry : "包含"
```

**Diagram sources**
- [IndexEntry.java](file://src/main/java/alchemystar/freedom/meta/IndexEntry.java#L18-L180)
- [InsertVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/InsertVisitor.java#L45-L67)
- [Value.java](file://src/main/java/alchemystar/freedom/meta/value/Value.java)
- [ValueString.java](file://src/main/java/alchemystar/freedom/meta/value/ValueString.java)
- [ValueLong.java](file://src/main/java/alchemystar/freedom/meta/value/ValueLong.java)

**Section sources**
- [IndexEntry.java](file://src/main/java/alchemystar/freedom/meta/IndexEntry.java#L18-L180)
- [InsertVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/InsertVisitor.java#L45-L67)

## WAL协议实现

```mermaid
sequenceDiagram
participant InsertExecutor as "InsertExecutor"
participant Session as "Session"
participant Trx as "Trx"
participant LogStore as "LogStore"
participant Database as "Database"
InsertExecutor->>Session : addLog(table, OpType.insert, null, indexEntry)
Session->>Trx : addLog(table, opType, before, after)
Note over Trx : 验证日志条目类型
Trx->>Log : 创建Log实例
Trx->>Log : 设置LSN、日志类型、事务ID
Trx->>Log : 设置操作类型、表名
Trx->>Log : 设置before=null, after=indexEntry
Trx->>LogStore : appendLog(log)
LogStore->>Database : 获取文件通道
Database->>FileUtils : append(fileChannel, dst)
FileUtils->>OS : 文件系统写入
OS-->>FileUtils : 写入完成
FileUtils-->>Database : 完成
Database-->>LogStore : 完成
LogStore-->>Trx : 完成
Trx->>Trx : 将日志添加到内存logs列表
Trx-->>Session : 完成
Session-->>InsertExecutor : 完成
```

**Diagram sources**
- [Trx.java](file://src/main/java/alchemystar/freedom/transaction/Trx.java#L44-L62)
- [Log.java](file://src/main/java/alchemystar/freedom/transaction/log/Log.java#L74-L118)
- [LogStore.java](file://src/main/java/alchemystar/freedom/store/log/LogStore.java#L72-L112)
- [FileUtils.java](file://src/main/java/alchemystar/freedom/store/fs/FileUtils.java)

**Section sources**
- [Trx.java](file://src/main/java/alchemystar/freedom/transaction/Trx.java#L44-L62)
- [Log.java](file://src/main/java/alchemystar/freedom/transaction/log/Log.java#L74-L118)

## 存储层写入流程

```mermaid
flowchart TD
Start([InsertExecutor执行]) --> GetTable["获取Table实例"]
GetTable --> CallInsert["调用table.insert(indexEntry)"]
CallInsert --> InsertCluster["clusterIndex.insert(entry, true)"]
subgraph 聚集索引插入
InsertCluster --> CheckSize["检查条目大小 <= Max/3"]
CheckSize --> IsUnique{"是否为唯一索引?"}
IsUnique --> |是| CheckExist["innerCheckExist(entry)"]
CheckExist --> |存在| ThrowException["抛出重复键异常"]
CheckExist --> |不存在| IsLeaf{"是否为叶子节点?"}
IsUnique --> |否| IsLeaf
IsLeaf --> |是| NoSplit{"无需分裂?"}
NoSplit --> |是| InnerInsert["innerInsert(entry)"]
NoSplit --> |否| SplitNode["分裂节点"]
SplitNode --> CreateNodes["创建左右节点"]
SplitNode --> Rebalance["重新平衡"]
SplitNode --> UpdateParent["更新父节点"]
InnerInsert --> CompleteCluster["聚集索引插入完成"]
end
subgraph 二级索引插入
CompleteCluster --> LoopSecond["遍历secondIndexes"]
LoopSecond --> InsertSecond["secondIndex.insert(entry, false)"]
InsertSecond --> CompleteSecond["二级索引插入完成"]
LoopSecond --> CheckLoop{"是否遍历完成?"}
CheckLoop --> |否| LoopSecond
CheckLoop --> |是| End([全部插入完成])
end
```

**Diagram sources**
- [Table.java](file://src/main/java/alchemystar/freedom/meta/Table.java#L64-L71)
- [BPTree.java](file://src/main/java/alchemystar/freedom/index/bp/BPTree.java#L232-L236)
- [BPNode.java](file://src/main/java/alchemystar/freedom/index/bp/BPNode.java#L126-L228)

**Section sources**
- [Table.java](file://src/main/java/alchemystar/freedom/meta/Table.java#L64-L71)
- [BPNode.java](file://src/main/java/alchemystar/freedom/index/bp/BPNode.java#L126-L228)

## 主键约束检查
在B+树节点插入过程中，系统会自动进行主键约束检查。当插入操作针对唯一索引（`isUnique=true`）时，系统会调用`innerCheckExist(entry)`方法检查是否存在重复键值。如果发现重复键，则抛出运行时异常，阻止插入操作。这种检查机制确保了主键的唯一性约束得到严格执行。

```mermaid
flowchart TD
Start([开始插入]) --> CheckUnique{"isUnique为true?"}
CheckUnique --> |否| DirectInsert["直接插入"]
CheckUnique --> |是| CallCheckExist["调用innerCheckExist(entry)"]
CallCheckExist --> SearchTree["在B+树中搜索entry"]
SearchTree --> Found{"找到匹配项?"}
Found --> |是| ThrowDuplicate["抛出重复键异常"]
Found --> |否| ContinueInsert["继续插入流程"]
ContinueInsert --> CheckLeaf{"是否为叶子节点?"}
CheckLeaf --> |是| CheckSplit{"需要分裂?"}
CheckLeaf --> |否| TraverseChildren["遍历子节点"]
```

**Diagram sources**
- [BPNode.java](file://src/main/java/alchemystar/freedom/index/bp/BPNode.java#L126-L140)
- [BPNode.java](file://src/main/java/alchemystar/freedom/index/bp/BPNode.java#L141-L228)

**Section sources**
- [BPNode.java](file://src/main/java/alchemystar/freedom/index/bp/BPNode.java#L126-L228)

## 多值插入处理
当前系统仅支持单条INSERT语句的处理。在`InsertVisitor`的`visit`方法中，代码注释明确指出"只支持单条insert"。系统通过`SQLInsertStatement.ValuesClause`获取值列表，但仅处理第一条记录。对于多值插入场景，如`INSERT INTO table VALUES (1,'a'), (2,'b')`，系统目前无法正确处理，需要进行功能扩展。

```java
// 只支持单条insert
valueExpr = valuesClause.getValues();
```

要支持多值插入，需要修改`InsertVisitor`和`InsertExecutor`的实现，使其能够遍历所有值列表，并为每个值列表构建独立的`IndexEntry`，然后依次执行插入操作。

**Section sources**
- [InsertVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/InsertVisitor.java#L33-L67)

## 错误处理策略
系统采用分层的错误处理策略。在解析阶段，如果遇到不支持的表源类型，会抛出运行时异常。在值提取阶段，如果遇到不支持的表达式类型（非字面量），也会抛出运行时异常。在索引插入阶段，如果条目大小超过页面空间的1/3，会抛出运行时异常。在唯一性检查阶段，如果发现重复键，会抛出运行时异常。所有异常都会中断当前操作并向上抛出，确保数据一致性。

```mermaid
graph TD
A[INSERT执行] --> B{解析阶段}
B --> C["不支持的表源类型?"]
C --> |是| D["抛出RuntimeException"]
C --> |否| E{值提取阶段}
E --> F["非字面量表达式?"]
F --> |是| G["抛出RuntimeException"]
F --> |否| H{插入阶段}
H --> I["条目大小 > Max/3?"]
I --> |是| J["抛出RuntimeException"]
I --> |否| K{唯一性检查}
K --> L["发现重复键?"]
L --> |是| M["抛出RuntimeException"]
L --> |否| N["插入成功"]
```

**Section sources**
- [InsertVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/InsertVisitor.java#L26-L98)
- [BPNode.java](file://src/main/java/alchemystar/freedom/index/bp/BPNode.java#L126-L228)

## 性能优化建议
1. **批量插入优化**：当前系统仅支持单条插入，建议扩展为批量插入功能。可以通过收集多个`IndexEntry`，然后一次性写入日志和存储层，减少I/O操作次数。

2. **日志写入优化**：考虑实现日志缓冲机制，在内存中累积多个日志条目后再批量写入磁盘，提高日志写入效率。

3. **索引插入优化**：对于大量数据插入场景，可以暂时禁用二级索引，先完成聚集索引插入，再批量构建二级索引。

4. **预分配空间**：对于已知大小的插入操作，可以预分配`Value[]`数组和字节缓冲区，避免频繁的内存分配。

5. **并行处理**：在多核环境下，可以考虑并行处理多个插入操作，充分利用系统资源。

6. **连接池优化**：复用`InsertVisitor`实例，避免频繁创建和销毁对象，减少GC压力。

**Section sources**
- [InsertExecutor.java](file://src/main/java/alchemystar/freedom/sql/InsertExecutor.java#L12-L38)
- [InsertVisitor.java](file://src/main/java/alchemystar/freedom/sql/parser/InsertVisitor.java#L26-L98)
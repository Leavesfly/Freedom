# Freedom - 轻量级SQL执行引擎

<div align="center">

[![Java Version](https://img.shields.io/badge/Java-1.6-orange.svg)](https://www.oracle.com/java/)
[![Maven](https://img.shields.io/badge/Maven-3.x-blue.svg)](https://maven.apache.org/)
[![Netty](https://img.shields.io/badge/Netty-5.0.0.Alpha1-green.svg)](https://netty.io/)
[![Druid](https://img.shields.io/badge/Druid-1.1.11-blue.svg)](https://github.com/alibaba/druid)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

**一个从零开始的数据库引擎实现 🚀**

</div>

## 📖 项目简介

Freedom 是一个专为学习和理解数据库底层原理而设计的轻量级 SQL 执行引擎。通过"从零造轮子"的方式，完整实现了从网络协议解析到磁盘存储的数据库核心功能，帮助开发者深入掌握数据库系统的内部机制。

### 🎯 设计目标

- **教育导向**：通过实际代码理解数据库核心概念
- **完整实现**：覆盖网络协议、SQL解析、存储引擎、事务管理等核心模块
- **MySQL兼容**：支持标准MySQL客户端和JDBC连接
- **可扩展性**：模块化设计，便于功能扩展和优化

### 🏗️ 系统架构

Freedom采用索引组织表，通过Druid SQL Parse来将SQL翻译为对应的索引操作符进而进行对应的语义操作。整体架构包含以下核心模块：

- **网络协议层**：基于Netty实现MySQL协议解析
- **SQL解析层**：使用Druid解析SQL，生成操作符树  
- **执行引擎**：将SQL操作转化为对B+树索引的访问
- **存储层**：B+树以Page为单位组织，通过Buffer Manager与文件系统交互
- **事务层**：通过Redo/Undo日志实现WAL协议
- **元数据管理**：表结构信息落盘，重启加载      
## ✨ 核心特性

### 🌐 网络层
- **MySQL协议兼容**：完整实现MySQL通信协议，支持握手、认证、命令处理
- **连接管理**：基于Netty的高性能NIO网络框架
- **多客户端支持**：同时支持MySQL客户端和JDBC连接

### 🔍 SQL处理
- **SQL解析**：基于Alibaba Druid的SQL语法解析
- **DDL支持**：CREATE TABLE等数据定义语言
- **DML支持**：INSERT、SELECT、DELETE等数据操作语言
- **复杂查询**：支持WHERE条件、JOIN查询、范围查询

### 💾 存储引擎
- **B+树索引**：基于B+树的磁盘存储结构，支持变长记录
- **索引类型**：聚簇索引（主键）+ 二级索引支持
- **页面管理**：完整的页面缓存、加载、分配机制
- **文件系统**：抽象的文件存储接口

### 🔄 事务管理
- **ACID特性**：通过WAL（Write-Ahead Logging）保证事务原子性
- **日志机制**：Redo/Undo双重日志保护
- **崩溃恢复**：系统重启后自动进行数据恢复
- **LSN管理**：日志序列号确保操作顺序

### 📊 元数据管理
- **表结构管理**：表定义的持久化存储和动态加载
- **索引元数据**：索引结构信息的完整管理
- **属性定义**：灵活的数据类型支持      
## 🛠️ 技术架构

### 技术选型

| 组件 | 技术栈 | 版本 | 作用 |
|------|--------|------|------|
| 核心语言 | Java | 1.6 | 基础运行环境 |
| 网络框架 | Netty | 5.0.0.Alpha1 | 高性能NIO通信 |
| SQL解析器 | Alibaba Druid | 1.1.11 | SQL语法树解析 |
| 日志框架 | Logback + SLF4J | 1.1.2 / 1.6.0 | 系统日志记录 |
| 构建工具 | Maven | 3.x | 项目依赖管理 |
| 测试框架 | JUnit | 4.12 | 单元测试支持 |
| 工具库 | Commons Lang/IO | 2.6 / 2.4 | 通用工具类 |

### SQL解析与优化

Freedom采用成熟的Druid SQL Parse作为解析器，将文本SQL转换为操作符树：

- **WHERE条件处理**：采用访问者模式构建表达式树，支持复杂谓词计算
- **JOIN优化**：基于B+树范围扫描优化，大幅减少笛卡尔积计算量
- **谓词下推**：在底层游标处理过程中预先过滤数据，减少上层处理负载

### 查询优化示例

对于如下SQL查询：
```sql
SELECT a.*, b.* FROM t_archer a JOIN t_rider b 
WHERE a.id >= 3 AND a.id <= 11 AND b.id >= 19 AND b.id <= 31
```

通过B+树范围扫描：
- 表a的扫描范围：[3, 11] 
- 表b的扫描范围：[19, 31]
- 循环次数从原来的15×15减少到4×4，优化到原来的7.1%      



 
### leaf和node节点在Page中的不同
虽然leaf和node在page中组织结构一致，但其item包含的项确有区别。由于Freedom采用的是索引组织表，所以对于leaf在聚簇索引(clusterIndex)和二级索引(secondaryIndex)中对item的表示也有区别,如下图所示:      
![](https://oscimg.oschina.net/oscnet/up-d649ccfa767c9ba83998209dfea4d576fb8.png)      
其中在二级索引搜索时通过secondaryIndex通过index-key找到对应的clusterId,再通过
clusterId在clusterIndex中找到对应的row记录。     
由于要落盘，所以Freedom在node节点中的item里面写入了index-key对应的pageno,
这样就可以容易的从磁盘恢复所有的索引结构了。
### B+Tree在文件中的组织
有了Page结构，我们就可以将数据承载在一个个page大小的内存里面，同时还可以将page刷新到对应的文件里。有了node.item中的pageno，我们就可以较容易的进行文件和内存结构之间的互相映射了。
B+树在磁盘文件中的组织如下图所示:      
![](https://oscimg.oschina.net/oscnet/up-c1740d8e165978f46129c0f9115a4d997e8.png)      
B+树在内存中相对应的映射结构如下图所示:      
![](https://oscimg.oschina.net/oscnet/up-cc89c3f7c515e6e572e8efb90d534372a0a.png)      
文件page和内存page中的内容基本是一致的,除了一些内存page中特有的字段，例如dirty等。
### 每个索引一个B+树
在Freedom中，每个索引都是一颗B+树，对记录的插入和修改都要对所有的B+树进行操作。
### B+Tree的测试
笔者通过一系列测试case,例如随机变长记录对B+树进行插入并落盘，修复了其中若干个非常诡异的corner case。
### B+Tree的todo
笔者这里只是完成了最简单的B+树结构，没有给其添加并发修改的锁机制，也没有在B+树做操作的时候记录log来保证B+树在宕机等灾难性情况下的一致性,所以就算完成了这么多的工作量，距离一个高并发高可用的bptree还有非常大的距离。
## Meta Data
table的元信息由create table所创建。创建之后会将元信息落盘，以便Freedom在重启的时候加载表信息。每张表的元信息只占用一页的空间，依旧复用page结构，主要保存的是聚簇索引和二级索引的信息。元信息对应的Item如下图所示:      
![](https://oscimg.oschina.net/oscnet/up-62b37cd2e6d45c59ca4a5b2bf643f04a60d.png)      
如果想让mybatis可以自动生成关于Freedom的代码，还需实现一些特定的sql来展现Freedom的元信息。这个在笔者另一个项目rider中有这样的实现。原理如下图所示:      
![](https://oscimg.oschina.net/oscnet/up-11a9f315bd4b77683d3b7dffd105f8c8ce3.png)      
实现了上述4类SQL之后，mybatis-generator就可以通过jdbc从Freedom获取元信息进而自动生成代码了。

## 事务支持
由于当前Freedom并没有保证并发，所以对于事务的支持只做了最简单的WAL协议。通过记录redo/undolog从而实现原子性。
### redo/undo log协议格式
Freedom在每做一个修改操作时，都会生成一条日志，其中记录了修改前(undo)和修改后(redo)的行信息，redo用来回滚,redo用来宕机recover。结构如下图所示:      
![](https://oscimg.oschina.net/oscnet/up-8672b50ccdf8ae795a07cff3dff33a674e2.png)     
### WAL协议
WAL协议很好理解，就是在事务commit前将当前事务中所产生的的所有log记录刷入磁盘。
Freedom自然也做了这个操作，使得可以在宕机后通过log恢复出所有的数据。      
![](https://oscimg.oschina.net/oscnet/up-83029d948723aea197e01beba17b26b1741.png)      

### 回滚的实现
由于日志中记录了undo，所以对于一个事务的回滚直接通过日志进行undo即可。如下图所示:      
![](https://oscimg.oschina.net/oscnet/up-b32f8511b08a172fb35bb8649f465501ac5.png)      
### 宕机恢复
Freedom如果在page全部刷盘之后关机，则可以由通过加载page的方式获取原来的数据。
但如果突然宕机,例如kill -9之后，则可以通过WAL协议中记录的redo/undo log来重新
恢复所有的数据。由于时间和精力所限，笔者并没有实现基于LSN的检查点机制。
## Freedom运行
```
git clone https://github.com/alchemystar/Freedom.git
// 并没有做打包部署的工作，所以最简单的方法是在java编辑器里面
run alchemystar.freedom.engine.server.main
```
以下是笔者实际运行Freedom的例子:    
![](https://oscimg.oschina.net/oscnet/up-4a0c9b32a6cf09d157bb07f357261793d0a.JPEG)      
join查询      
![](https://oscimg.oschina.net/oscnet/up-5f0ea6f33bfff904e8146d995319598c08d.JPEG)      
delete回滚      
![](https://oscimg.oschina.net/oscnet/up-34e77d656b9b19035e601133b150a2a6039.JPEG)      
## Freedom todo
Freedom还有很多工作没有完成，例如有层次的锁机制和MVCC等，由于工作忙起来就耽搁了。
于是笔者就看了看MySQL源码的实现理解了一下锁和MVCC实现原理，并写了两篇博客。比起
自己动手撸实在是轻松太多了^_^。

### MVCC
https://my.oschina.net/alchemystar/blog/1927425      
### 二阶段锁 
https://my.oschina.net/alchemystar/blog/1438839      
## 尾声
在造轮子的过程中一开始是非常有激情非常快乐的。但随着系统越来越庞大，复杂性越来越高，进度就会越来越慢，还时不时要推翻自己原来的设想并重新设计，然后再协同修改关联的所有代码，就如同泥沼，越陷越深。至此，笔者才领悟了软件工程最重要的其实是控制复杂度！始终保持简洁的接口和优雅的设计是实现一个大型系统的必要条件。    


## github链接
https://github.com/alchemystar/Freedom     
## 码云链接
https://gitee.com/alchemystar/Freedom     


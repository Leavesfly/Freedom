# Freedom - 轻量级SQL执行引擎

[![Java Version](https://img.shields.io/badge/Java-1.6-orange.svg)](https://www.oracle.com/java/)
[![Maven](https://img.shields.io/badge/Maven-3.x-blue.svg)](https://maven.apache.org/)
[![Netty](https://img.shields.io/badge/Netty-5.0.0.Alpha1-green.svg)](https://netty.io/)
[![Druid](https://img.shields.io/badge/Druid-1.1.11-blue.svg)](https://github.com/alibaba/druid)

## 项目简介

Freedom 是一个基于 Java 的轻量级 SQL 执行引擎，旨在通过"从零造轮子"的方式深入理解数据库底层原理。项目完整实现了从网络协议解析到磁盘存储的数据库核心功能，支持与 MySQL 客户端和 JDBC 的兼容连接。

### 核心特性

- **MySQL 协议兼容**：支持标准 MySQL 客户端和 JDBC 连接
- **SQL 支持**：实现 CREATE TABLE、INSERT、SELECT、DELETE 等基本 SQL 操作
- **B+ 树存储**：基于 B+ 树的磁盘存储结构，支持变长记录
- **事务处理**：通过 WAL（Write-Ahead Logging）实现事务原子性
- **JOIN 查询**：支持多表 JOIN 操作（基于笛卡尔积 + 条件过滤优化）
- **崩溃恢复**：通过 Redo/Undo 日志实现系统崩溃后的数据恢复
- **元数据管理**：表结构的持久化存储和加载
## 技术选型

| 组件 | 技术栈 | 版本 | 说明 |
|------|--------|------|------|
| 核心语言 | Java | 1.6 | 兼容性考虑 |
| 网络框架 | Netty | 5.0.0.Alpha1 | NIO 网络通信 |
| SQL 解析 | Alibaba Druid | 1.1.11 | SQL 语法解析 |
| 日志框架 | Logback + SLF4J | 1.1.2 / 1.6.0 | 日志记录 |
| 构建工具 | Maven | 3.x | 项目构建管理 |
| 测试框架 | JUnit | 4.12 | 单元测试 |

## 快速开始

### 环境要求

- JDK 1.6 或更高版本
- Maven 3.x
- Git

### 编译运行

```bash
# 克隆项目
git clone https://github.com/alchemystar/Freedom.git
cd Freedom

# 编译项目
mvn clean compile

# 运行服务
# 在 IDE 中运行主类：alchemystar.freedom.engine.server.FreedomServer
```

### 连接测试

服务启动后，默认监听 **8090** 端口，可以使用 MySQL 客户端连接：

```bash
# 使用 MySQL 客户端连接（默认用户名/密码：pay/MiraCle）
mysql -h localhost -P 8090 -u pay -p
```

### 基本使用示例

```sql
-- 创建表
CREATE TABLE test (
    id BIGINT PRIMARY KEY,
    name VARCHAR(256),
    comment VARCHAR(256),
    KEY name_idx (name)
);

-- 插入数据
INSERT INTO test (id, name, comment) VALUES (1, 'freedom', 'test data');
INSERT INTO test (id, name, comment) VALUES (2, 'database', 'more data');

-- 查询数据
SELECT * FROM test WHERE id = 1;
SELECT * FROM test WHERE name = 'freedom';
SELECT * FROM test WHERE id >= 1 AND id <= 10;

-- JOIN 查询
SELECT a.id, a.name, b.comment 
FROM test a JOIN test b ON a.id = b.id + 1 
WHERE a.id > 1;

-- 删除数据
DELETE FROM test WHERE id = 1;
```
## 项目结构

```
src/main/java/alchemystar/freedom/
├── access/              # 索引访问接口
├── config/              # 系统配置
├── engine/              # 引擎核心
├── index/               # B+ 树索引实现
├── meta/                # 元数据管理
├── sql/                 # SQL 执行器
├── store/               # 存储管理
├── transaction/         # 事务管理
└── test/                # 单元测试
```

## 测试

```bash
# 运行所有测试
mvn test

# 运行特定测试
mvn test -Dtest=CreateTest
mvn test -Dtest=SelectTest
```

## 限制与不足

- **无并发控制**：未实现锁机制和 MVCC
- **功能不完整**：缺少 UPDATE、外键等高级功能
- **版本过旧**：使用 Java 1.6 和 Netty 5.0.0.Alpha1

## 学习资源

- [MVCC 实现原理](https://my.oschina.net/alchemystar/blog/1927425)
- [二阶段锁机制](https://my.oschina.net/alchemystar/blog/1438839)

## 项目链接

- [GitHub 仓库](https://github.com/alchemystar/Freedom)
- [码云仓库](https://gitee.com/alchemystar/Freedom)

## 许可证

本项目仅供学习和研究使用，不建议用于生产环境。

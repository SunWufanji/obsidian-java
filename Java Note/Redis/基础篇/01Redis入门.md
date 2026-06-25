# Redis 入门

## 1.1 认识 NoSQL

### 什么是 NoSQL

**NoSQL** 可以翻译做 Not Only SQL（不仅仅是 SQL），或者 No SQL（非 SQL 的）数据库。是相对于传统关系型数据库而言，有很大差异的一种特殊数据库，因此也称之为**非关系型数据库**。

### 关系型 vs 非关系型

| 对比维度 | 关系型数据库 | 非关系型数据库 |
|---------|------------|--------------|
| 数据结构 | 结构化（有严格表结构约束） | 非结构化（键值、文档、图等） |
| 数据关联 | 有关联（外键） | 无关联（靠业务逻辑维护） |
| 查询方式 | SQL 统一标准 | 各数据库语法不同 |
| 事务 | 支持 ACID | 不支持或只保证基本一致性 |
| 存储方式 | 磁盘 | 内存为主 |
| 扩展性 | 垂直扩展（主从备份） | 水平扩展（数据分片） |
| 查询性能 | 复杂查询强 | 简单查询极快 |

### NoSQL 的数据格式

**键值型（Redis）：**
```
key: "user:1"
value: {"id":1, "name":"张三", "age":21}
```

**文档型（MongoDB）：**
```json
{
  "id": 1,
  "name": "张三",
  "orders": [{"id": 1, "item": "荣耀6"}]
}
```

**图格式（Neo4j）：** 节点 + 关系，适合社交网络

---

## 1.2 认识 Redis

### Redis 是什么

Redis 诞生于 2009 年，全称是 **Re**mote **D**ictionary **S**erver（远程词典服务器），是一个**基于内存的键值型 NoSQL 数据库**。

### Redis 的核心特征

- **键值（key-value）型**，value 支持多种数据结构（String、Hash、List、Set、SortedSet 等）
- **单线程**，每个命令具备原子性
- **低延迟，速度快**（基于内存 + IO 多路复用 + 优秀的编码实现）
- **支持数据持久化**（RDB、AOF）
- **支持主从集群、分片集群**
- **支持多语言客户端**（Java、Python、Go 等）

### Redis 为什么快？

```
1. 数据存在内存中，读写不需要磁盘 IO
2. 单线程模型，避免线程切换和锁竞争
3. IO 多路复用，一个线程处理多个连接
4. 底层数据结构高效（SDS、跳表、压缩列表等）
```

### Redis 官网

https://redis.io/

---

## 1.3 安装 Redis

### 方式一：WSL（推荐，Windows 用户）

```bash
# 打开 WSL 终端
sudo apt update
sudo apt install redis-server -y

# 启动 Redis
redis-server

# 新开终端连接
redis-cli
```

### 方式二：Linux 编译安装

```bash
# 安装依赖
yum install -y gcc tcl

# 解压
tar -xzf redis-6.2.6.tar.gz
cd redis-6.2.6

# 编译安装
make && make install
```

安装后默认路径 `/usr/local/bin`，包含：
- `redis-server`：服务端启动脚本
- `redis-cli`：命令行客户端
- `redis-sentinel`：哨兵启动脚本

### 启动方式

**前台启动（不推荐）：**
```bash
redis-server
```

**后台启动（修改配置文件）：**
```properties
# redis.conf 关键配置
bind 0.0.0.0        # 允许任意 IP 访问
daemonize yes       # 后台运行
requirepass 123321  # 设置密码
port 6379           # 监听端口
databases 1         # 数据库数量
maxmemory 512mb     # 最大内存
logfile "redis.log" # 日志文件
```

```bash
redis-server redis.conf
```

**停止服务：**
```bash
redis-cli -a 123321 shutdown
```

**开机自启（Linux systemd）：**
```bash
# 创建服务文件
vi /etc/systemd/system/redis.service
```
```ini
[Unit]
Description=redis-server
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/bin/redis-server /usr/local/src/redis-6.2.6/redis.conf
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
```bash
systemctl daemon-reload
systemctl enable redis
systemctl start redis
```

---

## 1.4 命令行客户端演示

### 基本连接

```bash
redis-cli [options] [commands]
```

常用选项：
- `-h 127.0.0.1`：指定 Redis 节点 IP，默认 127.0.0.1
- `-p 6379`：指定端口，默认 6379
- `-a 123321`：指定密码

```bash
# 连接并测试
redis-cli -h 127.0.0.1 -p 6379 -a 123321 ping
# 返回 PONG 说明连接成功

# 进入交互控制台
redis-cli
```

### 选择数据库

Redis 默认有 16 个数据库，编号 0~15，默认使用 0 号库：

```bash
# 切换到 1 号库
select 1

# 切换回 0 号库
select 0
```

### 快速体验

```bash
127.0.0.1:6379> set name zhangsan
OK
127.0.0.1:6379> get name
"zhangsan"
127.0.0.1:6379> del name
(integer) 1
127.0.0.1:6379> exists name
(integer) 0
```

---

## 1.5 图形化客户端演示

### 推荐工具

**Another Redis Desktop Manager（免费，推荐）：**
- GitHub：https://github.com/qishibo/AnotherRedisDesktopManager

**RedisDesktopManager（RDM）：**
- Windows 安装包：https://github.com/lework/RedisDesktopManager-Windows/releases

### 连接配置

| 配置项 | 值 |
|-------|-----|
| Host | 127.0.0.1 |
| Port | 6379 |
| Password | （你设置的密码） |

### 注意

图形化客户端只是方便查看数据，**不是必须的**。跟着课程学习用 `redis-cli` 完全够用。

---

## 面试简答

**Q: Redis 为什么这么快？**

A: 四个原因：
1. 数据存在内存，避免磁盘 IO
2. 单线程模型，无锁竞争
3. IO 多路复用，高效处理并发连接
4. 底层数据结构经过专门优化（SDS、跳表等）

**Q: Redis 和 MySQL 的区别？**

A: Redis 是内存型 NoSQL 数据库，读写速度极快，适合缓存、计数、排行榜等场景；MySQL 是磁盘型关系型数据库，支持复杂查询和事务，适合持久化存储业务数据。两者通常配合使用，Redis 做缓存层，MySQL 做持久层。

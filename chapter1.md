##复制

###什么是复制？

复制是指将一个服务器上的所有改变复制到另一个服务器。

###复制的两个重要应用

1. 横向扩展
2. 添加冗余来保证高可用性


##Mysql复制

###复制的基本步骤

首先，需要建立一个简单的复制，即单一的主从复制（Master---->Slave）

建立单一的主从复制，需要经过以下三个步骤：

1. 配置一个服务器作为Master
2. 配置一个服务器作为Slave  
3. 将Slave连接到Master

###配置Master

要将服务器配置为Master，要确保服务器有一个活动的二进制日志文件和一个唯一的服务器ID。二进制日志文件记录了Master上的所有改变，并且可以在Slave上重新执行。后面会详细介绍二进制日志文件，现在只需要知道以上的功能即可。服务器ID是用来区分服务器。要创建二进制日志文件和服务器ID，你需要停掉你的服务器，并按以下内容修改你的 *my.cnf* 配置文件。

```
[mysqld]
user             = mysql
pid-file         = /var/run/mysqld/mysql.pid
socket           = /var/run/mysqld/mysql.sock
port             = 3306
basedir          = /usr
datadir          = /var/lib/mysql
tmpdir           = /tmp
log-bin          = master-bin
log-bin-index    = master-bin.index
server-id        = 1
```
其中以下部分是添加的配置部分

```
log-bin          = master-bin
log-bin-index    = master-bin.index
server-id        = 1
```

###配置选项解释###

log-bin
: 二进制日志文件产生的所有文件的基本名（二进制日志文件包含了多个文件，稍后会介绍到）。

log-bin-index
: 二进制索引文件的文件名。索引文件中保存了所有binlog文件的列表。

server-id
: 每一个服务器都有一个唯一的ID，所以当一个Slave连接上了Master，并且它的server-id参数值是和Master相同，会产生Master和Slave服务器ID相同的错误。因此我们要保证每台服务器配置的server-id是唯一的。

修改Master配置文件后，重启Master，使配置生效。

###创建复制用户###

最后我们需要在Master节点上新建一个复制用户，并赋以适当的权限。

```
master> CREATE USER repl_user;
query OK, 0 rows affected (0.00 sec)
master> GRANT REPLICATION SLAVE ON *.* TO repl_user IDENTIFIED BY 'password';
query OK, 0 rows affected (0.00 sec)
```

经过以上的步骤，我们就完成了一个Master节点的配置。

##配置Slave##

与Master一样，需要为每个Slave分配一个唯一的服务器ID。使用relay-log和relay-bin-log选项向 *my.cnf* 文件中添加relay log file（中继日志文件）和relay log index file（中继日志索引文件）的文件名。下面的配置为Slave的 *my.cnf* 文件片段。

```
[mysqld]
user             = mysql
pid-file         = /var/run/mysqld/mysql.pid
socket           = /var/run/mysqld/mysql.sock
port             = 3306
basedir          = /usr
datadir          = /var/lib/mysql
tmpdir           = /tmp
relay-log        = slave-relay-bin
relay-log-index  = slave-relay-bin.index
server-id        = 2
```

修改 *my.cnf* 文件后，重启Slave使配置生效。

##连接Master和Slave##

这是创建基本的复制的最后一步，将Slave连接到Master，让Slave知道从哪进行复制。因此需要知道Master的4个信息：

- 主机名/IP
- 端口号
- Master上有用REPLICATION SLAVE权限的账号
- 该账号的密码

最后我们可以使用两个命令来创建和使用复制，使用CHANGE MASTER TO命令将Slave指向Master，然后使用START SLAVE命令来启动复制。

假设我们上面配置的Master的IP是192.168.1.100。

```
slave> CHANGE MASTER TO
    ->    MASTER_HOST = '192.168.1.100',
    ->    MASTER_PORT = '3306',
    ->    MASTER_USER = 'repl_user',
    ->    MASTER_PASSWORD = 'password';
Query OK, 0 rows affected (0.00sec)
slave> START SLAVE;
Query OK, 0 rows affected (0.15sec)
```

恭喜！你已经创建了Master和Slave之间的第一个复制，现在可以尝试在Master的数据库上做一些变动，比如创建表并填充数据，将发现这些改变都会复制到Slave。

##二进制日志简介##

binlog(二进制日志)它记录服务器数据库上的所有改变。语句执行结束时，将会在二进制日志末尾写入一条记录，并通知语句解析器语句已执行完毕。通常只有即将执行完毕的语句才会被写入，但在一些特殊情况下其他信息也会被写入。目前我们只需假设只有执行的语句才会被写入二进制文件。

###二进制日志记录了什么###

二进制日志的目的是记录数据库中表的更改，二进制日志只包括数据库的改动，对于不改变数据的语句则不会写入二进制日志。

####基于行的复制和基于语句的复制####

- 基于语句的复制

Mysql复制记录了产生变化的SQL语句，成为基于语句的复制（*statement-based replication*）。基于语句的复制的缺点在于无法保证所有的语句都能被正确的复制。

- 基于行的复制

在Mysql5.1版本开始提供基于行的复制（*row-based replication*）。基于行的复制每一次的改动记录为二进制日志中的一行。因此基于行的复制会更加的方便，而且有时会更加的快速。

####选择什么复制方式####

关于复制方式的选择，没有特定最好的选择，只有最适合的选择，可以参照下面一段场景描述：

> 一个含有多表链接或者有着很复杂的``WHERE``查询条件的更新，你可能真正关心的是更新后每行的状态，而不是把所有的逻辑都在Slave上执行一遍。相反的，如果某个更新改变了N多行的记录，那么你宁可只记录这个语句，而不是去记录这N多个的单独改动。

###复制的动作###

我们使用前面建立的复制的例子来看一些简单语句的二进制日志事件，先用mysql client连接上Master，然后再通过一些命令来获取二进制日志内容：

```
master> CREATE TABLE test (text TEXT);
Query OK, 0 rows affected (0.03sec)

master> INSERT INTO test VALUES ("哈喽！我来测试复制！");
Query OK, 1 row affected (0.00sec)

master> SELECT * FROM test;
+----------------------+
|text                  |
+----------------------+
|哈喽！我来测试复制！     |
+----------------------+
1 row in set (0.00sec)

master> FLUSH LOGS;
Query OK, 0 rows affected (0.24sec)

```

```FLUSH LOGS```命令强制轮转二进制日志，从而得到一个“完整”的二进制日志文件。使用```SHOW BINLOG EVENTS```命令查看该文件。

```
master> SHOW BINLOG EVENTS\G
************************** 1.row ***************************
   Log_name: master-bin.000001
        Pos: 4
 Event_type: Format_desc
  Server_id: 1
End_log_pos: 106
       Info: Server ver: 5.5.24-log, Binlog ver: 4
************************** 2.row ***************************
   Log_name: master-bin.000001
        Pos: 106
 Event_type: Query
  Server_id: 1
End_log_pos: 197
       Info: use `test`; CREATE TABLE test (text TEXT)
************************** 3.row ***************************
   Log_name: master-bin.000001
        Pos: 197
 Event_type: Query
  Server_id: 1
End_log_pos: 305
       Info: use `test`; INSERT INTO test VALUES ("哈喽！我来测试复制！")
************************** 4.row ***************************
   Log_name: master-bin.000001
        Pos: 305
 Event_type: Rotate
  Server_id: 1
End_log_pos: 349
       Info: master-bin.000002;pos=4
4 rows in set (0.03 sec)
```

上面这个二进制日志文件中包含了4个事件：一个格式描述事件（Format_desc）、两个查询事件（Query）和一个日志轮转（Rotate）事件。查询事件是将数据库上执行的更新语句写入二进制日志文件，而格式描述事件和日志轮转事件则用户服务器内部对二进制日志文件的管理。后面会详细介绍这些事件。

下面我们简单介绍一下每个事件中所有包含的一些字段的意义：

- Event_type
> 这是事件的类型。事件类型是给Slave传递信息的基本方法。

- Server_id
> 这是创建事件的服务器的ID。

- Log_name
> 这是用来存储事件的文件名。一个事件只能存储在一个文件中，不能跨文件存储。

- Pos
> 这是事件在文件中的开始位置，即事件的第一个字节。

- End_log_pos
> 这是事件在文件中的结束位置，也是下一个事件的开始位置。这个位置比事件的最后一个字节高一位，因此事件的字节范围是Pos~End_log_pos-1。通过End_log_pos-Pos可以计算出事件的长度。

- Info
> 这是事件的刻度文本信息，不同的事件显示的信息不同。

每个事件除了以上的字段外，还包含很多信息，比如时间戳等。
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

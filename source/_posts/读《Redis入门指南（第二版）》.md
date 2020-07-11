---
title: 读《Redis入门指南（第二版）》
date: 2020-02-15 08:54:41
tags:
---
Redis是一个开源的、高性能的、基于键值对的缓存与存储系统，通过提供多种键值数据类型来适应不同场景下的缓存与存储需求。

Redis是REmoteDIctionaryServer（远程字典服务器）的缩写，它以字典结构存储数据，并允许其他应用通过TCP协议读写字典中的内容。

到目前为止Redis支持的键值数据类型如下：

- 字符串类型
- 散列类型
- 列表类型
- 集合类型
- 有序集合类型

Redis虽然是作为数据库开发的，但由于其提供了丰富的功能，越来越多的人将其用作缓存、队列系统等。

<!--more-->

讨论关于Redis和Memcached优劣的讨论一直是一个热门的话题。
在性能上Redis是单线程模型，而Memcached支持多线程，所以在多核服务器上后者的性能理论上相对更高一些。
然而，前面已经介绍过，Redis的性能已经足够优异，在绝大部分场合下其性能都不会成为瓶颈，所以在使用时更应该关心的是二者在功能上的区别。
随着Redis3.0的推出，标志着Memcached几乎所有功能都成为了Redis的子集。
同时，Redis对集群的支持使得Memcached原有的第三方集群工具不再成为优势。
因此，在新项目中使用Redis代替Memcached将会是非常好的选择。

## 启动和停止Redis

Redis可执行文件说明

| 文件名  | 说明 |
|---|---|
| redis-server  |  Redis服务器         |
| redis-cli     |  Redis命令行客户端 |
| redis-benchmark  |  Redis性能测试工具 |
| redis-check-aof  |  AOF文件修复工具   |
| redis-check-dump  |  RDB文件检查工具|
| redis-sentinel |  Sentinel服务器 |


### Redis启动方式

1. 直接启动

```shell
redis-server
redis-server --port 6380
```

2. 通过初始化脚本启动

在Redis源代码目录的utils文件夹中有一个名为 [redis_init_script](https://github.com/antirez/redis/blob/unstable/utils/redis_init_script) 的初始化脚本文件，内容如下：

```shell
#!/bin/sh
#
# Simple Redis init.d script conceived to work on Linux systems
# as it does use of the /proc filesystem.

### BEGIN INIT INFO
# Provides:     redis_6379
# Default-Start:        2 3 4 5
# Default-Stop:         0 1 6
# Short-Description:    Redis data structure server
# Description:          Redis data structure server. See https://redis.io
### END INIT INFO

REDISPORT=6379
EXEC=/usr/local/bin/redis-server
CLIEXEC=/usr/local/bin/redis-cli

PIDFILE=/var/run/redis_${REDISPORT}.pid
CONF="/etc/redis/${REDISPORT}.conf"

case "$1" in
    start)
        if [ -f $PIDFILE ]
        then
                echo "$PIDFILE exists, process is already running or crashed"
        else
                echo "Starting Redis server..."
                $EXEC $CONF
        fi
        ;;
    stop)
        if [ ! -f $PIDFILE ]
        then
                echo "$PIDFILE does not exist, process is not running"
        else
                PID=$(cat $PIDFILE)
                echo "Stopping ..."
                $CLIEXEC -p $REDISPORT shutdown
                while [ -x /proc/${PID} ]
                do
                    echo "Waiting for Redis to shutdown ..."
                    sleep 1
                done
                echo "Redis stopped"
        fi
        ;;
    *)
        echo "Please use start or stop as first argument"
        ;;
esac
```

**（1）配置初始化脚本。**

首先将初始化脚本复制到/etc/init.d目录中，文件名为redis_端口号，其中端口号表示要让Redis监听的端口号，客户端通过该端口连接Redis。
然后修改脚本第6行的REDISPORT变量的值为同样的端口号。

**（2）建立需要的文件夹。**

建立表2-2中列出的目录。

| 目录名  | 说明 |
|---|---|
| /etc/redis  |存放Redis的配置文件|
| /var/redis/端口号 | 存放Redis的持久化文件 | 

**（3）修改配置文件。**

首先将配置文件模板（见2.4节介绍）复制到/etc/redis目录中，以端口号命名（如“6379.conf”），然后按照表23对其中的部分参数进行编辑。
表23需要修改的配置及说明

| 参数 | 值  | 说明 |
|---|---|---|
| daemonize  | yes | 使Redis以守护进程模式运行 | 
| pidfile | /var/run/redis_端口号.pid| 设置Redis的PID文件位置| 
| port | 端口号 | 设置Redis监听的端口号 | 
|dir | /var/redis/端口号| 设置持久化文件存放位置|

﻿现在就可以使用/etc/init.d/redis_端口号start来启动Redis了，而后需要执行下面的命令使Redis随系统自动启动：

```shell
$sudo updaterc.dredis_端口号defaults
```

### 停止Redis

考虑到Redis有可能正在将内存中的数据同步到硬盘中，强行终止Redis进程可能会导致数据丢失。
正确停止Redis的方式应该是向Redis发送SHUTDOWN命令，方法为：

```shell
$rediscli
>SHUTDOWN
```

当Redis收到SHUTDOWN命令后，会先断开所有客户端连接，然后根据配置执行持久化，最后完成退出。
Redis可以妥善处理SIGTERM信号，所以使用kill Redis进程的PID也可以正常结束Redis，效果与发送SHUTDOWN命令一样。

##﻿Redis命令行客户端
  
### 命令返回值

1. 状态回复

状态回复（status reply）是最简单的一种回复，比如向Redis发送SET命令设置某个键的值时，Redis会回复状态OK表示设置成功。
另外之前演示的对PING命令的回复PONG也是状态回复。状态回复直接显示状态信息，如：

```shell
redis>PING
PONG
```

2. 错误回复

当出现命令不存在或命令格式有错误等情况时Redis会返回错误回复（error reply）。
错误回复以(error)开头，并在后面跟上错误信息。
如执行一个不存在的命令：

```shell
redis>ERRORCOMMEND
(error)ERR unknown command 'ERRORCOMMEND'
```
在2.6版本时，错误信息均是以“ERR”开头，而在2.8版以后，部分错误信息会以具体的错误类型开头，如：

```shell
redis>LPUSH key 1
(integer) 1
redis>GET key
(error)WRONGTYPE Operation against a key holding the wrong kind of value
```
这里错误信息开头的“WRONGTYPE”就表示类型错误，这个改进使得在调试时能更容易地知道遇到的是哪种类型的错误。

3. 整数回复

Redis虽然没有整数类型，但是却提供了一些用于整数操作的命令，如递增键值的INCR命令会以整数形式返回递增后的键值。
除此之外，一些其他命令也会返回整数，如可以获取当前数据库中键的数量的DBSIZE命令等。
整数回复（integer reply）以(integer)开头，并在后面跟上整数数据：

```shell
redis>INCR foo
(integer) 1
```

4. 字符串回复

字符串回复（bulk reply）是最常见的一种回复类型，当请求一个字符串类型键的键值或一个其他类型键中的某个元素时就会得到一个字符串回复。
字符串回复以双引号包裹：

```shell
redis>GET foo 
"1"
```

特殊情况是当请求的键值不存在时会得到一个空结果，显示为(nil)。如：

```shell
redis>GET noexists
(nil)
```

5. 多行字符串回复

多行字符串回复（multi bulk reply）同样很常见，如当请求一个非字符串类型键的元素列表时就会收到多行字符串回复。
多行字符串回复中的每行字符串都以一个序号开头，如：

```shell
redis>KEYS * 
1) "bar"
2) "foo"
```
提示KEYS命令的作用是获取数据库中符合指定规则的键名，由于读者的Redis中还没有存储数据，所以得到的返回值应该是（empty list or set）。
3.1节会具体介绍KEYS命令，此处读者只需了解多行字符串回复的格式即可。

## 配置

启用配置文件的方法是在启动时将配置文件的路径作为启动参数传递给redis-server，如：

```shell
$redis-server /path/to/redis.conf
```
通过启动参数传递同名的配置选项会覆盖配置文件中相应的参数，就像这样：

```shell
$redis-server /path/to/redis.conf --loglevel warning
```
Redis提供了一个配置文件的模板redis.conf，位于源代码目录的根目录中。
除此之外还可以在Redis运行时通过CONFIGSET命令在不重新启动Redis的情况下动态修改部分Redis配置。就像这样：

```shell
redis> CONFIG SET loglevel warning
OK
```
并不是所有的配置都可以使用`CONFIG SET`命令修改，附录B列出了哪些配置能够使用该命令修改。
同样在运行的时候也可以使用CONFIGGET命令获得Redis当前的配置情况，如：

```shell 
redis> CONFIG GET loglevel
1) "loglevel"
2) "warning"
```
其中第一行字符串回复表示的是选项名，第二行即是选项值。

## 多数据库

每个数据库对外都是以一个从0开始的递增数字命名，Redis默认支持16个数据库，可以通过配置参数databases来修改这一数字。
客户端与Redis建立连接后会自动选择0号数据库，不过可以随时使用SELECT命令更换数据库，如要选择1号数据库：

```shell
redis> SELECT 1
OK
redis[1]> GET foo 
(nil)
```

然而这些以数字命名的数据库又与我们理解的数据库有所区别。
首先Redis不支持自定义数据库的名字，每个数据库都以编号命名，开发者必须自己记录哪些数据库存储了哪些数据。
另外Redis也不支持为每个数据库设置不同的访问密码，所以一个客户端要么可以访问全部数据库，要么连一个数据库也没有权限访问。
最重要的一点是多个数据库之间并不是完全隔离的，比如FLUSHALL命令可以清空一个Redis实例中所有数据库中的数据。
**综上所述，这些数据库更像是一种命名空间，而不适宜存储不同应用程序的数据。**
比如可以使用0号数据库存储某个应用生产环境中的数据，使用1号数据库存储测试环境中的数据，但不适宜使用0号数据库存储A应用的数据而使用1号数据库存储B应用的数据，不同的应用应该使用不同的Redis实例存储数据。
由于Redis非常轻量级，一个空Redis实例占用的内存只有1MB左右，所以不用担心多个Redis实例会额外占用很多内存。

## 数据类型

### 基础命令

1. 获得符合规则的键名列表

```shell
KEYS pattern
```

pattern支持glob风格通配符格式，具体规则如表31所示。

| 符号  | 含义 | 
|---|---|
| ? | 匹配一个字符  |
| * | 匹配任意个（包括0个）字符  |
| [] | 匹配括号间的任意字符，可以使用“-”符号表示一个范围，如a[b-d]可以匹配“ab”、“ac”和“ad”  |
|\x | 匹配字符x，用于转义符号。如需要匹配?就需要使用\?  |

注意KEYS命令需要遍历Redis中的所有键，当键的数量较多时会影响性能，不建议在生产环境中使用。

提示Redis不区分命令大小写，但在本书中均会使用大写字母表示Redis命令。

2. 判断一个键是否存在

```shell

EXISTS key
```

如果键存在则返回整数类型1，否则返回0。例如：

```shell

redis> EXISTS bar
(integer) 1
redis> EXISTS noexists
(integer)0
```

3. 删除键

```shell
DEL key [key…]
```

可以删除一个或多个键，返回值是删除的键的个数。例如：

```shell
redis> DEL bar
(integer) 1
redis> DEL bar
(integer) 0
```
第二次执行DEL命令时因为bar键已经被删除了，实际上并没有删除任何键，所以返回0。

技巧`DEL`命令的参数不支持通配符，但我们可以结合Linux的管道和xargs命令自己实现删除所有符合规则的键。
比如要删除所有以“user:”开头的键，就可以执行

```shell
redis-cli KEYS "user:*" | xargs redis-cli DEL
```

另外由于DEL命令支持多个键作为参数，所以还可以执行
```shell
redis-cli DEL `redis-cli KEYS "user:*"`
```
来达到同样的效果，但是性能更好。

4. 获得键值的数据类型

```shell 
TYPE key
```

TYPE命令用来获得键值的数据类型，返回值可能是string（字符串类型）、hash（散列类型）、list（列表类型）、set（集合类型）、zset（有序集合类型）。例如：

```shell
redis> SET foo 1
OK
redis> TYPE foo
string
redis> LPUSH bar 1
(integer) 1
redis> TYPE bar 
list
```
LPUSH命令的作用是向指定的列表类型键中增加一个元素，如果键不存在则创建它，3.4节会详细介绍。

### 字符串类型

字符串类型是Redis中最基本的数据类型，它能存储任何形式的字符串，包括二进制数据。
你可以用其存储用户的邮箱、JSON化的对象甚至是一张图片。
一个字符串类型键允许存储的数据的最大容量是512MB。

1. 赋值与取值

```shell
SET key value
GET key 
```

SET和GET是Redis中最简单的两个命令，它们实现的功能和编程语言中的读写变量相似，如key="hello"在Redis中是这样表示的：

```shell
redis> SET key hello
OK
```

想要读取键值则更简单：

```shell
redis> GET key
"hello"
```
当键不存在时会返回空结果。

2. 递增数字

```shell 
INCR key
```

前面说过字符串类型可以存储任何形式的字符串，当存储的字符串是整数形式时，Redis提供了一个实用的命令INCR，其作用是让当前键值递增，并返回递增后的值，用法为：
```shell
redis> INCR num
(integer) 1 
redis> INCR num
(integer) 2
```
当要操作的键不存在时会默认键值为0，所以第一次递增后的结果是1。
当键值不是整数时Redis会提示错误：
```shell
redis> SET foo lorem
OK
redis> INCR foo
(error) ERR value is not an integer or out of range
```
有些读者会想到可以借助GET和SET两个命令自己实现incr函数。
如果Redis同时只连接了一个客户端，那么上面的代码没有任何问题。
可当同一时间有多个客户端连接到Redis时则有可能出现竞态条件（race condition）。
例如有两个客户端A和B都要执行我们自己实现的incr函数并准备将同一个键的键值递增，当它们恰好同时执行到代码第二行时二者读取到的键值是一样的，
如“5”，而后它们各自将该值递增到“6”并使用SET命令将其赋给原键，结果虽然对键执行了两次递增操作，最终的键值却是“6”而不是预想中的“7”。
**包括INCR在内的所有Redis命令都是原子操作（atomic operation），无论多少个客户端同时连接，都不会出现上述情况。**
之后我们还会介绍利用事务（4.1节）和脚本（第6章）实现自定义的原子操作的方法。

**实践**

1. 文章访问量统计

Redis对于键的命名并没有强制的要求，但比较好的实践是用“对象类型\:对象ID\:对象属性”来命名一个键，如使用键user\:1\:friends来存储ID为1的用户的好友列表，使用一个名为post\:文章ID\:page.view的键来记录文章的访问量。
对于多个单词则推荐使用“.”分隔，一方面是沿用以前的习惯（Redis以前版本的键名不能包含空格等特殊字符），另一方面是在redis-cli中容易输入，无需使用双引号包裹。
另外为了日后维护方便，键的命名一定要有意义，如u\:1\:f的可读性显然不如user\:1\:friends好（虽然采用较短的名称可以节省存储空间，但由于键值的长度往往远远大于键名的长度，所以这部分的节省大部分情况下并不如可读性来得重要）。

2. 生成自增ID

那么怎么为每篇文章生成一个唯一ID呢？在关系数据库中我们通过设置字段属性为`AUTO_INCREMENT`来实现每增加一条记录自动为其生成一个唯一的递增ID的目的，
而在Redis中可以通过另一种模式来实现：
对于每一类对象使用名为 对象类型(复数形式)\:count的键（如users:count）来存储当前类型对象的数量，
每增加一个新对象时都使用INCR命令递增该键的值。INCR命令的返回值既是加入该对象后的当前类型的对象总数，又是该新增对象的ID。

3. 存储文章数据

由于每个字符串类型键只能存储一个字符串，而一篇博客文章是由标题、正文、作者与发布时间等多个元素构成的。
为了存储这些元素，我们需要使用序列化函数（如PHP中的serialize和JavaScript中的JSON.stringify）将它们转换成一个字符串。
除此之外因为字符串类型键可以存储二进制数据，所以也可以使用 [MessagePack](https://msgpack.org/) 进行序列化，速度更快，占用空间也更小。

**命令拾遗**

1. 增加指定的整数

```shell
INCRBY key increment
```
INCRBY命令与INCR命令基本一样，只不过前者可以通过increment参数指定一次增加的数值，如：

```shell
redis> INCRBY bar 2
(integer) 2 
redis> INCRBY bar 3
(integer) 5
```

2. 减少指定的整数

```shell
DECR key
DECRBY key decrement
```

DECR命令与INCR命令用法相同，只不过是让键值递减，例如：
```shell
redis> DECR bar
(integer) 4
redis> DECRBY key 2
(integer) 2
```

3. 增加指定浮点数

```shell
INCRBYFLOAT key increment
```
INCRBYFLOAT命令类似INCRBY命令，差别是前者可以递增一个双精度浮点数，如：
```shell
redis> INCRBYFLOAT bar 2.7
"4.7"
redis> INCRBYFLOAT bar 5E+4
"50004.69999999999999929"
```

4. ﻿向尾部追加值

```shell
APPEND key value
```
APPEND作用是向键值的末尾追加value。如果键不存在则将该键的值设置为value，即相当于SET key value。
返回值是追加后字符串的总长度。如：

```shell
redis> SET key hello
OK
redis> APPEND key "world!"
(integer) 12
```

5. 获取字符串长度

```shell
STRLEN key
```
STRLEN命令返回键值的长度，如果键不存在则返回0。例如：
```shell
redis> STRLEN key 
(integer) 12
redis> SET key 你好
OK
redis> STRLEN key
(integer)6
```
前面提到了字符串类型可以存储二进制数据，所以它可以存储任何编码的字符串。
例子中Redis接收到的是使用UTF8编码的中文，由于“你”和“好”两个字的UTF8编码的长度都是3，所以此例中会返回6。

6. 同时获得/设置多个键值

```shell
MGET key [key…]
MSET key value [key value…]
```
MGET/MSET与GET/SET相似，不过MGET/MSET可以同时获得/设置多个键的键值。例如：

```shell 
redis> MSET key1 v1 key2 v2 key3 v3
OK
redis> GET key2 
"v2"
redis> MGET key1 key3
1) "v1"
2) "v3"
```

7. 位操作

```shell
GETBIT key offset
SETBIT key offset value
BITCOUNT key [start] [end]
BITOP operation destkey key [key…]
```
利用位操作命令可以非常紧凑地存储布尔值。
比如如果网站的每个用户都有一个递增的整数ID，如果使用一个字符串类型键配合位操作来记录每个用户的性别（用户ID作为索引，二进制位值1和0表示男性和女性），那么记录100万个用户的性别只需占用100KB多的空间，而且由于GETBIT和SETBIT的时间复杂度都是O(1)，所以读取二进制位值性能很高。

### 散列类型

我们现在已经知道Redis是采用字典结构以键值对的形式存储数据的，而散列类型（hash）的键值也是一种字典结构，其存储了字段（field）和字段值的映射，但字段值只能是字符串，不支持其他数据类型，换句话说，散列类型不能嵌套其他的数据类型。
一个散列类型键可以包含至多232−1个字段。
除了散列类型，Redis的其他数据类型同样不支持数据类型嵌套。
比如集合类型的每个元素都只能是字符串，不能是另一个集合或散列表等。
散列类型适合存储对象：使用对象类别和ID构成键名，使用字段表示对象的属性，而字段值则存储属性值。

1. 赋值与取值

```shell
HSET key field value
HGET key field
HMSET key field value [fieldvalue…]
HMGET key field [field…]
HGET ALL key
```

HSET命令用来给字段赋值，而HGET命令用来获得字段的值。用法如下：
```shell
redis> HSET car price 500
(integer) 1
redis> HSET car name BMW
(integer) 1
redis> HGET car name
"BMW"
```

HSET命令的方便之处在于不区分插入和更新操作，修改数据时不用事先判断字段是否存在来决定要执行的是插入操作（update）还是更新操作（insert）。
当执行的是插入操作时（即之前字段不存在）HSET命令会返回1，当执行的是更新操作时（即之前字段已经存在）HSET命令会返回0。

设置多个字段的值时，可以使用HMSET命令。

```shell
HMSET key field1 value1 field2 value2
```

相应地，HMGET命令可以同时获得多个字段的值：
```shell
redis> HMGET car price name
1) "500"
2) "BMW"
```

如果想获取键中所有字段和字段值却不知道键中有哪些字段时，应该使用HGETALL命令。如：
```shell
redis> HGETALL car
1) "price"
2) "500"
3) "name"
4) "BMW"
```

2．判断字段是否存在

```shell
HEXISTS key field 
```

HEXISTS命令用来判断一个字段是否存在。如果存在则返回1，否则返回0（如果键不存在也会返回0）。
```shell
redis> HEXISTS car model
(integer) 0
redis> HSET car model C200
(integer) 1
redis> HEXISTS car model
(integer) 1
```

3．当字段不存在时赋值

```shell
HSETNX key field value
```

HSETNX命令与HSET命令类似，区别在于如果字段已经存在，HSETNX命令将不执行任何操作。HSETNX命令是原子操作，不用担心竞态条件。

4．增加数字

```shell
HINCRBY key field increment
```
上一节的命令拾遗部分介绍了字符串类型的命令INCRBY，HINCRBY命令与之类似，可以使字段值增加指定的整数。
散列类型没有HINCR命令，但是可以通过`HINCRBY key field 1`来实现。HINCRBY命令的示例如下：

```shell
redis> HINCRBY person score 60
(integer) 60
```

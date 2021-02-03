## 6.1.3 High performance unique id generating solutions

Applications usually use the unique id to identify the uniqueness of a table record. A classical solution is using Sequence which is an object of a RDBMS, to use it as the numeric part of an unique identification. TiDB started to support Sequence since v4. But in current Internet age, the applications are facing big challenges: massive data is stored everywhere, data explosion happening at anytime, the applications have to generate UIDs for a huge number of transactions concurrently. Even with the support of Sequence, a RDBMS will encounter a performce bottlenect of Sequence generating.

This document will introduce 2 kinds of high performance UID generating solutions.

### Solution 1: the snowflake UID generators
Snowflake is a distributed UID generating solution introduced by Twitter, it has several kind of implementations, Baidu uid-generator and Meituan leaf are 2 of the popular implementations. In this section the Baidu uid-generator will be introduced.

uid-generator generates 64 bit IDs, an ID is constituted by:

![uid-generator.png](/res/session4/chapter6/serial-number/uid-generator.png)

** Snowflake algorithm：** An unique id consists of worker node, timestamp and sequence within that timestamp. Usually, it is a 64 bits number(long), and the default bits of that three fields are as follows:

sign(1bit)
The highest bit is always 0.

delta seconds (28 bits)
The next 28 bits, represents delta seconds since a customer epoch(2016-05-20). The maximum time will be 8.7 years.

worker id (22 bits)
The next 22 bits, represents the worker node id, maximum value will be 4.2 million. UidGenerator uses a build-in database based worker id assigner when startup by default, and it will dispose previous work node id after reboot. Other strategy such like 'reuse' is coming soon.

sequence (13 bits)
the last 13 bits, represents sequence within the one second, maximum is 8192 per second by default.

Below items are needed to pay attention when using a snowflake UID generator:
* the delta seconds is generated in the local machine, it relies on that machine's local clock. If a turn back of the clock happens, it leads to UID duplications or even the UID generating service outage.
* the bits of the delta seconds can be adapted by considering the data TTL, usually it's between 28 bits to 44 bits.
* do not use the default value for the delta seconds' customer epoch, it should close to the current time.
* the bit of a worker node id is limited, the number can not be greater than 5 million. If the auto increment column is used to generate the worker node id, every restar of the tidb-server will make the auto increment id gain 30,000 at least, in this case, only 500 /3 = 166 times of the tib-server restart leads to a fault that the generated value is large than the worker node id could store. A workaround is to truncate the table of the auto increment id of worker node id generating, rebase the value of the auto increment column's value to 0. This problem can be resolved in UID-generator either.


## 6.1.3 高并发的唯一序列号生成方案

应用程序通常使用唯一标识符来确认一行记录。传统解决方案普遍依赖数据库的序列对象（Sequence）来生成唯一标识符中的数字部分。TiDB 4.0 正式支持序列。但是，互联网应用需要处理的数据量巨大且往往呈现爆发式增长，使得应用程序必须在短时间内为大量数据和消息生成唯一标识符。这种场景下，即使有了序列功能，数据库也可能会由于高并发的序列分配请求而出现性能瓶颈。

本节将介绍两种高性能的序列号生成方案。

### 方案一：类 Snowflake 方案
Snowflake 是 Twitter 提出的分布式 ID 生成方案。目前有多种实现，较流行的是百度的 uid-generator 和美团的 leaf。下面以 uid-generator 为例展开说明。

uid-generator 生成的 64 位 ID 结构如下

![uid-generator.png](/res/session4/chapter6/serial-number/uid-generator.png)

* sign：长度固定为 1 位。固定为 0，表示生成的 ID 始终为正数。
* delta seconds：默认 28 位。当前时间，表示为相对于某个预设时间基点 (默认 "2016-05-20") 的增量值，单位为秒。28 位最多可支持约 8.7 年。
* worker node id：默认 22 位。表示机器 id，通常在应用程序进程启动时从一个集中式的 ID 生成器取得。常见的集中式 ID 生成器是数据库自增列或者 Zookeeper。默认分配策略为用后即弃，进程重启时会重新获取一个新的 worker node id，22 位最多可支持约 420 万次启动。
* sequence：默认 13 位。表示每秒的并发序列，13 位可支持每秒 8192 个并发。

使用类 Snowflake 方案时需要注意几个问题：
* delta seconds 完全本地生成，强依赖机器时钟。如果发生时钟回拨, 会导致发号重复或者服务会处于不可用状态。
* 可根据数据预期寿命调整 delta seconds 位数, 一般在 28 位至 44 位之间。
* delta seconds 时间基点不要使用默认值，应该尽量贴近当前时间。
* worker node id 位数有限，对应数值不超过 500 万。 如果使用 TiDB 的自增列实现 worker node id，每次 TiDB 实例的重启都会让自增列返回值增加至少 3 万，这样最多 500 / 3 = 166 次实例重启后，自增列返回值就比 worker node id 可接受的最大值要大。这时就不能直接使用这个过大的值，需要清空自增列所在的表，把自增列值重置为零，也可以在 Snowflake 实现层解决这个问题。

### 方案二：号段分配方案
号段分配方案可以理解为从数据库批量获取自增 ID。本方案需要一张序列号生成表，每行记录表示一个序列对象。表定义示例如下：
| 字段名 | 字段类型 | 字段说明 |
| :---- | :------ | :------ |
| SEQ_NAME | varchar(128) | 序列名称，用来区分不同业务 |
| MAX_ID | bigint(20) | 当前序列已被分配出去的最大值 |
| STEP | int(11) | 步长，表示每次分配的号段长度 |

应用程序每次按配置好的步长获取一段序列号，并同时更新数据库以持久化保存当前序列已被分配出去的最大值，然后在应用程序内存中即可完成序列号加工及分配动作。待一段号码耗尽之后，应用程序才会去获取新的号段，这样就有效降低了数据库写入压力。实际使用过程中，还可以适度调节步长以控制数据库记录的更新频度。
 
最后，需要注意的是，上述两种方案生成的 ID 都不够随机，不适合直接作为 TiDB 表的主键。实际使用过程中可以对生成的ID进行位反转（bit-reverse）后得到一个较为随机的新 ID。

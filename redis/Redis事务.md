title: Redis事务
tag: redis
---
本文为redis学习笔记的第十一篇文章。详细探讨redis事务的用法和原理。
<!-- more -->

redis 事务是一组命令的集合，至少是两个或两个以上的命令，redis 事务保证这些命令被执行时中间不会被任何其他操作打断。

### 事务基本认识

当客户端处于非事务状态下时， 所有发送给服务器端的命令都会立即被服务器执行。

但是， 当客户端进入事务状态之后， 服务器在收到来自客户端的命令时， 不会立即执行命令， 而是将这些命令全部放进一个事务队列里， 然后返回 `QUEUED` ， 表示命令已入队。

![image](http://xiaozhao.oursnail.cn/redis%E4%BA%8B%E5%8A%A1.svg)

### 事务执行

前面说到， 当客户端进入事务状态之后， 客户端发送的命令就会被放进事务队列里。

但其实并不是所有的命令都会被放进事务队列， 其中的例外就是 `EXEC` 、 `DISCARD` 、 `MULTI` 和 `WATCH` 这四个命令 —— 当这四个命令从客户端发送到服务器时， 它们会像客户端处于非事务状态一样， 直接被服务器执行：


![image](http://xiaozhao.oursnail.cn/redis%E6%89%A7%E8%A1%8C%E4%BA%8B%E5%8A%A1.svg)

如果客户端正处于事务状态， 那么当 `EXEC` 命令执行时， 服务器根据客户端所保存的事务队列， 以先进先出（`FIFO`）的方式执行事务队列中的命令： 最先入队的命令最先执行， 而最后入队的命令最后执行。

### 事务基本命令介绍

除了 `EXEC` 之外， 服务器在客户端处于事务状态时， 不加入到事务队列而直接执行的另外三个命令是 `DISCARD` 、 `MULTI` 和 `WATCH` 。

`DISCARD` 命令用于取消一个事务， 它清空客户端的整个事务队列， 然后将客户端从事务状态调整回非事务状态， 最后返回字符串 OK 给客户端， 说明事务已被取消。

`Redis` 的事务是不可嵌套的， 当客户端已经处于事务状态， 而客户端又再向服务器发送 `MULTI` 时， 服务器只是简单地向客户端发送一个错误， 然后继续等待其他命令的入队。 `MULTI` 命令的发送不会造成整个事务失败， 也不会修改事务队列中已有的数据。

`WATCH` 只能在客户端进入事务状态之前执行， 在事务状态下发送 `WATCH` 命令会引发一个错误， 但它不会造成整个事务失败， 也不会修改事务队列中已有的数据（和前面处理 `MULTI` 的情况一样）。

### 正常情况
```
multi//开启事务，下面的命令先不执行，先暂时保存起来
set key val//命令入队
exec//提交事务（执行命令）
```

### 异常情况
```
multi//开启事务，下面的命令先不执行，先暂时保存起来
set key val//正常命令入队
set key//错误命令，直接报错
exec//事务被丢弃，提交失败
```

### 例外情况
```
multi//开启事务，下面的命令先不执行，先暂时保存起来
set key val//正常命令入队
incr key//虽然字符串不能增一，但是不报错，入队
exec//自增会失败，但是key被设置成功了，整个事务没有回滚
```

### 放弃事务
```
multi//开启事务，下面的命令先不执行，先暂时保存起来
set key val//正常命令入队
discard
```


### 乐观锁

乐观锁：每次拿数据的时候都认为别人不会修改该数据，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这条数据，一般使用版本号进行判断，乐观锁使用于读多写少的应用类型，这样可以提高吞吐量。


乐观锁大多情况是根据数据版本号(`version`)的机制实现的，何为数据版本？即为数据增加一个版本标识，在基于数据库表的版本解决方案中，一般是通过为数据库添加一个`version`字段来实现读取出数据时，将此版本号一起读出，之后更新时，对此版本号加1，此时将提交数据的版本号与数据库表对应记录的当前版本号进行比对，如果提交的数据版本号大于数据库表的当前版本，则予以更新，否则认为是过期数据，不予更新。


A | B
---|---
读出版本号为1，操作 | A操作时，读出版本号也为1，进行某个操作(修改)
执行修改，version+1=2，因为2>1，所以更新| ...
...    | 执行修改，version+1=2，发现数据库记录的版本也为2，2=2,更新失败

### watch机制

`WATCH` 命令用于在事务开始之前监视任意数量的键： 当调用 `EXEC` 命令执行事务时， 如果任意一个被监视的键已经被其他客户端修改了， 那么整个事务不再执行， 直接返回失败。


```
set k1 1     //设置k1值为1
watch k1     //监视k1(其他客户端不能修改k1值)
set k1 2     //设置k1值为2
multi        //开始事务
set k1 3     //修改k1值为3
exex         //提交事务，k1值仍为2，因为事务开始之前k1值被修改了
```

### watch机制举例

大家可能知道`redis`提供了基于`incr`命令来操作一个整数型数值的原子递增，那么我们假设如果`redis`没有这个`incr`命令，我们该怎么实现这个`incr`的操作呢？

正常情况下我们想要对一个整形数值做修改是这么做的(伪代码实现)：


```java
val = GET mykey
val = val + 1
SET mykey $val
```

但是上述的代码会出现一个问题,因为上面吧正常的一个`incr`(原子递增操作)分为了两部分,那么在多线程(分布式)环境中，这个操作就有可能不再具有原子性了。

研究过`java`的`juc`包的人应该都知道`cas`，那么`redis`也提供了这样的一个机制，就是利用`watch`命令来实现的。

具体做法如下:


```java
WATCH mykey
val = GET mykey
val = val + 1
MULTI
SET mykey $val
EXEC
```
和此前代码不同的是，新代码在获取`mykey`的值之前先通过`WATCH`命令监控了该键，此后又将`set`命令包围在事务中，这样就可以有效的保证每个连接在执行`EXEC`之前，如果当前连接获取的`mykey`的值被其它连接的客户端修改，那么当前连接的`EXEC`命令将执行失败。这样调用者在判断返回值后就可以获悉`val`是否被重新设置成功。

由于`WATCH`命令的作用只是当被监控的键值被修改后阻止之后一个事务的执行，而不能保证其他客户端不修改这一键值，所以在一般的情况下我们需要在`EXEC`执行失败后重新执行整个函数。

执行`EXEC`命令后会取消对所有键的监控，如果不想执行事务中的命令也可以使用`UNWATCH`命令来取消监控。

### watch机制原理

#### WATCH 命令的实现

在每个代表数据库的 `redis.h/redisDb` 结构类型中， 都保存了一个 `watched_keys` 字典， 字典的键是这个数据库被监视的键， 而字典的值则是一个链表， 链表中保存了所有监视这个键的客户端。

比如说，以下字典就展示了一个 `watched_keys` 字典的例子：

![image](http://xiaozhao.oursnail.cn/watch%E6%9C%BA%E5%88%B6%E5%8E%9F%E7%90%861.svg)

其中， 键 `key1` 正在被 `client2` 、 `client5` 和 `client1` 三个客户端监视， 其他一些键也分别被其他别的客户端监视着。

`WATCH` 命令的作用， 就是将当前客户端和要监视的键在 `watched_keys` 中进行关联。

举个例子， 如果当前客户端为 `client10086` ， 那么当客户端执行 `WATCH key1 key2` 时， 前面展示的 `watched_keys` 将被修改成这个样子：

![image](http://xiaozhao.oursnail.cn/watch%E6%9C%BA%E5%88%B6%E5%8E%9F%E7%90%862.svg)

通过 `watched_keys` 字典， 如果程序想检查某个键是否被监视， 那么它只要检查字典中是否存在这个键即可； 如果程序要获取监视某个键的所有客户端， 那么只要取出键的值（一个链表）， 然后对链表进行遍历即可。

#### WATCH 的触发

在任何对数据库键空间（`key space`）进行修改的命令成功执行之后 （比如 `FLUSHDB` 、 `SET` 、 `DEL` 、 `LPUSH` 、 `SADD` 、 `ZREM` ，诸如此类）， `multi.c/touchWatchedKey` 函数都会被调用 —— 它检查数据库的 `watched_keys` 字典， 看是否有客户端在监视已经被命令修改的键， 如果有的话， 程序将所有监视这个/这些被修改键的客户端的 `REDIS_DIRTY_CAS` 选项打开：

![image](http://xiaozhao.oursnail.cn/watch%E6%9C%BA%E5%88%B6%E5%8E%9F%E7%90%863.svg)

当客户端发送 `EXEC` 命令、触发事务执行时， 服务器会对客户端的状态进行检查：

- 如果客户端的 `REDIS_DIRTY_CAS` 选项已经被打开，那么说明被客户端监视的键至少有一个已经被修改了，事务的安全性已经被破坏。服务器会放弃执行这个事务，直接向客户端返回空回复，表示事务执行失败。
- 如果 `REDIS_DIRTY_CAS` 选项没有被打开，那么说明所有监视键都安全，服务器正式执行事务。

举个例子，假设数据库的 `watched_keys` 字典如下图所示：

![image](http://xiaozhao.oursnail.cn/watch%E6%9C%BA%E5%88%B6%E5%8E%9F%E7%90%864.svg)

如果某个客户端对 `key1` 进行了修改（比如执行 `DEL key1` ）， 那么所有监视 `key1` 的客户端， 包括 `client2` 、 `client5` 和 `client1` 的 `REDIS_DIRTY_CAS` 选项都会被打开， 当客户端 `client2` 、 `client5` 和 `client1` 执行 `EXEC` 的时候， 它们的事务都会以失败告终。

最后，当一个客户端结束它的事务时，无论事务是成功执行，还是失败， `watched_keys` 字典中和这个客户端相关的资料都会被清除。

### 事务的 ACID 性质

`Redis` 事务保证了其中的一致性（偶尔也有可能不一致）和隔离性，但并不保证原子性和持久性。

#### 原子性（Atomicity）
单个 `Redis` 命令的执行是原子性的，但 `Redis` 没有在事务上增加任何维持原子性的机制，所以 `Redis` 事务的执行并不是原子性的。

如果一个事务队列中的所有命令都被成功地执行，那么称这个事务执行成功。

另一方面，如果 `Redis` 服务器进程在执行事务的过程中被停止 —— 比如接到 `KILL` 信号、宿主机器停机，等等，那么事务执行失败。

当事务失败时，`Redis` 也不会进行任何的重试或者回滚动作。

#### 一致性（Consistency）
`Redis` 的一致性问题可以分为三部分来讨论：入队错误、执行错误、`Redis` 进程被终结。

前面两者上面已经讨论过了，这里再重复一下.

- 入队错误

入队错误一般是错误的命令(不考虑能不能执行，命令本身就是错误的)，带有不正确入队命令的事务不会被执行，也不会影响数据库的一致性；

- 执行错误

如果命令在事务执行的过程中发生错误，比如说，对一个不同类型的 `key` 执行了错误的操作， 那么 `Redis` 只会将错误包含在事务的结果中， 这不会引起事务中断或整个失败，不会影响已执行事务命令的结果，也不会影响后面要执行的事务命令， 所以它对事务的一致性也没有影响。


- `Redis` 进程被终结

如果 `Redis` 服务器进程在执行事务的过程中被其他进程终结，或者被管理员强制杀死，那么根据 `Redis` 所使用的持久化模式，可能有以下情况出现：

> 内存模式：如果 Redis 没有采取任何持久化机制，那么重启之后的数据库总是空白的，所以数据总是一致的。

> RDB 模式：在执行事务时，Redis 不会中断事务去执行保存 RDB 的工作，只有在事务执行之后，保存 RDB 的工作才有可能开始。所以当 RDB 模式下的 Redis 服务器进程在事务中途被杀死时，事务内执行的命令，不管成功了多少，都不会被保存到 RDB 文件里。所以显然会造成不一致

> AOF 模式：因为保存 AOF 文件的工作在后台线程进行，所以即使是在事务执行的中途，保存 AOF 文件的工作也可以继续进行,如果事务语句未写入到 AOF 文件，那么显然是一致的，因为事务里的操作全部失败；如果事务的部分语句被写入到 AOF 文件，并且 AOF 文件被成功保存，那么不完整的事务执行信息就会遗留在 AOF 文件里，当重启 Redis 时，程序会检测到 AOF 文件并不完整，Redis 会退出，并报告错误。需要使用 redis-check-aof 工具将部分成功的事务命令移除之后，才能再次启动服务器。还原之后的数据总是一致的，而且数据也是最新的（直到事务执行之前为止）。

#### 隔离性（Isolation）
`Redis` 是单进程程序，并且它保证在执行事务时，不会对事务进行中断，事务可以运行直到执行完所有事务队列中的命令为止。因此，`Redis` 的事务是总是带有隔离性的。

#### 持久性（Durability）

- 在单纯的内存模式下，事务肯定是不持久的。
- 在 `RDB` 模式下，服务器可能在事务执行之后、`RDB` 文件更新之前的这段时间宕机，所以 `RDB` 模式下的 `Redis` 事务也是不持久的。
- 在 `AOF` 的“总是 `SYNC` ”模式下，事务的每条命令在执行成功之后，都会立即调用 `fsync` 或 `fdatasync` 将事务数据写入到 `AOF` 文件。但是，这种保存是由后台线程进行的，主线程不会阻塞直到保存成功，所以从命令执行成功到数据保存到硬盘之间，还是有一段非常小的间隔，服务器也有可能出现问题，所以这种模式下的事务也是不持久的。
- 都是不持久的。

### 总结

- `MULTI` 命令的执行标记着事务的开始
-  当客户端进入事务状态之后， 服务器在收到来自客户端的命令时， 不会立即执行命令， 而是将这些命令全部放进一个事务队列里， 然后返回 `QUEUED` ， 表示命令已入队
-  `Redis` 的事务保证了 `ACID` 中的一致性（C）（偶尔也有可能不一致）和隔离性（I），但并不保证原子性（A）和持久性（D）。
-  不加入到事务队列而直接执行的四个命令为：`EXEC` 、 `DISCARD` 、 `MULTI` 和 `WATCH`
-  `DISCARD` 命令用于取消一个事务
-  `Redis` 的事务是不可嵌套的
-  `WATCH` 只能在客户端进入事务状态之前执行
-  `WATCH`机制的原理

参考：

- [事务](http://redisbook.readthedocs.io/en/latest/feature/transaction.html)
- [redis的事务和watch](https://www.jianshu.com/p/361cb9cd13d5)
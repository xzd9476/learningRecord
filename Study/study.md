[TOC]



# 第一章：MySQL

### 1、客户端——服务器交互流程

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/客_服交互流程.png" style="zoom:80%;" />

### 2、存储引擎：表处理器

- 连接管理、缓存、语法解析、查询优化，划分为**MySQL Server**的功能；


- 数据的存储形式、读写操作，划分为**存储引擎**的功能。


|      | MyISAM           | InnoDB | Memory |
| :--: | ---------------- | ------ | ------ |
|      | //todo第二节内容 |        |        |

### 3、索引分类

- **聚簇索引**：被索引的列必须是主键列，没有主键会指定一个唯一的为空键代替，聚簇索引是根据主键来创建的一颗B+树，叶子节点里面保存的是实际的一行数据；
- **辅助索引**：InnoDB引擎对于非主键列建立的索引，叶子节点中保存的是主键值，根据这个主键值再进行回表查找聚簇索引。
- **非聚簇索引**：MyISAM的索引结构，MyISAM引擎将索引和数据分开存储。叶子节点保存的是索引键的值和实际数据的内存地址。MyISAM把实际数据以表格形式存在一起，按照插入顺序排列。通过非聚簇索引找到数据表的行，在定位到实际的行读取数据。
- **覆盖索引**：用于建立联合索引的列，覆盖了需要查询的列，无需回表
  如：建立 name + sex + age 的索引
        slect name,sex,age from stu where name ='xzd'；是覆盖索引
        select name,birthday from stu where name = 'xzd'；不是覆盖索引


### 4、字符集与字符编码

- **字符集**：所有抽象字符的集，不同的字符集收录的字符不一样，ASCII字符集只有128个字符，且没有收录中文，unicode字符集收录的字符比较全
- **字符编码**：建立字符(key)与二进制值(value)的映射关系，通过字符编码规则，可以将字符转换成二进制，保存在系统中，常用的字符编码：utf8是unicode字符集的一种编码方式，utf8使用1~4字节表示一个字符。

### 5、InnoDB行记录结构

1. **COMPACT行格式**
   <img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/Compact行格式.png" style="zoom:80%;" />

   - 记录头信息包含：delete_mask，标记记录是否被删除；n_owned，标记当前记录拥有的记录数；next_record，标记下一条记录的位置
   - 真实数据包含：row_id，当建表没有指定主键和唯一键时，会自动添加一个隐藏主键row_id作为主键用于建立索引

2. **Redundant行格式**
   <img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/Redundant行格式.png" style="zoom:80%;" />

   

### 6、InnoDB数据页结构

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/数据页结构.png" style="zoom: 43%;" />

- **File Header**：页面通有信息，如：校验和（用于与Trailer的校验和对比）、上下页页号
- **Page Header**：页面专有信息，如：有多少记录，有多少槽；
- **Infimum+supremum**：最小记录和最大记录
- **User Records**：真实记录数据
- **FreeSpace**：空闲空间
- **Page Directory**：页目录，相当于书籍目录索引的页码表，由槽（表示数据在页中的位置，相当于页码）组成
- **File Trailer**：页尾，校验页是否完整

### 7、使用PageDirectory进行查找的过程

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/PageDirectory理解.png" style="zoom:65%;" />

通过二分法先定位到槽2，对比主键,发现target主键>槽2的最后一个数据，那么继续二分法定位到槽1，对比主键，然后在通过遍历槽内链表结构的数据，定位到具体数据。所以，槽就相当于页码，通过页码找到数据，再看数据是否是符合要求的。

### 8、B+树结构

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/B+树结构.png" style="zoom:70%;" />

- 实际用户记录存在于最底层的叶子节点
- 用来存放目录项的节点称为非叶子节点
- 页内的记录都是按逐渐大小顺序排列成链表
- 页与页之间是双向链表

### 9、连接join的原理

​      首先，将驱动表和被驱动表的记录都取出来，依次匹配的组合加入结果集，形成一个笛卡尔积。
​      **内连接：**把不符合ON条件 或 WHERE条件的记录过滤掉；
​      **左/右连接：**把结果集中，保留符合ON条件或WHERE条件的驱动表的所有记录，对应的被驱动表中的记录若为空，则用NULL补充。
​      对于外连接的驱动表的记录来说，如果无法在被驱动表中找到匹配ON子句中的过滤条件的记录，那么该记录仍然会被加入到结果集中，对应的被驱动表记录的各个字段使用NULL值填充； 

### 10、InnoDB的缓存机制Buffer pool

​       MySQL访问数据以页为基本单位，当需要访问某个页的数据时，就会把完整的页数据加载到内存中，完成访问后不急于把该页对应的内存空间释放掉，而是缓存起来。缓存的位置在内存中被叫做BufferPool。
​      表空间号 + 页号 作为key，缓存页作为value创建一个hash表，从而快速在buffer pool中找到缓存页。如果找不到，那就从free链表中选一个空闲页，再把磁盘中对应的缓存页加载进去。
​      **free链表：**把所有空闲的缓存页对应的控制块作为节点放到一个链表中；
​      **flush链表：**BufferPool中某个缓存页的数据被修改后，就和磁盘上的页不一样了，这样的被叫做脏页。脏页对应的控制块会被作为节点加入到一个链表中，叫做flush链表，意味着将来某个时间点会被刷新到磁盘的。
​      **LRU链表：**BufferPool中缓存页的组织形式。为了保留最经常被访问的缓存也，采用最近最少使用算法淘汰冷数据。
<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/bufferPool.png" style="zoom:80%;" />

### 11、事务ACID

需要保证原子性、隔离性、一致性、持久性的一个或多个数据库操作称之为一个事务
**原子性Atomicity：** 一个事务中的所有操作，要么全部完成，要么全部不完成，不会结束在中间某个环节。 事务过程中出错会回滚到事务开始前。MySQL通过undo日志来保证原子性

**隔离性Isolation：**数据库允许多个事务并发执行，隔离性可以防止多个事务并发时由于交叉执行而导致数据不一致。隔离级别分为读未提交、读已提交、可重复度、串行。

**一致性Consistency：**事务开始前和事务结束后，数据库的完整性没有被破坏。意味着事务操作必须完全符合所有预设规则，如唯一性、非空、两个账户余额总和不变。

**持久性Durability：**对数据库的修改，会被持久化到磁盘中。MySQL通过redo日志来保证持久性。

### 12、事务隔离级别

- **读未提交**：read uncommitted，最低级别，没有并发控制能力。会读到脏数据
- **读已提交**：read committed，访问到的是事务提交后的数据，避免脏数据，Oracle默认隔离级别。通过MVCC实现
- **可重复读**：repeatable read，MySql默认级别。通过MVCC实现
- **串行**：serializable，串行执行，吞吐量较低

### 13、并发下的事务问题

- **脏读：**读了未提交的数据。事务A修改了数据，但未提交，此时事务B来访问，读到了事务A未提交的脏数据。
- **不可重复读：**读了已提交的数据。一个事务还未执行完成，但过程中访问的数据被另一个事务更改：事务A读取数据，此时事务B对这个数据进行修改并提交成功，事务A再来读这个数据，两次读取的数据不一样，重复读的数据不一致。可通过行锁解决
- **幻读：**涉及到插入/删除动作。一个事务还未执行完成，但过程中其他事务进行了插入/删除；事务A第一次访问数据，数据为空（或数据有N条），此时事务B进行插入（或删除）动作，事务A再来读取，发现数据多了（或是少了）。通过行锁解决不了

不可重复读针对update操作，使用行级锁即可解决；幻读针对insert与delete操作，需要使用表级锁。

### 14、事务隔离级别的并发问题

|                  | 脏读   | 不可重复读 | 幻读   |
| ---------------- | ------ | ---------- | ------ |
| Read uncommitted | 存在   | 存在       | 存在   |
| Read committed   | 不存在 | 存在       | 存在   |
| Repeatable read  | 不存在 | 不存在     | 存在   |
| Serializable     | 不存在 | 不存在     | 不存在 |



### 15、redo日志

**概念：**InnoDB中的重做日志，用于实现事务的持久性，日志文件由两部分组成：内存中的redo日志缓存 + 磁盘中的redo日志文件。redo是  `物理日志`，记录的是“在某个数据页上做了什么修改”

**作用：**MySQL为了提升性能，数据的修改不会立即同步到磁盘，而是保存在bufferPool中，为了防止宕机造成数据丢失，引入redo log机制，对数据进行修改前会记录redo日志，在事务commit之前会先把redo日志刷到磁盘中。系统崩溃后，根据磁盘中的redo日志可以恢复最新的数据

### 16、undo日志

**概念：**InnoDB中的回滚日志，就是对用于实现事务原子性和MVCC，undo日志主要记录的是数据的逻辑变化，为了在发生错误时回滚之前的操作， 会将修改之前的操作都记录下来， 然后在发生错误时才可以回滚。 undo日志是 `逻辑日志` ，简单来说就是记录sql语句，如一条Insert语句，对应一条Delete的undo log。

**作用：**记录事务修改之前的数据信息，因此加入由于系统错误或者rollback而回滚，可以根据undo日志的信息来恢复之前的数据状态。

**分类：**
	insert undo log：Insert操作记录没有历史版本，只对当前事务本身可见。在事务提交后直接删除

​	update undo log：Update操作和Delete操作产生的log，delete操作相当于打个标记并没有实际删除。

### 17、undo及redo事务简化过程

假设有A、B两个数据，值分别为1、2
开始一个事务，操作内容为：把1修改为3 ， 2修改为4，那么实际的记录如下：

1.事务开始；

2.undo:记录A = 1；
3.修改A = 3；
4.redo: 记录A = 3；

5.undo: 记录B = 2；
6.修改B = 4；
7.redo: 记录B = 4；

8.写binlog；
9.将redo log写入磁盘；
9.事务提交；

### 18、redo log 和 binlog的区别

|          | redo log                                                 | binlog                                                     |
| -------- | -------------------------------------------------------- | ---------------------------------------------------------- |
| 日志类型 | 物理日志，记录值的变化                                   | 逻辑日志，记录的是sql                                      |
| 文件大小 | redo log大小是固定的                                     | 可通过                                                     |
| 功能所属 | InnoDB引擎层                                             | MySQL Server层                                             |
| 记录方式 | 采用循环写的方式记录，当写到结尾时，会回到开头循环写日志 | 通过追加的方式记录，当文件大小大于给定值后，会创建新的文件 |
| 适用场景 | 用于崩溃恢复 crash-safe                                  | 用于主从复制和数据恢复                                     |

​      由 `binlog` 和 `redo log` 的区别可知：`binlog` 日志只用于归档，只依靠 `binlog` 是没有 `crash-safe` 能力的。但只有 `redo log` 也不行，因为 `redo log` 是 `InnoDB`特有的，且日志上的记录落盘后会被覆盖掉。因此需要 `binlog`和 `redo log`二者同时记录，才能保证当数据库发生宕机重启时，数据不会丢失。

在binlog写入磁盘之前，redo log已经准备好在内存中；
在binlog写入磁盘之后，redo log才开始刷盘；

### 19、MVCC原理

**概念：**
    多版本并发控制Multi-Version Concurrency Control。
    指的就是在使用ReadCommitted、RepeatableRead这两种隔离级别的事务在执行普通SELECT操作（快照读）时，访问记录的版本链的过程。该过程依赖undo log 和视图ReadView。

**原理：**
    通过ReadView来确定数据的可见性，使用undo log来读取旧版本数据。
    每条记录有一个trx_id字段和roll_pointer字段，trx_id记录最近修改了这条数据的事务ID，rool_pointer指向undo日志，用于遍历历史版本的数据。

**作用：**
    1.解决**读-写冲突**的无锁控制（乐观锁解决**写-写冲突**的无锁控制）。每个修改保存一个版本，版本与事务id关联，读操作只读readView开始前的数据库的快照。
    2.实现事务隔离级别RC 、RR。

**ReadView视图:** 
    记录当前活跃的事务，用于确定数据版本的可见性。RC级别和RR级别生成ReadView的时机不一样。 
    包含`m_ids`活跃的事务id列表、`min_trx_id`活跃事务中最小的事务ID、`max_trx_id`应该分配给下一个事务的ID值、`creator_trx_id`生成这个ReadView的事务ID。
<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/ReadView结构.png" style="zoom:80%;" />

**使用ReadView进行快照读的规则：**
      1.被访问数据的trx_id与Read_View的creator_trx_id相同，说明是自己的数据，可见；
      2.被访问数据的trx_id < min_trx_id说明早已经提交，可见；
      3.被访问数据的trx_id >= max_trx_id，说明该事务数据在生成ReadView之后开启的，不可见；
      4.被访问数据的trx_id < max_trx_id 并 trx_id > min_trx_id，判断trx_id是否在m_ids中：
            4.1如果在，说明事务还在活跃，不可见；
            4.2如果不在，说明该版本已提交，可见。

**不同的事务隔离级别生成ReadView的时机：**ReadView的时机决定不同max_trx_id和m_ids列表。
      Read Committed：每次select都会生成一个ReadView。保证每次都读取到最新的已提交数据。
      Repeatable Read：在第一次select时生成ReadView ，整个事务期间只有一个ReadView。查询值承认在事务启动前就已经提交完成的数据。

**快照读&当前读：** 
1.MVCC中的快照读，通过MVCC机制读取快照数据，是属于乐观锁的读取方式。
2.MVCC中的当前读，通过加锁来进行控制，读的是最新版本，并且对读取的记录加锁，阻塞其他事务。
   当前读的情况：select ... lock in share mode;     select ... for update;     update; delete; insert
   当前读操作都是先读后写（读是更新语句执行，不是我们手动执行），读的就是当前版本的值，叫当前读；而我们普通的查询语句就叫快照读。 

### 20、MySQL的锁

**锁的分类：**

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/锁的分类.png" style="zoom:80%;" />

**兼容性：**
	共享锁：也叫S锁、读锁。共享锁与共享锁兼容，与排它锁不兼容
	排它锁：也叫X锁、写锁。与共享锁和排它锁皆不兼容
	意向共享锁、意向排它锁： 为了允许行锁和表锁共存，实现多粒度锁机制 。如果不在**表**上加意向锁，对**表**加锁的时候，都要去检查表中的**某一行**上是否加有行锁，意图锁的主要目的是某个事务正在锁定表中的一行，或者将要锁定表中的一行。 

**锁粒度：**
	表锁：MyISAM只支持表锁。
	行锁：InnoDB行锁是通过给索引上的索引项加锁来实现的，只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁。默认情况下Innodb不会上行锁。可通过语句指定加锁：select... lock in share mode、select...for update。

**加锁机制：**
	乐观锁：一种用来解决**写-写冲突**的无锁并发控制。数据记录会有一个版本号，每次对数据的修改都会版本号加一。事务在提交修改前，会检查版本号是否被改变，如果被改变则重试或回滚。
	悲观锁：排它锁。使用select...for update或者update语句会触发悲观锁。

**锁模式：**
	间隙锁GAP：用范围条件检索数据请求共享或排他锁时，InnoDB会给符合范围条件的已有记录的索引项加锁。
	意向锁：意图锁的主要目的是某个事务正在锁定表中的一行，或者将要锁定表中的一行。事务给某一行加锁的时候，需要先取得相应的意向锁。







# 第二章：计网

### 1、OSI七层模型

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/OSI七层.png" style="zoom:80%;" />



### 2、TCP/IP四层

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/TCPIP四层.png" style="zoom:80%;" />



### 3、数据包结构

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/数据包结构.png" style="zoom:80%;" />



### 4、TCP首部结构

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/TCP首部.png" style="zoom:80%;" />

- 端口号：用来标记应用进程。
- 序列号seq：标识当前传输报文段中的第一个数据字节，随机产生seq并在三握时通过SYN传给接收端。保证TCP传输的有序性。
- 确认应答号ack：即acknumber，表明下一个期待收到的字节序号。
- 数据偏移：表示TCP数据部分应该从哪个位算起，可看成TCP首部的长度。
- 控制位：控制标志，ACK=1表示确认号有效，SYN=1表示建立连接，FIN=1表示释放连接。
- 窗口：滑动窗口大小，用于告知发送端的缓存大小，用于流量控制。
- 校验和：校验TCP报文段。



### 5、三次握手

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/三握.png" style="zoom:80%;" />



**为什么不能用两次握手：**
因为三次握手完成两个事：
	1.协商交换序列号seq，这个seq在握手中被发送和确认。
	2.通知彼此都已准备好，防止ACK丢失导致S端资源占用。如果两次握手，S端应答ACK后进入就绪状态，但ACK丢失，C端不知道已经连接成功，会占用服务器资源。

### 6、四次挥手

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/四挥.png" style="zoom: 80%;" />

1.客户端发送FIN报文段，seq为随机值，客户端进入FIN_WAIT_1 状态；

2.服务器响应ACK报文段，ack为seq+1，客户端进入FIN_WAIT_2状态，服务器进入CLOSE_WAIT状态；

3.服务器发送FIN报文段，服务器进入LAST_ACK状态；

4.客户端响应ACK报文段，进入TIME_WAIT状态，等待2MSL报文段最大存活期之后进入CLOSE（防止客户端发出的ACK丢失），服务器收到ACK后进入CLOSE；

**为什么四次挥手？**
	TCP是全双工的，客户端发出FIN并接受ACK表示客户端可以关闭，服务器也要发出FIN和接受ACK才会关闭连接。

**为什么客户端从TIME_WAIT到CLOSED要等待2MSL？**
	MSL表示报文段存活的最大时间，FIN+ACK所以需要2MSL。如果过了2MSL内没有再收到服务器发的FIN，这表示没有丢包。



### 7、流量控制

​	在TCP首部，16位的字段用来表示窗口大小。接收端可以将自己接受的缓冲区大小放到这个字段通知给发送端。发送端主机会根据接收端主机的指示，对发送数据的量进行控制。 

### 8、拥塞控制

​	虽然使用流量控制可以知道接收方的缓存大小，从而控制发送的数据量，当网络环境是不得而知的，所以为了避免造成网络拥堵的情况，还需要进行拥塞控制。
​	发送方维护名为拥塞窗口的变量cwnd，拥塞窗口通过算法动态更新，发送方会选取拥塞窗口和接收端通知的窗口中最小的那个，最为发送的窗口大小。

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/拥塞控制.png" style="zoom:80%;" />

1.慢开始算法：拥塞窗口初始值为1个MSS最大报文段长度（标准值为1460字节），每次收到ACK，cwnd 就乘2；

2.拥塞控制：当cwnd增长到慢开始算法阈值的时候，不再慢开始算法，而是拥塞控制算法，每次cwnd增加值为1

3.快重传：发送方只要连续收到三个重复对同一个报文M2的确认，就立即重传对方尚未收到的M3报文，而不是等到M3设置的重传计时器到期。

4.快恢复：发送方连续收到三个重复确认，将慢开始阈值减半，但不把cwnd设为1，而是设置成阈值减半后的树脂，并开始执行拥塞避免算法。

### 9、流量控制和拥塞控制的区别

1.流量控制用于通知发送端：接收端的接受缓存是多大；

2.拥塞控制用于发送端实时监测网络拥塞情况，进而调整发送窗口。

发送端会选取流量控制窗口和拥塞窗口中较小的那个，作为发送窗口大小。

### 10、TCP UDP区别

- 面**向字节流/面向报文**：
  	TCP会对数据报文进行拆分和拼接，然后以字节流为基本单位进行发送，例如应用程序发来两个100字节的数据，TCP可能会分两次发，每次发100，也可能分四次发每次发50字节；利于流量控制和顺序控制。
    	UDP对应用程序发下来的报文，在添加首部后直接发送，即不合并也不拆分，。例如应用程序发来两份100字节的数据报，那么UDP添加首部后直接发出去。
- **面向连接/面向无连接**：UDP不需要握手就可以发送数据了。
- **可靠/不可靠**：可靠体现在三次握手、流量控制、拥塞控制、超时重传、顺序确认

### 11、Http协议

1. **Http报文结构**

   <img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/Http报文结构.png" style="zoom:80%;" />

1. **Http请求方法**
   get：请求参数跟在url后面，没有请求体，一般用于查询；
   post：请求参数放在请求体，非明文较安全，一般用于更新资源信息；
   put：把资源放在指定位置。类似post，但put指定了资源存放位置，而post数据存放位置由服务器决定。
   head：类似与get，但服务端只返回响应头，通常用于查看页面状态；
   delete：删除某个资源
   
2. **常见请求头**
   Referer：表示这个请求是从哪个url跳过来的，如果是直接访问则没有这个头。
   Accept：告诉服务器，该请求支持的响应数据类型，如text/html，application/xml，image/apng
   Cokkie：客户端的Cookie
   Connection：连接类型，如keepAlive持久链接，close已关闭
   Host：服务器主机名
   Content-Length：请求体长度
   Content-Type：请求体类型， application/x-www-form-urlencoded  ，  application/json 、text/xml
   
3. **Http状态码**
   1xx：指示信息，表示请求已接收，继续处理；
   2xx：成功，处理成功；
   3xx：重定向
         301：永久重定向
         302：临时重定向
         303：临时重定向，303明确客户端应该使用GET访问
         307：临时重定向，307明确客户端应该使用POST访问
   4xx：客户端错误
         400：客户端请求语法错误
         401：请求未经授权
         403：拒绝请求
         404：资源不存在
   5xx：服务器错误
   
   
   常用的：
   200 OK  ：服务器成功返回用户请求的数据 
   201 CREATED ：用户新建或修改数据成功。 
   202 Accepted：表示请求已进入后台排队。 
   400 INVALID REQUEST ：用户发出的请求有错误。 
   401 Unauthorized ：用户没有权限。 
   403 Forbidden ：访问被禁止。 
   404 NOT FOUND ：请求针对的是不存在的记录。 
   406 Not Acceptable ：用户请求的的格式不正确。 
   500 INTERNAL SERVER ERROR ：服务器发生错误。
   
5. **Http协议版本**1.0/1.1/2
   **http1.0**：规定客户端-服务器只保持短暂的连接，每次请求都要建立新的TCP连接，完成请求后断开TCP连接。
   **http1.1**：
         支持长连接：一个tcp连接中可以传送多个http请求，connection请求头值为keep-alive时，表示本次请求后继续保持连接。
         新增host字段：1.0认为每台服务器绑定一个ip地址，但实际一个服务器存在多个虚拟主机且共享一个地址，host字段用于标记主机名
   **http2.0**：
         实现完全多路复用：一个连接里，可以同时发送多个请求和响应，不用按照顺序排队，避免了队头拥堵
         报头压缩：降低流量。
         服务器推送：服务器不再是被动接受的一方，服务器可以主动推送数据。

### 12、幂等

幂等意味着对同意URL的多个请求应该返回同样的结果。
**get是幂等的**：虽然两次对同一个url的GET可能返回的数据不同，但没有改变资源
**post是非幂等的**：用于提交请求，可以更新或创建资源。比如新建一个便签条，提交几次post就会创建几张便签。
**put是非幂等的**：用于向指定的url传送更新资源，比如对便签1输入ABC，不管更新几次都一样，PUT的幂等是覆盖性的，新指覆盖旧值。
      比如我有一个博客系统，用http://localhost/create_new/{$name}来新生成一个文章，生成的文章的标题叫做$name，如果对接口发两次请求，生成了两篇叫做name的文章，那么这个就不是幂等的。如果第二次的请求把第一次请求覆盖掉了，那么这个接口就是幂等的。  后一种情况应该使用PUT方法，而前一种情况应该是使用post方法 

### 13、Https协议

证书校验-->非对称加密交换密钥-->对称加密通信

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/Https流程.png" style="zoom:80%;" />

1. 客户端发起请求，请求报文中包含SSL版本，加密算法；
2. 服务器向CA机构申请数字证书，数字证书包含服务器公钥。数字证书是被CA机构签名的，签名用于保证数字证书的完整性和真实性，防止数字证书被篡改；
3. 服务器响应，发送数字证书；
4. 客户端收到数字证书后，先用CA机构的公开密钥校验数字证书的真实性，然后提取出服务器公钥，使用公钥加密客户端生成的随机码KEY；
5. 非对称加密发送随机码KEY；
6. 服务器用私钥解密随机码KEY；
7. 使用随机码进行对称加密通信。



# 第三章：JVM

### 1、JVM内存模型

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/java7的jvm内存模型.png" style="zoom:80%;" />

- **线程独占：**
  - **程序计数器**：当前线程执行的字节码行号指示器。
  - **虚拟机栈**：每个方法的执行都会创建一个栈帧，栈帧存储局部变量表、方法出口、操作栈等信息，方法的调用和完毕对应栈帧的入栈和出栈。局部变量表存放基本数据类型，对象引用，返回地址值类型，其所需空间在编译器完成分配。
  - **本地方法栈**：与虚拟机栈类似，只是为native方法服务。
- 虚拟机栈：每个方法的执行都会创建一个栈帧，栈帧存储局部变量表、方法出口、操作栈等信息，方法的调用和完毕对应栈帧的入栈和出栈。局部变量表存放基本数据类型，对象引用，返回地址值类型，其所需空间在编译器完成分配。本地方法栈：与虚拟机栈类似，只是为native方法服务。
- **线程共享：**
  - **堆**：存放对象实例和数组。分为新生代和老年代
    	1、新生代占堆内存的1/3
      			1.1、新生代的80%是Eden区
      			1.2、新生代的10%、10%分别是From Survivor、To Survivor。
      			1.3、new出来的对象放到Eden，Eden满了进行GC，存活的放S0，S0满了就对S0进行GC，存活的放S1，S1满了就 GC存活的放老年代。
      	2、老年代占堆内存的2/3
  - **方法区**：存储字符串常量池、运行时常量池（编译期生成的字面量和符号引用）、静态变量、类信息（对象类型、父类、接口、方法等）。在JDK1.8之前方法区的实现叫永久代用堆内存，JDK及以后方法去的实现叫元空间用本地内存。



### 2、方法区的实现：从永久代到元空间（jdk1.8）

​	方法区存储：常量、静态变量、类信息（对象类型、父类、接口、方法等）。方法区是一种设计规范永久代和元空间都是方法区的实现方式，在JDK1.8之后，用元空间取代永久代。图示JDK1.7的jvm划分。
​	<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/java7的方法区模型.png" style="zoom:80%;" />
​	JDK1.7及以前：方法区的实现方式是永久代，永久代在物理上和堆区连续（字符串过大会造成堆溢出），逻辑上是分开的，受堆区大小影响。
​	JDK1.8及以后：使用元空间代替永久代，字符串常量池保留在堆中，类信息放到元空间中，元空间使用的是本地内存，空间很大。
​	字符串永远在堆中的，类信息在1.8之后移到元空间。至于静态变量和运行时常量，没资料说明。

### 3、对象创建过程

![](/home/xiaozhidong/Desktop/learningRecord/Study/pic/对象创建过程.jpg)

1. **类加载**：检查是否能够在常量池中定位到这个类的符号引用，如果没有，先执行类加载。
2. **分配内存**：类加载完毕后可确定对象大小，在堆中为对象分配内存。
3. **初始化**：内存分配后，虚拟机必须将分配的内存控件都初始化为零值，保证对象的实例字段可以被程序访问。
4. **设置对象头**：对对象头设置必要的信息，包括：类的元数据信息、对象的hashcode、GC分代年龄、锁状态标志。
5. **执行init方法**：按照代码的意愿对对象进行初始化。

### 4、对象的结构

对象的结构包括三部分：对象头、实例数据、对齐填充

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/对象结构.png" style="zoom: 90%;" />

- **对象头**：分为两部分：对象自身的运行时数据（也叫Mark Word）,另一部分是类型指针
  1、**MarkWord**：包括
  		1.1对象hashcode、GC分代年龄、锁状态标志、
  		1.2偏向锁所有者线程ID（指向线程ID）、
  		1.3轻量锁指针（指向锁持有者线程的栈帧中的锁记录）、
  		1.4重量锁指针（当锁标志为重量锁时，指向monitor对象）
  <img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/MardWord结构.png" style="zoom: 67%;" />
  2、**类型指针**：指向方法区的类元数据，用来确定这个对象属于哪个类。
- **实例数据**：对象真正存储的有效信息，在代码里定义的各个字段内容
- **对齐填充**：用于填充对象，使对象大小刚好为8字节倍数。

### 5、对象的访问

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/对象引用方法.png" style="zoom:80%;" />

1. 直接指针法**：虚拟机栈的局部变量表中存放的引用直接指向堆中对象，好处是快，缺点是对象移动时，引用要重新定位
2. **句柄法**：堆中划分内存作为句柄池存放对象的地址，栈中的引用指向句柄池，句柄再指向真实对象。好处是对象的移动只需修改句柄池，缺点是两次的定位比较慢。



### 6、判断对象是否需要被回收

1. **引用计数算法** ：给对象添加一个引用计数器，对象被引用，计数器+1，否则-1，计数器的值为0，说明对象可被回收。但是无法解决相互引用的问题。

2. **可达性分析算法（根搜索）** ：以一系列被称为GC Root的根对象作为起始节点，向下搜索，走过的路径叫做“引用链”，当一个对象到GCRoot之间没有任何引用链，说明对象可被回收。

   ​	GC Roots：**1、**虚拟机栈中的方法引用的对象，**2、**本地方法栈中native方法引用的对象，
   ​						**3、**方法区中静态变量引用的对象，**4、**方法区中常量引用的对象

### 7、引用的分类

1. **强引用**：最传统的引用定义，如Object obj = new Obj() ， 只要强引用存在就不会被GC
2. **软引用**：SoftReference类来实现软引用，内存空间不足就会被GC，
3. **弱引用**：WeakReference类实现弱引用，每次执行GC时，不管空间够不够，都会被回收
4. **虚引用**：PhantomRefence类实现虚引用，类似与弱引用，但必须与引用队列（ReferenceQueue）联合使用，主要用于跟踪GC活动。

### 8、finalize()方法

​    当对象被判定GC Roots不可达时，GC判断对象是否覆盖了finalize()方法，如果没覆盖，直接回收。如果覆盖了finalize，则调用finalize方法，本次GC不回收该对象。等到下次再次GC时，再判断该对象是否可达，不可达则回收。

### 9、垃圾收集算法

新生代GC：Minor GC/Young GC
老年代GC：Major GC/Old GC
全GC：整个堆区和方法区Full GC

1. **标记清除算法：**首先用可达性分析算法标记存活对象，然后统一回收未被标记对象。缺点产生大量内存碎片
2. **标记复制算法：**“半区复制”思想，适用于存活对象较少的新生代。将内存等大划分出两块，使用可达性分析算法标记存活对象，将存活对象复制到空闲区域，全部清理工作区的对象。对于Eden，工作区是自己空闲区是S0，工作S0->空闲S1，S1->老年代。简单高效，缺点是内存空间划分容量变小
3. **标记整理算法：**适用于存活对象校对的老年代。先标记存活对象。然后让所有存活对象向一段移动，然后直接清理掉边界以外的内存。移动对象并更新引用是很慢的操作，需要全程暂停用户操作Stop the world

### 10、垃圾收集器

并行：说的是垃圾收集器线程之间的并行关系，用户线程等待，垃圾收集器多线程执行。
并发：说的是垃圾收集器与用户线程的并发关系，用户线程在执行，垃圾收集器线程也在执行。
**标记垃圾和清理垃圾都要暂停用户线程，以及空间利用的考虑，所有的优化都是针对时间和空间来进行**

1. 串**Serail**：用于新生代的单线程收集器，复制算法
2. 串**Serial Old**：老年代的单线程收集器，标记整理算法
3. 并行**PraNew**：Serial的多线程版本，新生代复制算法
4. 并行**Parallel Scavenge**，新生代复制算法，与Parnew的区别是，ParNew尽可能缩短垃圾收集所需时间，Parallel尽可能降低（ GC时间：用户线程时间）的比值（也叫吞吐量）。
5. 并行**Parallel Old**：用于老年代的，标记整理算法
6. 并发**CMS**：用于老年代标记清除算法，并发标记+并发清除机制，可以不暂停用户线程，上面的算法都要暂停线程。
7. 并发**G1**：
   1、服务端的默认收集器，能够独立管理整个堆（新生代和老年代），不需要搭配其他收集器。
   2、保留分代概念，但将堆区划分多个大小相等的独立区域（Region），每个Region都可以扮演Eden或Survivor或Old。
   3、整体基于标记整理，Region之间使用复制算法。
   4、并发标记+并发清除

### 11、编译→类加载→运行

1、编译：通过javac 命令将.java文件编译成.class字节码文件
2、类加载：JVM把 .class文件中的类信息加载到内存，并进行校验解析，最终生成可被JVM直接使用的Java数据类型。
3、运行把编译生成的.class文件交给JVM执行

### 12、开始类加载的时机

1. new对象，创建实例的时候
2. 访问类或接口的static变量或方法
3. 反射，Class.forName("");
4. 初始化这个类的子类，会首先加载子类的父类
5. JVM会先初始化main函数所在的类。

### 13、类加载过程

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/类加载过程.png" style="zoom: 80%;" />

加载→链接（验证、准备、解析）→初始化

1. **加载：**
   将各个来源的字节码.class文件通过加载器载入内存中。
   字节码文件来源：本地编译的.class文件、jar包中的.class文件、通过网络加载的.class文件
   类加载器：JVM提供，也可自实现ClassLoader基类来定义
2. **链接：**
   2.1、**验证**：验证字节流是否符合class文件规范。例如：文件格式验证、元数据验证、字节码验证、符号引用的验证
   2.2、**准备**：为类的静态变量分配内存。设置默认零值，若是static final修饰的则赋予真实值。
   2.3、**解析**：将常量池内的符号引用替换成直接引用。 符号引用就是一个类中，引入了其他的类，可是JVM并不知道引入的其他类在哪里，所以就用一个符号来代替，等到类加载器去解析的时候，就把符号引用指向被引用类的地址，这个地址也就是直接引用 
3. **初始化：**为static赋予正确的值

### 14、类加载器

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/类加载器.png" style="zoom: 80%;" />

- **BootStrap ClassLoader：**根加载器，负责加载 <JAVA_HOME>/lib目录中的 java核心库类。C++实现的
- **Extension ClassLoader：**用来加载<JAVA_HOME>/ext目录下的拓展库类
- **Application ClassLoader：**加载应用目录下的类，Java应用的类几乎都是由他加载。

### 15、双亲委派模型

​      除了最顶层的BootStrap ClassLoader，类加载器收到类加载请求，首先不会自己加载这个类，而是委派给父类加载器，因此所有类加载起都应传给BootStrap ClassLoader，只有父类无法完成加载的时候，子类才会自己去加载。
​      好处：避免类的重复加载，保证类在所有的类加载器环境中都是同一个类。

### 16、破坏双亲委派模型

1. **Tomcat破坏：**
   	每个Tomcat的webappClassLoader加载自己目录下的class文件，不会传给父类加载。
      	1、因为对于不同webapp下的jar包，需要相互隔离，可能出现同名但版本不同的类，不能出现某个应用的jar包加载影响了其他应用。
      	2、因为web库有的类名与java库冲突，为了实现web类与java类隔离。
2. **JDBC破坏：**
   1、JDBC4.0之前使用Class.forName("")方式加载驱动是**不会破坏**双亲委派的。
   2、JDBC4.0之后使用spi机制**才会**破坏双亲委派机制。 
         因为Bootstrap ClassLoader和Extension ClassLoader，只能加载自己指定目录下的类，无法加载运行时的类。
         Java提供的DiverManager接口位于<JAVA_HOME>/lib目录下，本应由BootStrap ClassLoader加载，而Diver接口的实现类实际是位于服务商(如MySQL)提供的Jar里面的。所以因为BootStrapClassLoader不能加载mysql.jar包里面的类，所以要破坏双亲委派模型，将DriverManager和Driver的实现类都交由自定义的加载器加载。

### 17、JVM调优

1. **目的：**减少FullGC和GC所需时间
2. **方法：**设置新生代和老年代大小、选择合适的GC器、减少对象的创建
3. **常用JVM分析工具：**JConsole，图表化的形式显示各种数据，并可远程监视服务器VM，
4. **重要参数：**
   堆配置：
       -XX：InitialHeapSize = 100m //初始堆大小
       -XX：MaxHeapSize = 100m   //最大堆大小
       -XX：NewSize = 100m            //年轻代大小
       -XX：NewRatio = 2              //年轻代：老年代的比值
       -XX：SurvivorRatio = 8       //Eden区：Survivor区的比值
   元空间配置：
       -XX：MetaspaceSize = 50m  //设置元空间大小
   GC器配置：
       -XX：+UseSerialGC  //设置为使用串行Serial GC器
       -XX：+UseG1GC      //设置为使用G1收集器
       -XX：+PrintGC         //打印GC过程 

# 第四章：Spring

### 1、Spring四种注入方式

1. **Set注入**
   使用@AutoWired  或者 <property name="userDao" ref="userDao" /> 
2. **构造器注入**
   使用@AutoWired 或者 <constructor-arg index="0" ref="userDao"></constructor-arg> 
3. 静态工厂的方法注入
4. 动态工厂的方法注入

### 2、Spring容器启动流程 & Bean初始化过程

1. **容器的初始化**
   1.1、实例化BeanFactory对象；
   1.2、实例化注解扫描器对象（用于@Autowired自动注入）；spring自动装配的实现
   1.3、实例化路径扫描器对象（用于xml的扫描和注入）；
2. **注册BeanDefinition**
   加载配置文件，解析出beanDefinition，注册到容器中的Map<>中
3. **刷新refresh()**,包含Bean初始化过程
   1、BeanFactoryPostProcessor()，对beanfactory进行进一步设置；
   2、registerBeanPostProcessors()：如果用户想要在bean的初始化前后进行操作，如代理对象，修改属性等。
   3、InitMessageSource()：如果用户向支持国际化、消息机制
   4、registerListeners()：如果用户向监听容器启动、刷新等事件
   5、finishBeanFactoryInitialization()：初始化所有单例例bean
   ![](/home/xiaozhidong/Desktop/learningRecord/Study/pic/fresh流程2.png)



### 3、SpringBean生命周期

Spring管理bean，会使用BeanDefinition对象来描述对象的元信息。Spring启动时，扫描配置封装成BeanDifinition注册到Map，通过反射将BeanDefinition实例化成对象。
SpringBean的生命周期中，提供了很多hook供开发者拓展
1、Bean实例化之前有BeanFactoryPostProcessor
2、Bean实例化之后，有相关的Aware接口去设置属性
3、Bean初始化阶段，有BeanPostProcessor的before和after方法（AOP的关键）
4、初始化阶段，有各种init方法
![](/home/xiaozhidong/Desktop/learningRecord/Study/pic/springBean生命周期.png)

### 4、Spring怎么解决循环依赖

**大致过程：**首先A对象实例化，进行属性注入时发现依赖B对象，转头去实例化B对象，对B进行属性注入时发现依赖A对象，从缓存中拿到A对象，B对象初始化完成并返回到A对象进行属性注入，完成初始化。

**原理：**三级缓存，就是三个Map
singletonObjects：一级缓存，正式对象
earlySingletonObjects：二级缓存，半成品，还没完成属性注入，里面的对象来自三级缓存
singletonFactories：三级缓存，存的是对象工厂，用于生产半成品对象（可以支持AOP增强）
对象实例化后，会把对象放到三级缓存。
A对象实例化后放到三级缓存中，进行属性注入时，发现依赖B对象，去实例化B，B被放到三级缓存，发现依赖A，从三级缓存取出A，并放到二级缓存中，等到初始化完毕之后就把二级缓存移到i一级缓存。

![](/home/xiaozhidong/Desktop/learningRecord/Study/pic/spring解决循环依赖流程图.png)

### 5、BeanFactory和ApplicationContext的区别

Spring有两种容器，都是对beans进行管理的。

1. **BeanFactory**：最底层的接口提供了最简单的容器功能，DI依赖注入的实现；在启动的时候不会实例化Bean，getBean()方法执行的时候才会去实例化，适用于资源紧张的小型应用；
2. **ApplicationContext**：应用上下文，集成BeanFactory接口，提供了更多功能：国际化、资源访问、AOP。在启动的时候就把所有Bean实例化了。

### 6、BeanFactory & FactoryBean的区别

1. **BeanFactory**是Spring IOC的核心容器接口，用于bean的管理；
2. **FactoryBean**是一个实现了FactoryBean接口的bean，根据该bean的ID从容器中获取出来的不是FactoryBean本身，而是其getObject()返回的对象。用于实现代理和对对象方法做拦截。

### 7、bean的作用域

singleton：单例，spring默认的作用域。

prototype：多实例，每次对bean的调用都会生成一个实例。

request：每次http请求都会创建一个bean

session：在一个session中，生成一个bean实例

global：在一个全局的HTTP Session中，一个bean对应一个实例。

### 8、bean的自动装配

1. 方式一：通过XML配置，使用byName、byType参数可指定装配的bean
2. 方式二：使用@Autowired @Resource注解
   在Spring容器启动时，容器自动装载了一个AutowiredAnnotationBeanPostProcessor后置处理器。会扫描Spring容器中的所有bean，当扫描到@Autowired的时候，会在IOC容器中查找需要的bean，并注入给该对象的属性。

### 9、Spring AOP 和AspectJ AOP区别

AOP代理分为静态代理和动态代理

AspectJ是静态代理的增强，就是AOP框架会**在编译阶段生成AOP代理类**，也称编译时增强。会在编译阶段将AspectJ切面织入到 Java字节码中。

Spring AOP使用的是动态代理，不会修改字节码文件，而是**在运行时在内存中为对象生成**一个AOP对象。

### 10、Spring事务管理方式有几种

1. **编程式事务：**在代码中硬编码
2. **声明式事务：**在配置文件中配置，分为基于xml的声明式事务和基于注解的声明式事务

### 11、Spring事务的隔离级别

ISOLATION_DEFAULT：使用后端数据库默认的隔离级别。例如MySQL默认采用REPEATABLE_READ隔离级别。

ISOLATION_READ_UNCOMMITTED：读未提交

ISOLATION_READ_COMMITTED：读已提交

ISOLATION_READ_REPEATABLE_READ：可重复读

ISOLATION_SERIALIZABLE：串行。

### 12、Spring事务传播行为

事务传播，指的是当一个事务方法被另一个事务方法调用时，这个事务方法应该如何进行。例如当methodA事务方法调用methodB事务方法时，methodB是继续在methodA的事务中运行，还是为自己开启一个新事务运行。这就是由methodB的事务传播行为决定的。

1. PROPAGATION_REQUIRED：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务
2. PROPAGATION_SUPPORTS：如果当前存在事务，则加入该事物；如果当前没有事务，则以非事务的方式继续运行
3. PROPAGATION_MANDATORY：如果当前存在失误，则加入该事务，如果当前没有事务，则抛出异常
4. PROPAGATION_REQUIRES_NEW：创建一个新的事务，如果当前存在事务则把当前事务挂起。
5. PROPAGATION_NOT_SUPPORTED：以非事务方式运行，如果当前存在事务，则把当前事务挂起。
6. PROPAGATION_NEVER：以非事务方式运行，如果当前存在事务，则抛出异常
7. PROPAGATION_NESTED：如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行，如果当前没有事务，则该取值等价于PROPAGATION_REQUIRED



# 第五章：Redis

### 1、Redis数据类型

redis的数据结构都是key-value型，value的类型有五种：

1. **String**：字符串 set keyName "rose"
2. **List**：双端列表 lpush keyName "rose"
3. **Set**：是一个string的无须集合 sadd keyName “rose”
4. **Hash**：是一个map集合，hmset keyName   name rose   sex woman
5. **ZSet**：sorted set有序集合，有一个分数与之关联。zadd keyName 100 "rose"

### 2、Redis为什么这么快

1. 完全基于内存，数据在内存中的存在形式是key -value，查找复杂度是O1
2. 采用单线程，避免了用户态/内核态切换 和锁的竞争
3. 使用多路I/O复用模型，非阻塞IO

### 3、Redis持久化机制

1. **RDB**（默认）：Redis DataBase。每隔一段时间将内存的数据持久化到硬盘，缺点：数据更新了不能实时同步到硬盘
2. **AOF**：Append Obly File，每次Redis执行的写命令记录到日志文件中，当重启Redis会通过持久化日志恢复数据。优点：数据安全，不用担心redis宕机。缺点：AOF文件大恢复速度慢。

### 4、Redis事务

Redis事务功能通过MULTI、EXEC、DISCARD、WATCH四个原语实现的

Redis单进程，支持隔离性
Redis没有回滚机制，不保证原子性，某条命令失败会继续往下执行
Redis开启AOF持久化机制，可以实现持久性

### 5、缓存击穿、缓存穿透、缓存雪崩

1. **缓存击穿：**
   描述：热点key过期了，导致大量请求直接打到数据库。
   解决：
   1、使用互斥锁，对“读取数据库加载到缓存”的操作进行加互斥锁。
   2、异步线程去更新即将过期的数据。
2. **缓存穿透：**
   描述：缓存和数据库都没有某个值，导致每次对这个值的查询请求都会穿透到数据库。
   解决：
   1、可以给不存在的值在缓存中设置为空值。
   2、通过布隆过滤器快速判断数据是否存在，存在才继续查询。布隆过滤器：使用bitmap位图，用位的0 1 来表示某个值是否存在。
3. **缓存雪崩：**
   描述：缓存中大批量数据到期，导致数据库查询压力大，引起mysql压力过大down机。或者是redis故障导致mysql服务器宕机。
   解决：
   1、通过设置不同的过期时间来避免同时间的大量过期。
   2、搭建redis集群来避免redis宕机。

### 6、Redis高可用：三种部署模式

1. **主从模式**：一主多从，主节点用来读写，从节点用来读。增量复制保证主从一致性。
   优点：分担主节点的访问压力。
   缺点：mater故障无法自动晋升主节点。
2. **哨兵模式**（Sentinel）：由一个哨兵实例负责监视所有的Redis主节点和从节点。在master故障时可以自动将某个从节点升级为主节点。
   优点：检测主节点宕机，自动升级从节点切换为主节点。
   缺点：切换推举新master时，redis服务是不可用的；并且从节点保存的内容都是一样的，浪费内存。
3. **集群模式**（Cluster）：一个Redis集群由多个节点组成。cluster集群采用的分布式算法不是一致性hash而是slot插槽算法。把整个redis集群分为16384个槽。每个节点负责一部分槽位。每个进入redis的键值对 通过取余进行散列。每个节点是主从结构，节点down掉就启动从节点。

### 7、加入Redis里面有1亿个key，其中有10w个key是以某个固定的已知的前缀开头的，如何将它们全部找出来？

# 第六章：RabbitMQ

### 1、MQ的优点

异步、削锋、解耦

### 2、MQ对比

ActiveMQ：老牌MQ，功能完备

RabbitMQ：基于erlang开发，并发能力强，性能好，消息可靠性保证

RocketMQ：阿里开源MQ，消息0丢失

Kafka：高吞吐，适用于大数据领域。

### 3、延时队列

适用场景：下单30min后未付款，取消该订单

原理：利用Time To Live消息过期时间、Dead Letter Exchanges死信队列。在A队列中过期的消息会发给死信队列，消费者监听死信队列

### 4、消息不丢失

1. 生产者->MQ：事务机制或者Confirm机制；
2. MQ->消费者：basicAck机制，死信队列，消息补偿机制；
3. MQ自身：持久化，集群。

### 5、消息顺序性

必须是一个MQ一个消费者，如果有多个消费者就保证不了消息有序被消费。



# 第七章：Java基础

### 1、Synchronized锁升级

对象头中的MarkWord信息：

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/MardWord结构.png" style="zoom:70%;" />
	Java中每个对象都可以作为锁，对象头中的MarkWord部分保存了与锁相关的数据。按照量级从轻到重分为：无锁、偏向锁、轻量锁、重量锁。

1. **自旋与适应性自旋：**自旋是指发生锁竞争时，线程不会被阻塞，而是不停重试获取锁，从而让线程一直运行，避免非常耗时的 用户态 内核态 的转变。适应性自旋：根据上一次是否自旋成功来决定本次自旋次数。
2. **偏向锁：**在没有线程竞争的环境。偏向锁只需要一次CAS，让对象头的偏向锁指针指向ThreadID，只需判断偏向锁的ThreadID是否等于当前线程ID即可，如果相等，就执行同步代码，如果CAS修改ThreadID失败，说明出现线程竞争，升级轻量锁。偏向锁的优点在于只用一次CAS，轻量锁需要多次CAS；
3. **轻量锁：**在有线程竞争但竞争不激烈的环境。线程将对象头的MardWord拷贝到自己的栈中，然后通过CAS将锁对象的MardWord更新为指向当前线程。当竞争线程自旋多次失败，升级重量锁。
4. **重量锁：**在线程激烈竞争的环境。加解锁过程类似轻量锁，区别是：重量锁竞争失败，直接进入阻塞态，不使用自旋。MarkWord中的重量锁指针指向monitor对象

### 1、微服务、分布式、集群。通俗理解

微服务是设计层面的，分布式是部署层面的。分布式是微服务的实现方式。

1.假设饭店是一个完整的系统，那么饭店的工作细分为切菜、烧菜、端菜、洗碗，这些是不同的服务（**微服务**）。

2.这些服务由不同的人（服务器）来干，就是**分布式**（不同服务在不同服务器上面）。

3.烧菜的一个不够就多招几个人来干，就是**集群**。

分布式实现了服务之间互不影响，专职专责。集群是为了降低同一份工作的压力。

# 第八章：Java并发

### 1、线程和进程

1. 进程是系统进行资源分配和调度的基本单位，是线程的容器
2. 线程是进行运算的基本单位

### 2、线程的主要状态

1. 新建状态：创建出一个线程对象，但没运行。
2. 就绪状态：调用对象的start( )方法，这时候还在等待cpu的调用。
3. 运行状态：cpu对程序进行调用。就绪状态的线程拿到cpu时间片之后变为运行中状态。
4. 阻塞状态：同步阻塞--线程synchronized失败；等待阻塞--调用wait方法，等待其他线程唤醒；其他阻塞--调用sleep或join或发出io请求。
5. 死亡状态：线程运行完，或者异常退出。

### 3、线程主要方法

yield( )：让线程从运行状态转为就绪状态，重新等待cpu调用，不释放锁。

sleep( )：运行态->阻塞态，不释放锁。

wait( )：运行态->阻塞态度，释放当前拥有的锁，知道其他线程notify( )、notifyAll( )、或超过了timeout时间会再次进入就绪态。

join( )：阻塞（原理是wait）执行join语句的线程，让目标线程插队开始执行。可以实现线程顺序，比如在main中执行ThreadA.start、ThreadA.join、ThreadB.start，可实现线程A、B顺序执行。或是在线程B中调用线程A的join，会让线程B阻塞，直到线程A结束。

notify( )：唤醒一个等待的线程，具体唤醒哪个不确定。

notifyAll( )：唤醒所有等待的线程。

interrupt( )：中断线程。

### 4、sleep和wait方法区别

1. sleep没有释放锁，wait释放了锁
2. sleep通常用于暂定，wait通常用于线程间的通信
3. sleep属于Thread类，Wait属于Object

### 5、线程状态转换

<img src="/home/xiaozhidong/Desktop/learningRecord/Study/pic/线程状态.png" style="zoom:90%;" />

### 6、创建线程的方式

1、继承Thread类；2、实现Runnable接口；3、实现Callable接口；4、线程池

### 7、Callable和Runnable区别

1. Callable的方法是call，Runnable的方法是run
2. Runnable没有返回值，Callable返回执行结果，是个泛型，和Futrue、FutrueTask配合可以用来获取执行结果
3. call方法允许抛出一场，run的异常只能内部catch

### 8、死锁和避免

1. 死锁：多个线程互相占用其他线程所需要的资源，因争夺资源产生的一种相互等待的现象
2. 四个条件：1、互斥；2、请求和保持；3、不剥夺；4、循环等待
3. 避免死锁：1、破坏互斥；2、破坏请求和保持，一次申请所有资源；3、破坏不剥夺，申请不到资源时，释放自身持有的资源；4、破坏循环等待

### 9、并发三大特性

### 10、java内存模型JMM

### 11、Happens-before

### 12、volatile关键字

### 13、CAS 和Automic类

### 14、Synchronized

### 15、可重入锁Reentrant Lock

### 16、Synchronized对比ReentrantLock

### 17、AQS框架

### 18、Thread Local

​	线程局部变量，以空间换时间的手段，为每个线程提供变量的独立副本，以无锁的情况下保障线程安全。主要解决的就是让每个线程执行完成之后再结束，这个时候就要用到join()方法。

**适用场景：**

- 在并发不是很高的时候，加锁的性能会更好
- 在高并发量场景下，使用ThreadLocal可以在一定程度上减少锁竞争

### 19、Executor框架
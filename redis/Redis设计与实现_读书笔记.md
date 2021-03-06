- [第一部分](#第一部分)
	- [简单动态字符串 SDS(Simple Dynamic String)](#简单动态字符串-sdssimple-dynamic-string)
	- [双向链表 list](#双向链表-list)
	- [字典(dict)](#字典dict)
	- [跳跃表](#跳跃表)
	- [整数集合](#整数集合)
	- [压缩列表](#压缩列表)
	- [对象](#对象)
		- [字符串对象](#字符串对象)
		- [列表对象](#列表对象)
		- [哈希对象](#哈希对象)
		- [集合对象](#集合对象)
		- [有序集合对象](#有序集合对象)
		- [其他](#其他)
- [第二部分](#第二部分)
	- [数据库](#数据库)
		- [关于键的生存时间](#关于键的生存时间)
			- [redis的过期键删除策略](#redis的过期键删除策略)
			- [AOF、RDB和复制对过期键的处理](#aofrdb和复制对过期键的处理)
	- [RDB持久化](#rdb持久化)
		- [持久化命令](#持久化命令)
		- [关于自动保存](#关于自动保存)
		- [RDB文件结构](#rdb文件结构)
	- [AOF持久化](#aof持久化)
		- [AOF重写](#aof重写)
	- [事件](#事件)
- [第三部分](#第三部分)
	- [主从复制](#主从复制)
		- [主从复制的实现](#主从复制的实现)
			- [同步之SYNC](#同步之sync)
			- [命令传播](#命令传播)
			- [仅使用SYNC同步与命令传播的问题](#仅使用sync同步与命令传播的问题)
			- [更高效的同步命令之PSYNC](#更高效的同步命令之psync)
			- [部分重同步的实现](#部分重同步的实现)
			- [主从复制的步骤：](#主从复制的步骤)
		- [心跳检测](#心跳检测)
	- [Sentinel（哨兵）](#sentinel哨兵)
		- [主观下线的判断](#主观下线的判断)
		- [客观下线的判断](#客观下线的判断)
		- [对下线服务器的故障转移操作](#对下线服务器的故障转移操作)
	- [集群](#集群)
			- [集群如何执行命令](#集群如何执行命令)
			- [计算键属于哪个槽](#计算键属于哪个槽)
			- [节点数据库](#节点数据库)
		- [集群复制](#集群复制)
- [第四部分](#第四部分)
	- [事务](#事务)
		- [事务的阶段](#事务的阶段)
		- [WATCH命令](#watch命令)
		- [ACID](#acid)

部分参考自https://www.cnblogs.com/monxue/p/3961380.html

## 第一部分

redis存储键值对，键总为字符串，值的数据类型有：字符串、列表、哈希、集合、有序集合。

redis用到的所有主要数据结构：简单动态字符串（SDS）、双向链表、字典、跳跃表、压缩列表、整数集合。

整个redis系统的底层数据支撑被设计为如下几种：  

### 简单动态字符串 SDS(Simple Dynamic String)  
内部结构：len属性记录长度，free属性记录未使用的字节数量，char[]记录具体的值。  
优化策略：
1. 空间预分配：若修改时需要扩展大小，且修改后SDS长度小于1M，将分配和len属性相同大小的未使用空间（len==free），大于1M时，将分配1M的未使用空间。  
2. 惰性空间释放：缩短字符串的操作时，不立即进行内存重分配而是维护free属性记录的未使用字节数量。  
	
SDS的优点：  
1. 获取字符串长度所需的复杂度从O（N）降低到了O（1）。维护有len属性。
2. 杜绝缓冲区溢出。对sds修改时会先检查SDS的空间是否满足需求，不满足的话，会自动扩展大小，再执行实际的修改操作。  
3. （空间预分配、惰性空间释放优化策略）通多free属性避免频繁的内存重分配。
4. 二进制安全。可以保存任意格式的二进制数据。c中以"\0"作为字符串结束，而redis使用len属性判断字符串是否结束。  
	
### 双向链表 list    
结构：表头、尾指针，长度len，函数（dup复制节点值、free释放节点值、match比较节点值是否与某值相等）


### 字典(dict)   
用于保存键值对的抽象数据结构。字典中键唯一。redis使用字典作为底层实现的，对redis的增删改查也是构建在对字典的操作之上。  
字典是哈希键的底层实现之一（当一个哈希键包含的键值对比较多，又或者键值对中的元素都是比较长的字符串时，Redis就会使用字典作为哈希键的底层实现。）。  
字典底层实现是哈希表，每个哈希表节点保存字典中的一个键值对。  
每个字典含有2个哈希表（ht[2]），一般只是用0号哈希表，1号哈希表是在rehash过程中才使用的。  
Redis的哈希表使用链地址法来解决键冲突。    

rehash过程：  
 1. 为ht[1]分配空间  
 2. 将ht[0]中的所有键值对重映射到ht[1]上  
 3. 释放ht[0]，将ht[1]设置为ht[0]，ht[1]新建一个空白哈希表  
 
 rehash的条件：（满足其一即可）
  1. 服务器目前没有在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1。
  2. 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5。
  3. 当哈希表的负载因子小于0.1时，程序自动开始对哈希表执行收缩操作。

渐进式rehash  
为了避免rehash对服务器性能造成影响，服务器不是一次性将ht[0]里面的所有键值对全部rehash到ht[1]，而是分多次、渐进式地将ht[0]里面的键值对慢慢地rehash到ht[1]。  

步骤：
1. 为ht[1]分配空间，字典同时持有ht[0]和ht[1]。
2. 将rehashidx置为0，表示rehash开始。
3. rehash期间对字典的增删改查操作会顺带将ht[0]表中在rehashidx索引上的所有键值对重映射到ht[1]，完成后将rehashidx加1。
4. 所有键值对都被rehash到ht[1]后，将rehashidx置-1表示完成。

进行渐进式rehash的过程中，字典会同时使用ht[0]和ht[1]两个哈希表，所以在渐进式rehash进行期间，字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行。例如，要在字典里面查找一个键的话，程序会先在ht[0]里面进行查找，如果没找到的话，就继续到ht[1]里面进行查找，诸如此类。新添加到字典的键值对一律会被保存到ht[1]里面，而ht[0]则不再进行任何添加操作，保证ht[0]随着rehash操作的执行最终变成空表。

### 跳跃表    
跳跃表是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。  
节点查找平均O（logN）、最坏O（N）时间复杂度。  
redis用跳跃表实现有序集合、在集群节点中用作内部数据结构。  

### 整数集合    
当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数集合作为集合键的底层实现。 
数据结构：encoding保存元素所使用的类型的长度，length记录元素个数，contents[]保存元素的数组。  
contents 有两个特点：没有重复元素；元素在数组中从小到大排序。  
整数集合的特点：  
1. 保存有序、无重复的整数元素
2. 根据元素的值自动选择对应的类型，但是int set只升级、不降级
3. 升级会引起整个int set中的contents数组重新内存分配，并移动所有的元素（因为内存不一样了），所以复杂度为O(N)
4. 因为int set是有序的，所以查找使用的是binary search

### 压缩列表  
当一个列表键只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做列表键的底层实现。
节点结构：previous_entry_length , encoding , content三部分  
previous_entry_length用于求前一节点的地址（现节点的指针-pel得前一节点指针，压缩列表从表尾到表头进行遍历）  
encoding属性记录了节点的content属性所保存数据的类型以及长度。  
content属性负责保存节点的值。  
连锁更新：前一节点长度<254字节时pel为1字节，>=254字节时为5字节。增加或删除节点时可能导致对pel字段大小的连锁更新(较为罕见，因为条件需要连续的若干个250字节到253字节大小的节点)。

### 对象  
对象由redisObject结构表示，此结构中与保存数据有关的属性分别是type（类型）、encoding（编码，ptr指向对象的数据结构）、ptr（指向底层的数据结构）。此外还有refcount（引用计数）、lru（记录对象最后一次被访问的时间）。  
Redis共有字符串、列表、哈希、集合、有序集合五种类型的对象，每种类型的对象至少都有两种或以上的编码方式，不同的编码可以在不同的使用场景上优化对象的使用效率。  

#### 字符串对象  
字符串对象的编码可以是int、raw或者embstr。  
int：当字符串对象保存的是long类型范围内的整数值时，值会保存在ptr属性里。
raw：字符串对象保存的是一个字符串值，且长度>32字节，那么字符串对象将使用SDS来保存这个字符串值，并将对象的编码设置为raw。
embstr：字符串对象保存的是一个字符串值，且长度<=32字节。  
embstr与raw编码的区别：  
embstr编码是专门用于保存短字符串的一种优化编码方式，这种编码和raw编码一样，都使用redisObject结构和sdshdr结构来表示字符串对象，但raw编码会调用两次内存分配函数来分别创建redisObject结构和sdshdr结构，而embstr编码则通过调用一次内存分配函数来分配一块连续的空间，空间中依次包含redisObject和sdshdr两个结构。
embstr的优点：
1. 内存分配次数从raw编码的两次降低为一次。
2. 释放embstr编码的字符串对象只需要调用一次内存释放函数，而raw编码需要调用两次。
3. embstr编码的字符串对象的所有数据都保存在一块连续的内存里面，所以这种编码的字符串对象比起raw编码的字符串对象能够更好地利用缓存带来的优势。  
  
notice：
1. 对embstr编码的字符串对象执行修改时，程序会先将对象的编码从embstr转换成raw，然后再执行修改命令。embstr编码的字符串对象在执行修改命令之后，总会变成一个raw编码的字符串对象。
2. 对对象的操作可能会改变对象的编码方式。  

#### 列表对象    
列表对象的编码可以是ziplist或者linkedlist。  
ziplist：对象使用压缩列表作为底层实现，每个压缩列表节点（entry）保存了一个列表元素。  
linkedlist：对象使用双端链表作为底层实现，每个双端链表节点（node）都保存了一个字符串对象，而每个字符串对象都保存了一个列表元素。  
满足以下条件时，使用ziplist编码：
1. 列表对象保存的所有字符串元素的长度都小于64字节；  
2. 列表对象保存的元素数量小于512个。    
  
否则使用linkedlist编码。

#### 哈希对象  
哈希对象的编码可以是ziplist或者hashtable。  
ziplist：对象使用压缩列表作为底层实现，每当有新的键值对要加入到哈希对象时，程序会先将保存了键的压缩列表节点推入到压缩列表表尾，然后再将保存了值的压缩列表节点推入到压缩列表表尾。  
hashtable：对象使用字典作为底层实现，哈希对象中的每个键值对都使用一个字典键值对来保存：字典的每个键都是一个字符串对象，对象中保存了键值对的键；字典的每个值都是一个字符串对象，对象中保存了键值对的值。  
满足以下条件时，使用ziplist编码：
1. 哈希对象保存的所有键值对的键和值的字符串长度都小于64字节；  
2. 哈希对象保存的键值对数量小于512个。  
  
否则使用hashtable编码。  

#### 集合对象  
集合对象的编码可以是intset或者hashtable。  
intset：使用整数集合作为底层实现，集合对象包含的所有元素都被保存在整数集合里面。  
hashtable：使用字典作为底层实现，字典的每个键都是一个字符串对象，每个字符串对象包含了一个集合元素，而字典的值则全部被设置为NULL。  

满足以下条件时，使用intset编码：
1. 集合对象保存的所有元素都是整数值；  
2. 集合对象保存的元素数量不超过512个。  
  
否则使用hashtable编码。  

#### 有序集合对象  
有序集合的编码可以是ziplist或者skiplist。  
ziplist：对象使用压缩列表作为底层实现，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存元素的成员（member），而第二个元素则保存元素的分值（score）。  压缩列表内的集合元素按分值从小到大进行排序。  
skiplist：使用zset结构作为底层实现，一个zset结构同时包含一个字典和一个跳跃表。通过跳跃表可以对有序集合进行范围型操作。通过字典dict可以在O（1）复杂度查找成员分值。  

满足以下条件时，使用ziplist编码：
1. 有序集合保存的元素数量小于128个；  
2. 有序集合保存的所有元素成员的长度都小于64字节。  
  
否则使用skiplist编码。  

#### 其他  
使用refcount对对象进行引用计数。  
redis只对包含整数值的字符串对象进行共享（其他对象检验是否相同复杂度较高）。  
如果服务器打开了maxmemory选项，并且服务器用于回收内存的算法为volatile-lru或者allkeys-lru，那么当服务器占用的内存数超过了maxmemory选项所设置的上限值时，空转时长较高的那部分键（即lru值较小）会优先被服务器释放，从而回收内存。  


## 第二部分    

### 数据库  
redis的数据库都保存在redisServer结构的db数组（由redisDb结构组成）中。dbnum默认值为16，即Redis服务器默认会创建16个数据库。    
redisDb结构的dict字典保存了数据库中的所有键值对，称为键空间（key space）。  针对数据库的redis命令是通过对键空间进行处理来完成的。  
键空间的维护操作：
1. 读取键时会更新键空间的命中次数或未命中次数。分别为INFO stats命令的keyspace_hits和keyspace_misses值。  
2. 读取键时会更新键的LRU值。使用OBJECT idletime命令查看key的闲置时间。  
3. 读取一个键时发现该键已经过期，则会先删除这个过期键，再执行余下的操作。
4. 若客户端使用WATCH命令监视了某个键，那么服务器在对被监视的键进行修改之后，会将这个键标记为脏（dirty），从而让事务程序注意到这个键已经被修改过。
5. 服务器每次修改一个键之后，都会对脏（dirty）键计数器的值增1，这个计数器会触发服务器的持久化以及复制操作。
6. 如果服务器开启了数据库通知功能，修改键后，服务器将按配置发送相应的数据库通知。  
#### 关于键的生存时间  
查看键的生存时间的方法：TTL < key >。（单位s）。PTTL < key >。（单位ms）。
设置键的生存时间共有四种命令：
1. EXPIRE＜key＞＜ttl＞命令用于将键key的生存时间设置为ttl秒。  
2. PEXPIRE＜key＞＜ttl＞命令用于将键key的生存时间设置为ttl毫秒。  
3. EXPIREAT＜key＞＜timestamp＞命令用于将键key的过期时间设置为timestamp所指定的秒数时间戳。  
4. PEXPIREAT＜key＞＜timestamp＞命令用于将键key的过期时间设置为timestamp所指定的毫秒数时间戳。  

前三个命令都会调用PEXPIREAT。  
redisDb的expires字典保存了数据库中所有键的过期时间，称为过期字典。
过期字典的属性：
1. 过期字典的键是一个指针，指向键空间中的某个键对象（aka某个数据库键）。  
2. 过期字典的值是一个long long类型的整数，值为键的过期时间（毫秒精度的UNIX时间戳）。  

PERSIST命令可以移除一个键的过期时间。

判定键过期的过程：
1. 检查给定键是否存在于过期字典：如果存在，那么取得键的过期时间。  
2. 检查当前UNIX时间戳是否大于键的过期时间：如果是，则已经过期；否则未过期。  

##### redis的过期键删除策略   
过期键的删除策略：  
1. 定时删除：创建定时器在键的过期时间执行对键的删除操作。   
	特点：对内存友好，但过期键较多时cpu开销较大，影响吞吐量。
2. 惰性删除：操作某键时检查是否过期，过期则删除，未过期则继续相应操作。  
	特点：对cpu友好，但长期未被访问的过期键仍会占用内存导致内存泄漏。  
3. 定期删除：每隔一段时间，删除过期键。  
	特点：是前两种策略的一种折中，每隔一段时间执行一次删除过期键操作，若时间间隔设置合适，就减少了因为过期键而带来的内存浪费，并通过限制删除操作执行的时长和频率来减少删除操作对CPU时间的影响。若删除过频繁或执行时间太长，会出现定时删除存在的问题。若删除操作频率过低，会出现惰性删除存在的问题。  
	
Redis服务器采用配合使用惰性删除和定期删除两种策略的方法。

##### AOF、RDB和复制对过期键的处理  
执行SAVE命令或者BGSAVE命令创建一个新的RDB文件时，会对数据库中的键进行检查，已过期的键不会被保存到新创建的RDB文件中。  
启动redis服务器时，如果开启了RDB功能将载入RDB文件。分为主从服务器两种情况：
 1. 以主服务器模式运行时，会检查文件中保存的键，载入未过期的键，忽略过期键。  
 2. 以从服务器模式运行时，文件中的所有键，不论是否过期都会被载入。但主从服务器进行数据同步的时候，从服务器的数据库就会被清空。  
  
服务器以AOF持久化模式运行时，过期但未删除的键对AOF文件没有影响，当过期键被删除时会在AOF文件追加一条DEL命令。  
在执行AOF重写（BGREWRITEAOF命令）的过程中，已过期的键不会保存到重写后的AOF文件中。  

服务器在复制模式下运行时，从服务器删除过期键的动作由主服务器控制，这保证了主从服务器的一致性。以下三个特点：  
 1. 主服务器删除一过期键时，会显式向从服务器发送DEL命令删除此过期键。  
 2. 从服务器在执行客户端发来的命令时，不会主动删除发现的过期键，而是像处理未过期的键一样来处理过期键。  
 3. 从服务器只有在收到主服务器发来的DEL命令时才会删除过期键。  

### RDB持久化   
#### 持久化命令  
有两个Redis命令可以用于生成RDB文件，一个是SAVE，另一个是BGSAVE。
RDB持久化功能所生成的RDB文件是一个经过压缩的二进制文件，可通过它还原数据库状态。  
SAVE命令会阻塞Redis服务器进程直至RDB文件创建完毕（阻塞期间不能处理命令请求）。  
BGSAVE命令会派生出一个子进程去负责创建RDB文件，父进程继续处理命令请求。  
BGSAVE命令执行期间，服务器仍然可以处理客户端的命令请求，但处理SAVE、BGSAVE、BGREWRITEAOF三个命令的方式会和平时有所不同。SAVE命令和BGSAVE命令会被拒绝，BGREWRITEAOF命令会被延迟到BGSAVE命令执行完毕之后执行。（如果BGREWRITEAOF命令正在执行，那么客户端发送的BGSAVE命令会被服务器拒绝。）

RDB文件的载入工作是在服务器启动时自动执行的，只要Redis服务器在启动时检测到RDB文件，就会自动载入。服务器在载入RDB文件期间，会一直处于阻塞状态，直到载入工作完成为止。   

判断用哪个文件还原数据库状态的原则：  
1. 如果服务器开启了AOF持久化功能，那么服务器会优先使用AOF文件来还原数据库状态。  
2. 只有在AOF持久化功能处于关闭状态时，服务器才会使用RDB文件来还原数据库状态。  

#### 关于自动保存  
可以通过save选项设置多个保存条件，但只要其中任意一个条件被满足，服务器就会执行BGSAVE命令。（save 900 1代表900秒内对数据库进行了至少1次修改就执行BGSAVE）。save选项未设置的话会有默认条件。     
服务器维持着一个dirty计数器以及一个lastsave属性。dirty计数器记录上次成功执行SAVE或BGSAVE命令之后，对所有数据库状态进行的操作次数。lastsave属性是一个UNIX时间戳，记录上次成功执行SAVE或BGSAVE命令的时间。  
Redis的服务器周期性操作函数serverCron默认每隔100毫秒就会执行一次，该函数用于对正在运行的服务器进行维护，它的其中一项工作就是检查save选项所设置的保存条件是否已经满足，如果满足的话，就执行BGSAVE命令。   

#### RDB文件结构  
RDB文件的最开头是REDIS部分，这个部分的长度为5字节，保存着“REDIS”五个字符。  
第二部分是db_version，长度为4字节，它的值是一个字符串表示的整数，这个整数记录了RDB文件的版本号。  
第三部分是database，包含着零个或任意多个数据库，以及各个数据库中的键值对数据。
第四部分是EOF，长度为1字节，这个常量标志着RDB文件正文内容的结束，当读入程序遇到这个值的时候，它知道所有数据库的所有键值对都已经载入完毕了。  
第五部分是check_sum，是一个8字节长的无符号整数，保存着一个校验和（对REDIS、db_version、databases、EOF四个部分的内容进行计算得出的）。  
其中database部分可以包含多个数据库，每个数据库分为三部分：SELECTDB、db_number、key_value_pairs。SELECTDB常量的长度为1字节，表示接下来是数据库号码。db_number保存数据库号码，程序读到此处时会切换到对性的数据库。key_value_pairs部分保存了数据库中的所有键值对数据，如果键值对带有过期时间，那么过期时间也会和键值对保存在一起。    
key_value_pairs部分保存的内容：
1. 不带过期时间的键值对在RDB文件中由TYPE、key、value三部分组成。TYPE记录了value的类型，长度为1字节。key总是一个字符串对象。value部分保存值对象，根据TYPE类型的不同，以及保存内容长度的不同，保存value的结构和长度也会有所不同。   
2. 带有过期时间的键值对在RDB文件中由EXPIRETIME_MS、ms、TYPE、key、value，后三部分与前一部分相同。EXPIRETIME_MS常量的长度为1字节，表示接下来是以毫秒为单位的过期时间。ms是一个8字节长的带符号整数，记录着一个以毫秒为单位的UNIX时间戳，这个时间戳就是键值对的过期时间。  

### AOF持久化  
AOF持久化功能的实现可以分为命令追加（append）、文件写入、文件同步（sync）三个步骤。  
命令追加：服务器在执行完一个写命令之后，会以协议格式将被执行的写命令追加到服务器状态的aof_buf缓冲区的末尾。  
在服务器每次结束一个事件循环之前，它都会调用flushAppendOnlyFile函数，考虑是否需要将aof_buf缓冲区中的内容写入和保存到AOF文件里面。flushAppendOnlyFile函数的行为由服务器配置的appendfsync选项的值来决定。appendfsync 的值有三种情况，always代表将aof_buf文件的所有内容写入并同步到AOF文件，everysec（默认值）代表将aof_buf文件的所有内容写入到AOF文件，如果上次同步AOF文件的时间距离现在超过1s，就对AOF文件进行同步，no代表将aof_buf文件的所有内容写入到AOF文件，何时同步由操作系统决定。  
always的效率最低，但最安全，故障停机数据库也只丢失一个事件循环的数据。everysec模式足够快，故障停机数据库只丢失1s的数据。no模式与everysec模式的效率类似，故障停机数据库丢失上次同步AOF文件后的所有写命令数据。  
Redis读取AOF文件并还原数据库状态的步骤：  
1. 创建一个不带网络连接的伪客户端。（因为只有客户端才能执行redis命令。）  
2. 从AOF文件中分析并读取出一条写命令。  
3. 使用伪客户端执行被读出的写命令。  
4. 一直执行步骤2和步骤3，直到AOF文件中的所有写命令都被处理完毕为止。  

#### AOF重写  
命令是BGREWRITEAOF。
AOF重写得到的文件所保存的数据库状态与原本文件相同，但新AOF文件不会包含冗余命令，体积要小得多。  AOF文件重写是通过读取服务器当前的数据库状态来实现的，并不需要现有的AOF文件。   
AOF重写功能的实现原理：首先从数据库中读取键现在的值，然后用一条命令去记录键值对，代替之前记录这个键值对的多条命令。重写程序在处理列表、哈希表、集合、有序集合这四种可能会带有多个元素的键时，会先检查键所包含的元素数量，如果元素的数量超过了redis.h/REDIS_AOF_REWRITE_ITEMS_PER_CMD常量的值，那么重写程序将使用多条命令来记录键的值，而不单单使用一条命令（为避免输入缓冲区溢出）。   
AOF重写程序在子进程里执行，这样做可以同时达到两个目的：  
1. 服务器进程（父进程）不会阻塞，可以继续处理命令请求。  
2. 子进程带有服务器进程的数据副本，使用子进程而不是线程，可以在避免使用锁的情况下，保证数据的安全性。
为解决子进程重写后的AOF文件所保存的数据库状态与当前服务器的状态可能不一致的问题：父进程在子进程进行AOF重写期间，会将执行后的写命令加入AOF缓冲区期间，父进程收到子进程完成AOF重写工作的信号后，会调用函数将AOF重写缓冲区的所有内容写入到新AOF文件中，并用新AOF文件原子地覆盖现有的AOF文件。这样既不会有一致性问题，也将AOF重写对服务器的影响降低。    

### 事件    
redis中的实践可分为文件事件和时间事件。  
文件事件处理器使用I/O多路复用程序来同时监听多个套接字，并根据套接字执行的任务来为套接字关联不同的事件处理器（可以理解为实现对应任务的函数）。  
何时产生文件事件：  
当被监听的套接字准备好执行连接应答（accept）、读取（read）、写入（write）、关闭（close）等操作时，与操作相对应的文件事件就会产生，这时文件事件处理器就会调用套接字之前关联好的事件处理器来处理这些事件。  
尽管多个文件事件可能会并发地出现，但I/O多路复用程序总是会将所有产生事件的套接字都放到一个队列里面，然后通过这个队列，以有序（sequentially）、同步（synchronously）、每次一个套接字的方式向文件事件分派器传送套接字。当上一个套接字产生的事件被处理完毕之后（该套接字为事件所关联的事件处理器执行完毕），I/O多路复用程序才会继续向文件事件分派器传送下一个套接字。  
时间事件  
服务器将所有时间事件都放在一个无序链表中，每当时间事件执行器运行时，它就遍历整个链表，查找所有已到达的时间事件，并调用相应的事件处理器。  
一种周期性的时间事件serverCron的主要工作包括：  
1. 更新服务器的各类统计信息，比如时间、内存占用、数据库占用情况等。  
2. 清理数据库中的过期键值对。  
3. 关闭和清理连接失效的客户端。  
4. 尝试进行AOF或RDB持久化操作。  
5. 如果服务器是主服务器，那么对从服务器进行定期同步。  
6. 如果处于集群模式，对集群进行定期同步和连接测试。  

以下是事件的调度和执行规则：  
1. aeApiPoll函数的最大阻塞时间由到达时间最接近当前时间的时间事件决定，这个方法既可以避免服务器对时间事件进行频繁的轮询（忙等待），也可以确保aeApiPoll函数不会阻塞过长时间。  
2. 因为文件事件是随机出现的，如果等待并处理完一次文件事件之后，仍未有任何时间事件到达，那么服务器将再次等待并处理文件事件。随着文件事件的不断执行，时间会逐渐向时间事件所设置的到达时间逼近，并最终来到到达时间，这时服务器就可以开始处理到达的时间事件了。  
3. 对文件事件和时间事件的处理都是同步、有序、原子地执行的，服务器不会中途中断事件处理，也不会对事件进行抢占，因此，不管是文件事件的处理器，还是时间事件的处理器，它们都会尽可地减少程序的阻塞时间，并在有需要时主动让出执行权，从而降低造成事件饥饿的可能性。比如说，在命令回复处理器将一个命令回复写入到客户端套接字时，如果写入字节数超过了一个预设常量的话，命令回复处理器就会主动用break跳出写入循环，将余下的数据留到下次再写；另外，时间事件也会将非常耗时的持久化操作放到子线程或者子进程执行。  
4. 因为时间事件在文件事件之后执行，并且事件之间不会出现抢占，所以时间事件的实际处理时间，通常会比时间事件设定的到达时间稍晚一些。  

## 第三部分    
### 主从复制  
SLVAEOF命令设置主从复制。主从服务器保存相同的数据，称为数据库状态一致。  
#### 主从复制的实现  
同步（sync）和命令传播（command propagate）两个操作：  
1. 同步操作用于将从服务器的数据库状态更新至主服务器当前所处的数据库状态。  
2. 命令传播操作则用于在主服务器的数据库状态被修改，导致主从服务器的数据库状态出现不一致时，让主从服务器的数据库重新回到一致状态。   

##### 同步之SYNC  
从服务器对主服务器的同步操作需要通过向主服务器发送SYNC命令来完成，以下是SYNC命令的执行步骤：
1. 从服务器向主服务器发送SYNC命令。  
2. 主服务器收到后执行BGSAVE命令，在后台生成一个RDB文件，并使用一个缓冲区记录从现在开始执行的所有写命令。  
3. 当主服务器的BGSAVE命令执行完毕时，主服务器会将BGSAVE命令生成的RDB文件发送给从服务器，从服务器接收并载入这个RDB文件。  
4. 从服务器载入完RDB文件后向主服务器发送信息。  
5. 主服务器收到信息后，将记录在缓冲区里面的所有写命令发送给从服务器，从服务器执行这些写命令，主从服务器实现状态一致。  

概括为从服务器向主发送命令；主生成RDB文件，使用缓冲区记录写命令；主向从发送RDB文件，从载入；主向从发送缓冲区里的命令，从执行，状态达到一致。  

##### 命令传播   
为维持主从服务器的一致性，主服务器会向从服务器发送自己执行的写命令，从服务器执行后主从一致。  

##### 仅使用SYNC同步与命令传播的问题  
SYNC命令非常耗费资源，在从服务器断线后重复制时效率低。  
SYNC命令生成RDB文件时大量占用CPU，内存和磁盘I/O资源，发送RDB文件时占用大量网络资源（影响相应命令请求），从服务器载入RDB文件时阻塞无法处理请求。  

##### 更高效的同步命令之PSYNC   
redis从2.8开始使用PSYNC命令代替SYNC。PSYNC有完整重同步和部分重同步两种模式：  
 - 其中完整重同步用于处理初次复制情况：完整重同步的执行步骤和SYNC命令的执行步骤基本一样，它们都是通过让主服务器创建并发送RDB文件，以及向从服务器发送保存在缓冲区里面的写命令来进行同步。  
 - 而部分重同步则用于处理断线后重复制情况：当从服务器在断线后重新连接主服务器时，如果条件允许，主服务器可以将主从服务器连接断开期间执行的写命令发送给从服务器，从服务器只要接收并执行这些写命令，就可以将数据库更新至主服务器当前所处的状态。  
  
##### 部分重同步的实现  
部分重同步功能由以下三个部分构成：1. 主服务器的复制偏移量（replication offset）和从服务器的复制偏移量。2. 主服务器的复制积压缓冲区（replication backlog）。3. 服务器的运行ID（run ID）。   

复制偏移量：主服务器每次向从服务器传播N字节数据时，就将自己的复制偏移量加上N。从服务器每次收到主服务器N字节时，将自己的偏移量加上N。因此处于一致状态的主从服务器偏移量总是相同。  

复制积压缓冲区：由主服务器维护的一个固定长度的队列（默认1MB）。当主服务器进行命令传播时会将写命令入队到此缓冲区，并记录每个字节记录相应的复制偏移量。  
当从服务器重新连接上主服务器时，会通过PSYNC命令将自己的复制偏移量offset发送给主服务器，主服务器会根据这个复制偏移量来决定对从服务器执行哪种同步：  
1. 如果offset偏移量之后的数据（也即是偏移量offset+1开始的数据）仍然存在于复制积压缓冲区里面，那么主服务器将对从服务器执行部分重同步操作。  
2. 相反，如果offset偏移量之后的数据已经不存在于复制积压缓冲区，那么主服务器将对从服务器执行完整重同步操作。  
可以将复制积压缓冲区的大小设为2 * reconnect_average_second * write_size_per_second。  

服务器运行ID：每个Redis服务器都有自己的运行ID（启动时随机生成地40位十六进制字符串）。  
从服务器初次与主服务器同步时，主服务器会将运行ID发送给从服务器，从服务器会保存主运行ID。  
从服务器断线重连主服务器时，会向当前主服务器发送之前保存的主运行ID。若和当前连接的主服务器的运行ID相同，主服务器继续尝试部分重同步。不相同时，主服务器将对从服务器执行完整重同步操作。  

##### 主从复制的步骤：    
1. 客户端执行SLAVEOF命令后，记录主服务器信息。  
	从服务器将主服务器IP及端口保存到服务器状态的masterhost和masterport中。SLAVEOF命令是异步命令，在完成第一步后从服务器将向客户端返回OK，复制工作在在OK返回之后才真正开始执行。    
2. 建立套接字连接。  
	成功连接的话，从服务器将为这个套接字关联用于处理复制工作的文件事件处理器（接受RDB文件，接受命令传播传来的写命令）。主服务器会为该套接字创建相应的客户端状态，并将从服务器看作是一个连接到主服务器的客户端来对待。（之后的步骤从服务器向主服务器发送命令请求）  
3. 发送PING命令。    
	从服务器成为主服务器的客户端之后，首先发送PING命令，用于检查套接字的读写状态及主服务器能否正常处理请求。从服务器收到PONG回复的话表示可以继续下一步骤。  
4. 身份验证。  
	根据从服务器是否设置masterauth选项，决定是否进行身份验证。密码错误，主设置了密码而从未设置，从设置了密码而主未设置，这三种情况均会发生错误，并从创建套接字开始重新执行复制，直到身份验证通过，或者从服务器放弃执行复制为止。  
5. 发送端口信息。  
	从服务器会向主服务器发送从服务器的监听端口号，主服务器会将端口号记录在从服务器所对应的客户端状态的slave_listening_port属性中。  
6. 同步。  
	从服务器将向主服务器发送PSYNC命令，执行同步操作。在同步操作执行之前，只有从服务器是主服务器的客户端，但是在执行同步操作之后，主服务器也会成为从服务器的客户端。完整重同步操作时，那么主服务器需要成为从服务器的客户端，才能将保存在缓冲区里面的写命令发送给从服务器执行。部分重同步操作时，那么主服务器需要成为从服务器的客户端，才能向从服务器发送保存在复制积压缓冲区里面的写命令。  
7. 命令传播。  
	这时主服务器只要一直将自己执行的写命令发送给从服务器，而从服务器只要一直接收并执行主服务器发来的写命令，就可以保证主从服务器一直保持一致了。  
	
#### 心跳检测  
在命令传播阶段，从服务器默认会以每秒一次的频率，向主服务器发送命令：REPLCONF ACK < replication_offset >。replication_offset是从服务器当前的复制偏移量。  

心跳检测的作用：  
1. 检测主从服务器的网络连接状态。  
	如果主服务器超过一秒钟没有收到从服务器发来的REPLCONF ACK命令，那么主服务器就知道主从服务器之间的连接出现问题了。（可通过客户端向主服务器发送INFO replication命令可查看从服务器相关信息）  
2. 辅助实现min-slaves选项。  
	Redis的min-slaves-to-write和min-slaves-max-lag两个选项可以防止主服务器在不安全（从服务器数量过少或连接情况不佳）的情况下执行写命令。
3. 检测命令丢失。  
	写命令在网络传播中丢失时，主服务器会发现从服务器的复制偏移量少于自己的，主服务器根据从服务器提交的复制偏移量，在复制积压缓冲区里面找到从服务器缺少的数据，并将这些数据重新发送给从服务器。（当3条命令中间那条丢失怎么确定？）（推测是与TCP的确认号机制相似，原书中并未说明如何确认）   
	
### Sentinel（哨兵）  
由一个或多个Sentinel实例（instance）组成的Sentinel系统（system）可以监视任意多个主服务器，以及这些主服务器属下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器，然后由新的主服务器代替已下线的主服务器继续处理命令请求。  

当主服务器下线时间超过用户设定的上限时，Sentinel系统就会对主服务器执行故障转移操作：
1. 首先，在原来的从服务器中选一个作为新的主服务器  
2. 之后，让其他原来的从服务器成为新的主服务器的从服务器  
3. 此外，监视原来的主服务器，在它重新上线时将其设置为新的主服务器的从服务器

对于每个被Sentinel监视的主服务器来说，Sentinel会创建两个连向主服务器的异步网络连接：一个是命令连接，这个连接专门用于向主服务器发送命令，并接收命令回复。另一个是订阅连接，这个连接专门用于订阅主服务器的__sentinel__:hello频道。   
Sentinel默认会以每十秒一次的频率，通过命令连接向被监视的主服务器发送INFO命令，并通过分析INFO命令的回复来获取主服务器的当前信息。   
监视同一个主服务器的多个Sentinel可以通过分析接收到的频道信息来获知其他Sentinel的存在，并通过发送频道信息来让其他Sentinel知道自己的存在。这些Sentinels互相之间会创建命令连接，从而形成互相连接的网络。   

#### 主观下线的判断   
sentinel每秒一次的频率向各已连接的实例发送PING命令，如果一个实例在down-after-milliseconds毫秒内，连续向Sentinel返回无效回复（除PONG、LOADING、MASTERDOWN外皆为无效），那么Sentinel会修改这个实例所对应的实例结构，在结构的flags属性中打开SRI_S_DOWN标识，以此来表示这个实例已经进入主观下线状态。   

#### 客观下线的判断   
当Sentinel将一个主服务器判断为主观下线之后，为了确认这个主服务器是否真的下线了，它会向同样监视这一主服务器的其他Sentinel进行询问，若从其他Sentinel那里接收到足够数量（Sentinel配置中设置的quorum参数的值）的已下线判断之后，Sentinel就会将从服务器判定为客观下线，并对主服务器执行故障转移操作。   

#### 对下线服务器的故障转移操作  
首先监视这个下线主服务器的各个Sentinel会进行协商，选举出一个领头Sentinel，并由领头Sentinel对下线主服务器执行故障转移操作。   

在选举产生出领头Sentinel之后，领头Sentinel将对已下线的主服务器执行故障转移操作，包含三个步骤。   

1. 选出新的主服务器。   
挑选出一个状态良好、数据完整的从服务器，然后向这个从服务器发送SLAVEOF no one命令，将这个从服务器转换为主服务器。   
新的主服务器如何挑选：   
领头Sentinel会将已下线主服务器的所有从服务器保存到一个列表里，从列表中过滤掉已下线的、最近五秒内未回复领头Sentinel的INFO命令的、与已下线主服务器连接断开超过down-after-milliseconds* 10毫秒的从服务器，从余下的从服务器中选出优先级最高的。如果有多个具有相同最高优先级的从服务器，选出其中偏移量最大的从服务器（即保存最新数据的服务器），偏移量相同则选出其中运行ID最小的从服务器。在发送SLAVEOF no one命令之后，领头Sentinel会以每秒一次的频率（平时是每十秒一次），向被升级的从服务器发送INFO命令，并观察命令回复中的角色（role）信息，当被升级服务器的role从原来的slave变为master时，领头Sentinel就知道被选中的从服务器已经顺利升级为主服务器了。   

2. 修改从服务器的复制目标。   
让已下线主服务器属下的所有从服务器去复制新的主服务器，这一动作可以通过向从服务器发送SLAVEOF命令来实现。   

3. 将旧的主服务器变为从服务器。   
当原来的主服务器重新上线时，Sentinel就会向它发送SLAVEOF命令，让它成为新的主服务器的从服务器。   

### 集群   
一个节点就是一个运行在集群模式下的Redis服务器，向一个节点node发送CLUSTER MEET命令，可以让node节点与ip和port所指定的节点进行握手（handshake），当握手成功时，node节点就会将ip和port所指定的节点添加到node节点当前所在的集群中。    

节点的信息通过Gossip协议传播给集群中的其他节点。    

Redis集群通过分片的方式来保存数据库中的键值对：集群的整个数据库被分为16384个槽（slot），数据库中的每个键都属于这16384个槽的其中一个，集群中的每个节点可以处理0个或最多16384个槽。当数据库中的16384个槽都有节点在处理时，集群处于上线状态（ok）；相反地，如果数据库中有任何一个槽没有得到处理，那么集群处于下线状态（fail）。   

clusterState结构中的slots数组记录了集群中所有16384个槽的指派信息，clusterNode.slots数组只记录了clusterNode结构所代表的节点的槽指派信息。   

向节点发送CLUSTER ADDSLOTS命令，我们可以将一个或多个槽指派（assign）给节点负责。   

##### 集群如何执行命令  
当客户端向节点发送与数据库键有关的命令时，接收命令的节点会计算出命令要处理的数据库键属于哪个槽，并检查这个槽是否指派给了自己。如果键所在的槽正好就指派给了当前节点，那么节点直接执行这个命令。如果键所在的槽并没有指派给当前节点，那么节点会向客户端返回一个MOVED错误，指引客户端转向（redirect）至正确的节点，并再次发送之前想要执行的命令。   

##### 计算键属于哪个槽
CRC16（key）& 16383
CLUSTER KEYSLOT < key > 查看键属于哪个槽。
其中CRC16（key）语句用于计算键key的CRC-16校验和，而&16383语句则用于计算出一个介于0至16383之间的整数作为键key的槽号。   

##### 节点数据库   
除了将键值对保存在数据库里面之外，节点还会用clusterState结构中的slots_to_keys跳跃表来保存槽和键之间的关系。slots_to_keys跳跃表每个节点的分值（score）都是一个槽号，而每个节点的成员（member）都是一个数据库键。通过跳表可以很方便地对某个或某些槽地键进行批量操作。   

#### 集群复制    
Redis集群中的节点分为主节点（master）和从节点（slave），其中主节点用于处理槽，而从节点则用于复制某个主节点（相当于主节点的从服务器），并在被复制的主节点下线时，代替下线主节点继续处理命令请求。     

设置从节点：向一个节点发送CLUSTER REPLICATE < node_id >命令。     

一个节点未在规定时间内恢复其他节点的PING，则被视为疑似下线（PFAIL），节点间会互相发送消息交换节点状态信息（下线报告）。如果在一个集群里面，半数以上负责处理槽的主节点都将某个主节点x报告为疑似下线，那么这个主节点x将被标记为已下线（FAIL），将主节点x标记为已下线的节点会向集群广播一条关于主节点x的FAIL消息，所有收到这条FAIL消息的节点都会立即将主节点x标记为已下线，并向集群广播一条关于此节点的FAIL消息。   

故障转移：    
当一个从节点发现自己正在复制的主节点进入了已下线状态时，从节点将开始对下线主节点进行故障转移，以下是故障转移的执行步骤：   
1. 复制下线主节点的所有从节点里面，会有一个从节点被选中。  
2. 被选中的从节点会执行SLAVEOF no one命令，成为新的主节点。  
3. 新的主节点会撤销所有对已下线主节点的槽指派，并将这些槽全部指派给自己。  
4. 新的主节点向集群广播一条PONG消息，这条PONG消息可以让集群中的其他节点立即知道这个节点已经由从节点变成了主节点，并且这个主节点已经接管了原本由已下线节点负责处理的槽。  
5）新的主节点开始接收和自己负责处理的槽有关的命令请求，故障转移完成 。  

新的主节点选举与选举领头Sentinel方法相似，都是基于Raft算法的领头选举（leader election）方法来实现。大概流程是：每轮中每个节点都会向最先要求自己投票的节点投票，每个节点每轮只能投一票，票数过半的节点当选，若没有选出则开始下一轮选举。（在选新的主节点时，投票为回复自己广播地CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST消息，回复内容为CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK）    

## 第四部分   
### 事务   
Redis通过MULTI、EXEC、WATCH等命令来实现事务（transaction）功能。在事务执行期间，服务器不会中断事务而改去执行其他客户端的命令请求，它会将事务中的所有命令都执行完毕，然后才去处理其他客户端的命令请求。   

#### 事务的阶段   
事务从开始到结束通常会经历三个阶段：1）事务开始。2）命令入队。3）事务执行。   

事务开始：
MULTI命令是事务的开始，将执行该命令的客户端从非事务状态切换至事务状态。
命令入队：
非事务状态的客户端发送的命令会立即被服务器执行。  
事务状态的客户端会立即执行EXEC、DISCARD、WATCH、MULTI四个命令，其他命令会放入事务队列。   
执行事务：
客户端向服务器发送EXEC命令，这个EXEC命令将立即被服务器执行。服务器会遍历这个客户端的事务队列，执行队列中保存的所有命令，最后将执行命令所得的结果全部返回给客户端。

#### WATCH命令  
WATCH命令是一个乐观锁（optimistic locking），它可以在EXEC命令执行之前，监视任意数量的数据库键，并在EXEC命令执行时，检查被监视的键是否至少有一个已经被修改过了，如果是的话，服务器将拒绝执行事务，并向客户端返回代表事务执行失败的空回复。  
每个Redis数据库都保存着一个watched_keys字典，这个字典的键是某个被WATCH命令监视的数据库键，而字典的值则是一个链表，链表中记录了所有监视相应数据库键的客户端。  
所有对数据库进行修改的命令，（SET、LPUSH、SADD、ZREM、DEL、FLUSHDB等），在执行之后都对watched_keys字典进行检查，查看是否有客户端正在监视刚刚被命令修改过的数据库键，如果有的话，那么touchWatchKey函数会将监视被修改键的客户端的REDIS_DIRTY_CAS标识打开，表示该客户端的事务安全性已经被破坏。   
客户端向服务器发送EXEC命令，若客户端的REDIS_DIRTY_CAS标识已被打开，那么服务器就拒绝执行此事务，若每被打开，就执行此事务。   

#### ACID   
Redis事务总具有原子性（Atomicity）、一致性（Consistency）和隔离性（Isolation），当Redis运行在某种特定的持久化模式下时，也具有耐久性（Durability）。   

notice： 
1. redis事务不支持回滚。一个事务中的某条命令执行错误并不影响其他命令的执行。  
2. 若事务在命令入队过程中若出现命令不存在或格式不正确，事务将不会被执行。  
3. Redis的事务总是以串行的方式运行的，在执行事务期间不会对事务进行中断。

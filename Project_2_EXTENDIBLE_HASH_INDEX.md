# Project 2 EXTENDIBLE HASH INDEX
## 前言
### EXTENDIBLE HASH INDEX 主要目标
1.  实现一个可扩展哈希表结构；
2. 结合Buffer Pool，我们的extendible hash index 也需要满足并发要求。

### 可扩展哈希简要介绍
* 定义：普通的静态哈希桶的数量是固定的，即使是在扩容时其桶数量也是近似二倍扩容的一个新的质数（参考STL hashtable底层实现），而可扩展哈希的**真实**桶数量是可以随意变化的（其实虚拟实现上仍是二倍关系）；
* 优势：
	- 避免哈希倍数扩容时的**瞬时**巨大开销，在并发情况下，静态哈希可能一段时间无法被访问；
	- 减少哈希扩容时桶内容的迁移。
* 实现的简要分析
	* Directory: 一个字典表，记录每个桶位置，同时记录一个global_depth_；
	* Bucket: 多个桶，每个桶记录多条实际数据，当桶内容满时，需要分裂桶，当桶内容空时，需要合并桶，同时记录一个local_depth_。
* 可扩展哈希的插入，删除，查询数据过程略微复杂，建议自行查找。

### EXTENDIBLE HASH INDEX 基础组件
1. hash_table_directory_page: 字典页；
2. hash_table_bucket_page: 桶页；
3. extendible_hash_table: 可扩展哈希框架，连接字典页，桶页，应用Buffer Pool 并且满足并发要求

---
## Part 1 hash_table_directory_page 具体实现
### directory 主要参数
* ```page_id_t page_id_ ``` : 页面表对应的页号；
* ```uint32_t global_depth_ ``` : 全局深度，用于查询和插入数据时选择桶；
* ```uint8_t local_depth_[]``` : 数组，记录每个桶对应的局部深度；
* ```page_id_t bucket_page_ids_ ```: 数组，记录每个桶对应的页号。

### directory 主要方法
- ```IncrGlobalDepth() ```: 页表内容（最大桶数量）二倍扩容，实际并不增加新桶，同时桶数组要新增指向复制，（例如：4到8个桶的扩容，4->0 5->1 6->2 7->3，这是位掩码从最低位开始取的结果）；
- ```GetSplitImageIndex(uint32_t) ```: 获取其镜像桶的索引，具体用途在之后的扩容中的桶内容迁移，实现按照注释完成即可，本桶的索引当前深度位(local_depth_)取反。

---
## Part 2 hash_table_bucket_page 具体实现
该模块需要大量的位运算，按部就班完成即可。

### bucket 主要参数
* ```char occupied_[] ``` : 没用上，不必纠结；
* ```char readable_[] ``` : 位图思想，记录桶中所有数据是否可读，可以的话对应位置为1，使用char数组而不是byte数组，效率快了将近8倍，但之后的位运算略复杂一些；
* ```MappingType array_[]``` : 实际记录kv数对的数组。

### bucket 主要方法
- ```IsFull() ```: 判断桶是否已满，注意桶容量不一定是8的整数倍，不能直接判断readable_每个char == 255，最后的剩余部分要按位依次判断；
- ```GetArrayCopy() ```: 新增方法，方便之后扩容桶数据迁移，复制所有数据（注：理论上调用此方法是桶一定是满的，因此就不需要复制，直接取array_数组指针即可，本人没有尝试）。

---
## Part 3 extendible_hash_table 具体实现
该项目的重头戏，连接字典和桶，同时使用读写锁尽最大能力满足并发要求，其中的桶分裂和桶合并方法要考虑的内容也比较多（坑比较多），之后也对这两个方法重点分析。

### hash_table 主要参数
* ```page_id_t directory_page_id_ ``` : 字典页对应页号；
* ```BufferPoolManager *buffer_pool_manager_ ``` : 缓存池管理工具，即我们的Part 1；
* ```ReaderWriterLatch latch_ ``` : 读写锁，除了桶分裂和桶合并两个方法涉及字典页面的指向更改需要使用写锁外，其它方法都只需要读锁即可，使锁的粒度尽可能小；

### hash_table 主要方法
* ```ExtendibleHashTable() ``` : 构造函数，在初始化创建一个字典页，此时```global_depth = 0``` ，因此只需要创建一个桶页即可；
* ```FetchDirectoryPage() ``` : 从内存中抓取字典页面，并开始使用，注意调用者最后要记得**Unpin住**该页面；
* ```FetchBucketPage() ``` : 从内存中抓取桶页面，其余同上；
* ```Insert(Transaction *, KeyType , ValueType) ``` : 插入数据对，如果桶未满直接插入即可，否则调用```SplitInsert() ```；
* ```Remove(Transaction *, KeyType, ValueType) ``` : 删除数据对，如果桶未空直接删除即可，否则调用```Merge()```；
* ```SplitInsert(Transaction *, KeyType, ValueType) ``` : 我的实现只是桶分裂，分裂后要再次调用```Insert()``` 完成插入，该函数的主要流程如下：
	* 在函数开始时，需要再次判断桶是否为满，若不满，不分裂了，直接插入；
	* 创建一个镜像桶```GetSplitImageIndex()```，取出桶中所有原有数据，重新分配至两个桶中；
	* 分裂完成，再次调用```Insert()```。
* ```Merge(Transaction *, KeyType, ValueType) ``` : 桶合并，该函数的主要流程如下：
	* 在函数开始时判断局部深度是否为0，若为0不能合并；
	* 判断其镜像桶的局部深度是否相同，若不相同不能合并；
	* 再次判断桶是否为空，若不为空，不合并了（同上）；
	* 将所有指向删除桶的字典页项全部指向其镜像桶；
	* 循环缩减全局深度。

---
## 代码亮点 & 踩坑
### 比较好的细节
* 在位图部分，位运算不使用byte数组，而是使用char数组，效率快了将近8倍；
* ```global_depth_``` 从最低位开始向高位取，这样在寻找其镜像桶时好算一些，但是数据内容不再是桶排序了；
* 使用读写锁，除了桶分裂和合并时，全部使用读锁，粒度更小，并发更大。

### 踩坑
* ```SplitInsert()``` 开始时要重新判断桶是否满了，因为我们把插入和桶分裂拆成了两个函数，由于并发中间过程可能会有数据删除，就不再需要分裂（不然有用例不能通过）；
* 桶分裂，合并细节很多，要考虑的也很多，按照注释一步步来。

---
## Score
![[project 2 score.png]]
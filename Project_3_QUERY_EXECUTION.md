# Project 3 QUERY EXECUTION
## 前言
### QUERY EXECUTION 实现目标
1. SQL语句（执行器）的实现，包括SELECT, INSERT, DELETE, UPDATE, JOIN, HASH JOIN, AGGREGATION, LIMIT, DISTINCT；
2. 该项目暂时不需要考虑并发问题，每个组件实现其```Init(), Next()``` 即可；
3. 虽然项目总体难度不如Project 2，但是里面涉及到许多组件，需要先对组件有足够的理解；

### Catalog
该组件为数据库存储引擎中最大的组件，维护表信息和索引信息

### TableInfo 核心组件
该组件用来存储一个表的主要信息。
* ```Schema schema_ ``` : 记录表的结构，即各个属性；
* ```std::unique_ptr<TableHeap> table_ ``` : 指向TableHeap的指针，之后会详细阐述；
* ```const table_oid_t oid_ ``` : 唯一标识表。

### TableHeap 核心组件
该组件用来存储表的所有数据记录，并且维护迭代器，锁，缓存池等。
* ```pgae_id_t first_page_id_ ``` : 数据库表内容对应的页号；
* ```BufferPoolManager *buffer_pool_manager_ ``` : 缓存池管理工具。

### Tuple
该组件用来存储表的一条记录（数据行），其最终存储在页面（TableHeap的页面）当中；
```rid ``` : 唯一标识表中的一条记录。

### Executor 相关
执行器相关的实现使用了**工厂模式**，外部调用使用如下接口：
```
Execute(const AbstractPlanNode *plan, std::vector<Tuple> *result_set, Transaction *txn, ExecutorContext *exec_ctx)
// 其中，plan 作为抽象节点用来标识具体执行哪种语句(select insert ...)，result_set针对需要返回表记录的语句，若不需要返回，传递nullptr即可
```

---
## Seqential Scan
### Seqential Scan主要方法
```Init() ``` : 初始化表迭代器和表数据内容(TableHeap)即可；
```Next() ``` : **方法是返回一个满足条件的记录，不是全部记录，** 先从表数据内容中返回一条记录，之后调用谓词迭代器，若不满足条件，再次调用，注意迭代器总是指向将要输出(未输出)的记录。

## Insert
### Insert 主要方法
```Init() ``` : 初始化存储目录，表信息和表数据内容；
```Next() ``` : 先判断有无子计划，若有先执行子计划，最后还要更新索引，注意：它一次性完成全部操作，不需要循环，因此总是返回false。

## Delete
### Delete 主要参数
```std::unique_ptr<AbstractExecutor> child_executor_ ``` : 内置一个扫描子执行器，因此在执行删除时，先使用扫描执行器找到要删除的记录，之后进行删除（组合思想）。

## Aggregation
核心组件，简单聚合哈希，里边的实现已经写好了，只要研究明白调用即可（老师写的真好）

### SimpleAggregationHashTable 主要参数
* ```std::unordered_map<AggregakeKey, AggregateValue> ht_{} ``` : 哈希表，键按照group by进行划分，值是一个数组，**每一项表示不同聚合函数的计算结果**；
* ```const std::vector<AggregationType> &agg_types_ ``` : 聚合函数类型数组，对应上述哈希表的值数组的聚合函数类型。

### Aggregation 主要参数
* ```const AggregationPlanNode *plan_ ``` : 计划节点，同上；
* ```std::unique_ptr<AbstractExecutor> child_ ```  : 子执行器，查询功能，同样类似于Insert的子执行器；
* ```SimpleAggregationHashTable aht_ ``` : 核心组件，简单聚合哈希；
* ```SimpleAggregationHashTable::Iterator aht_iterator_ ``` : 哈希迭代器。

### Aggregation 主要方法
* ```Init() ``` : 初始化子查询执行器，并使用其将所有记录放入按group by分类的简单聚合哈希当中，**注意这里每个group by的聚合函数结果都已经算好了**；
* ```Next() ``` : 只是筛选，依次取出每个group by的聚合函数结果（仍是每次函数调用只取一个），遇到不符合Having的直接跳过。

## Nested_loop_join
嵌套循环连接，这里是带谓词条件的全外连接，比较简单

### Nested_loop_join 主要参数
* ```std::unqiue_ptr<AbstractExecutor> left_executor_ ``` : 左连接表的查询子执行器；
* ```std::unqiue_ptr<AbstractExecutor> right_executor_ ``` : 右连接表的查询子执行器；
* ```std::vector<Tuple> result_ ``` : 连接的结果记录数组；
* ```uint32_t now_id_ ``` : 结果记录的当前索引下标。

### Nested_loop_join 主要方法
* ```Init() ``` : 两个子执行器的双重循环，注意谓词条件。

## Hash Join
哈希连接，其特点如下：
* 哈希连接不一定会被排序；
* 哈希连接只适用于等值连接；
* 哈希连接保证驱动表和被驱动表所有记录最多只被访问一次；
* 嵌套循环连接时间复杂度 O(m * n)，哈希连接时间复杂度 O(m + n)。

### Hash Join 主要参数
* ```std::unique_ptr<AbstractExecutor> left_child_executor_ ``` : 左连接表的子执行器；
* ```std::unique_ptr<AbstractExecutor> right_child_executor_ ``` : 右连接表的子执行器；
* ```std::unordered_map<HashJoinKey, std::vector<Tuple>> map_ ``` : 记录驱动表（左表）的所有元组经过哈希后的结果，注意这里需要自己写HashJoinKey的哈希函数；
* ```uint32_t now_id_ ``` : 结果记录的当前索引下标。

### Hash Join 主要方法
* ```Init() ``` : 实质就是哈希连接的原理，将驱动表的所有元组保存到哈希表，再次使用子执行器遍历被驱动表，找到哈希表对应元组，连接，存储。

---
## 总结
本项目阅读代码的工作量远大于书写代码，这里建议先从Test代码开始看，一步一步看向底层代码，可以学到很多东西，最让我惊艳的Aggregation和Hash join的实现。

---
## Score
这里我上传几次都是clang-tidy风格不符合，我看网上有相应的解决方案，不过好像也无效了，而且我Project 4 也通过了，就不提交了。
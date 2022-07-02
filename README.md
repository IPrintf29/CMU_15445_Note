# 前言
## 项目简介
CMU 15-445 课程讲述了数据库管理系统的设计和实现，课程非常注重编程实践，设计了一套前后连贯的代码实验，其中最底层的实现老师都已经写好了，通过实验不仅会学习到数据库设计特点，更为重要的是其实现原理。

注：应课程老师要求，本篇文章不会发布代码，重点讲解实验过程中的基础实现以及踩过的坑（附带课程分数）。

---
## 实验内容
CMU 15-445 (2021) 实验课分为以下四个部分：
* BUFFER POOL，为避免频繁访问磁盘，设计一个结合LRU淘汰机制的缓存池；
* EXTENDIBLE HASH INDEX，优化记录查询等操作，手写可扩展哈希索引（结合了哈希和B+树的部分优点）；
* QUERY EXECUTION，采用工厂模式，实现了9种常见的数据库执行指令（查询，更新，插入，连接，聚合等）；
* CONCURRENCY CONTROL，锁管理器，实现了RR，RC，RU三种不同隔离级别下的加锁机制，并且具备死锁预防功能。

此外，除了造轮子之外，还必须具备一些关于实现的基本原理，包括可扩展哈希，LRU，两阶段锁协议，死锁预防策略等，并且动手实现一遍能够更深入地理解。

---
## 目录
| Index     |  Link                  |         Content                 |
|-----------|:----------------------:|------------------------------------:|
| Project 1 | [Project 1 BUFFER POOL](./Project_1_BUFFER_POOL.md)  |      缓存池，LRU      |
| Project 2 | [Project 2 EXTENDIBLE HASH INDEX](./Project_2_EXTENDIBLE_HASH_INDEX.md)  | 可扩展哈希 |
| Project 3 | [Project 3 QUERY EXECUTION](./Project_3_QUERY_EXECUTION.md)         | 执行器，嵌套循环连接，哈希连接       |
| Project 4 | [Project 4 CONCURRENCY CONTROL](./Project_4_CONCURRENCY_CONTROL.md)   | 并发控制，2PL，隔离级别   |

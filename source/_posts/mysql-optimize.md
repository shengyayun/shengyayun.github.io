---
title: mysql配置与运行状态的分析与优化
date: 2017-08-15 19:38:10
tags: mysql
---


## 指令：

mysql> show variables;//查看mysql具体配置

mysql> show global status;//查看mysql运行状态

配置是写在my.cnf里面的，而状态是具体执行时的指标


## 慢查：

#### 配置：

slow_launch_time：2 //执行时间超过2秒就会被记录为慢查

slow_query_log：ON //慢查会被记录

slow_query_log_file：iZ23qcdbhgiZ-slow.log //慢查日志文件名

#### 状态：

slow_launch_threads：0 //创建时间超过slow_launch_time秒的线程数

slow_queries：0 //慢查个数



## 最大连接数：

#### 配置：

max_connections：256 //允许的最大连接数

#### 状态：

max_used_connections：123 //实际运行中的最大连接数

#### 期望：

max_used_connections / max_connections * 100% ≈ 85%


## 索引缓存(对MyISAM性能影响很大)

#### 配置：

key_buffer_size：134217728 //索引缓存大小

#### 状态：

key_read_requests：123 //索引读取请求个数

Key_reads： 123 //没有命中缓存而去硬盘读取的次数

Key_blocks_unused：123 //未使用的缓存簇(blocks)数

Key_blocks_used： //曾经用到的最大的blocks数

#### 期望：

key_reads / key_read_requests * 100% < 0.1%

key_blocks_used / (key_blocks_unused + key_blocks_used) * 100% ≈ 80%


## 临时表

#### 配置：

max_heap_table_size：16777216 //内存表的最大大小

tmp_table_size：16777216 //内存临时表最大大小


#### 状态：

Created_tmp_disk_tables：0 //创建(内存或磁盘)临时表的次数

Created_tmp_files：6 //临时文件个数 

Created_tmp_tables：22 //创建磁盘临时表的次数

#### 期望：

理想#### 状态：created_tmp_disk_tables / created_tmp_tables * 100% <= 25%

备注：

当查询需要使用临时表的时候，系统会优先创建内存临时表(MEMORY引擎)。但如果内存临时表大小超过了max_heap_table_size或者tmp_table_size，它就会转为磁盘临时表(MyISAM引擎)。


## 表缓存

#### 配置：

table_cache(table_open_cache)： 3//缓存的打开的表的数量

#### 状态：

open_tables：1 //打开表的数量

opened_tables： 2//表示打开过的表数量

#### 期望：

Open_tables / Opened_tables * 100% >= 85%

Open_tables / table_cache * 100% <= 95%



## 线程缓存

#### 配置：

thread_cache_size：13 //服务器缓存的用来处理用户查询的线程

#### 状态：

threads_created：12 //表示创建过的线程数


## 文件打开数

#### 配置：

open_files_limit：1024 //允许同时打开的最大文件数

#### 状态：

open_files：123 //当前打开的文件数

#### 期望：

open_files / open_files_limit * 100% <= 75％


## 表锁

#### 状态：

table_locks_immediate： 123//立即释放表锁数

table_locks_waited： 456//需要等待的表锁数

#### 期望：

table_locks_immediate / table_locks_waited > 5000 时推荐使用高并发更新性能好的innodb引擎

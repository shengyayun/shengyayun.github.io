---
title: mysql表占用空间分析
date: 2019-07-25 14:12:25
tags: mysql
---

```sql
SELECT 
    table_schema, #库名
    table_name, #表名
    table_comment, #表注释
    `engine`, #表引擎
    ROUND(data_length / 1024 / 1024, 2) AS 'Data',  #数据占用(MB)
    ROUND(data_free / 1024 / 1024, 2) AS 'Free', #碎片占用(MB)
    ROUND(index_length / 1024 / 1024, 2) AS 'Index', #索引占用(MB)
    ROUND((data_length + data_free + index_length) / 1024 / 1024, 2) AS 'Total' #合计占用
FROM
    information_schema.tables
WHERE
    table_schema NOT IN ('mysql' , 'sys', 'information_schema', 'performance_schema')
ORDER BY data_length + data_free + index_length DESC;
```
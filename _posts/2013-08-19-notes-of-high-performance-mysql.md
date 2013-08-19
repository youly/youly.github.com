---
layout: post
title: 《High Performance Mysql》学习笔记
category: 笔记
tag: [mysql]
---

1. Any operation that changes an innodb table's structure will rebuild the entire table,including all the indexes;

2.InnoDB tables are built on a clustered index,As a result, it provides very fast primary key lookups. How- ever, secondary indexes (indexes that aren’t the primary key) contain the primary key columns, so if your primary key is large, other indexes will also be large.

3.Simply put, a phantom read can happen when you select some range of rows, another transaction inserts a new row into the range, and then you select the same range again; you will then see the new “phantom” row.

4.A benchmark measures your system’s perfor- mance. This can help determine a system’s capacity, show you which changes mat- ter and which don’t, or show how your application performs with different data.In contrast, profiling helps you find where your application spends the most time or consumes the most resources. In other words, benchmarking answers the question “How well does this perform?” and profiling answers the question “Why does it perform the way it does?”.

5.Choosing Optimal Data Types:Smaller is usually better,Simple is good,Avoid NULL if possible.It’s harder for MySQL to optimize queries that refer to nullable columns, because they make indexes, index statistics, and value comparisons more com- plicated. A nullable column uses more storage space and requires special pro- cessing inside MySQL.

6.MySQL lets you specify a “width” for integer types, such as INT(11). This is mean- ingless for most applications: it does not restrict the legal range of values, but simply specifies the number of characters MySQL’s interactive tools (such as the command- line client) will reserve for display purposes. For storage and computational pur- poses, INT(1) is identical to INT(20).

7.DECIMAL was just a storage format; DECIMAL numbers were converted to DOUBLEs for computational purposes.Because of the additional space requirements and computational cost, you should use DECIMAL only when you need exact results for fractional numbers—for example, when storing financial data.

8.In MySQL, the simplest query cost metrics are:
  
  • Execution time

  • Number of rows examined

  • Number of rows returned
没有预写日志（WAL）的表。

创建或者修改现有的表为无日志表：

```sql
CREATE UNLOGGED TABLE foobar (id int);

ALTER TABLE mytable SET UNLOGGED; -- cheap!
ALTER TABLE mytable SET LOGGED; -- expensive!
```

优点：

- 写入很快，差不多是有日志表的2-3倍。
- 占用磁盘空间更小，因为不需要记录WAL。

缺点：

- 崩溃重启后表的所有数据会被清空（正常restart不会清空）
- 只能在主节点访问
- 建不了GiST索引，但可以使用btree和gin。
- 一些备份工具会跳过无日志表。

使用场景：缓存一些只要在主节点访问且丢失了也没关系的数据。

# MySQL 删除数据后，为什么没有提升查询效率？

MySQL 作为一款广泛使用的关系型数据库，已经被应用于众多大型应用系统中。随着系统运行，数据库表的数据量也会变得越来越庞大。在这种情况下，对于 MySQL 数据库进行数据维护和性能优化就变得愈发重要。

而在实际使用中，使用 DELETE 命令删除数据后，我们发现查询速度并没有显著提高，甚至可能会降低。为什么？

这是因为 DELETE 命令只是标记该行数据为“已删除”状态，并不会立即释放该行数据在磁盘中所占用的存储空间，这样就会导致数据文件中存在大量的空洞与碎片，从而影响查询性能。



例如，我们根据表格字段进行查询，如下所示：

```mysql
mysql> EXPLAIN SELECT * FROM user WHERE gender = 1;
```

结果：

```
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows  |
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+
|  1 | SIMPLE      | user  | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 10000 |
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+
```

从上述查询的执行计划（EXPLAIN）可以看出，MySQL将使用全表扫描的方式获取所需的数据，即使gender字段有索引，但MySQL也不会使用该索引，因为索引被DELETE命令所删除的数据所占据。

接下来进行DELETE命令操作。例如：

```mysql
mysql> DELETE FROM user WHERE gender = 1;
```

从上述SQL语句执行后，我们还可以使用EXPLAIN命令再次查询执行计划：

```mysql
mysql> EXPLAIN SELECT * FROM user WHERE gender = 1;
```

结果：

```
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows  |
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+
|  1 | SIMPLE      | user  | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 10000 |
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+
```

从执行计划可以发现，MySQL仍然会使用全表扫描的方式获取要查询的数据，这意味着删除操作之后的表格查询性能并没有得到有效提升。原因在于该DELETE操作之后，数据文件中仍然存在大量的空洞和碎片，因此，即使表格中的数据量已经减少，查询速度也不会有显著的改善。

为了解决这个问题并提高查询性能，我们可以使用OPTIMIZE TABLE命令清理表格碎片，优化表格的磁盘空间布局。例如：

```mysql
mysql> OPTIMIZE TABLE user;
```

执行过OPTIMIZE TABLE之后，可使用EXPLAIN命令验证执行计划的改变。结果：

```
+----+-------------+-------+------------+------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | user  | NULL       | ref  | idx_gender    |idx_gender | 4      | const | 4999 |    50.00 | NULL  |
+----+-------------+-------+------------+------+---------------+---------+---------+-------+------+----------+-------+
```

从执行计划可以发现，MySQL将使用索引查找的方式获取所需的数据，查询性能得到了提升。

综上所述，DELETE命令只是将数据标记为删除而并不会真正删除数据，这样会导致数据文件中存在大量的空洞和碎片。因此，在使用DELETE命令删除数据后，我们需要考虑清理表格的碎片和空洞，采用OPTIMIZE TABLE命令将表格重新组织，以填补空洞和缩减碎片。通过这样的操作可以提高表格的查询性能和优化存储空间。需要注意的是，OPTIMIZE TABLE命令通常需要一段时间来执行，也可能需要额外的磁盘空间，因此在执行之前需要进行备份和检查。另外，如果表格主键是自增类型，执行OPTIMIZE TABLE命令时可能会导致主键ID值的断裂，需要小心处理。

总体来说，删除操作会对表格查询性能产生影响，需要正确清理表格碎片和空洞。为了避免影响查询性能，最好可以利用索引来避免全表扫描的情况出现，同时还需要定期清理表格的碎片和空洞，以保持MySQL数据库良好的性能和稳定性。

## 实例来验证一下

我们有一张表：courier_consume_fail_message。在表碎片清理前，我们关注以下指标。

1. 表的状态：`SHOW TABLE STATUS LIKE 'courier_consume_fail_message';`
2. 表的实际行数：`SELECT count(*) FROM courier_consume_fail_message;`
3. 要清理的行数：`SELECT count(*) FROM courier_consume_fail_message where created_at < '2023-04-19 00:00:00';`
4. 表查询的执行计划：`EXPLAIN SELECT * FROM courier_consume_fail_message WHERE service='courier-transfer-mq';`

第二步，执行删除操作：

```sql
DELETE FROM courier_consume_fail_message WHERE created_at < '2023-04-19 00:00:00';
```

第三步，清理碎片：

```sql
OPTIMIZE TABLE courier_consume_fail_message;
```

经过上面语句的执行，汇总一下 `OPTIMIZE TABLE courier_consume_fail_message;` 执行前后的结果。

### 清理前

第一，表的状态：

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230423093331751.png)

第二，表的实际行数：76986

第三，要清理的行数：76813

第四，表查询的执行计划：

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230423094157266.png)

### 清理数据

下面是执行 `DELETE FROM courier_consume_fail_message WHERE created_at < '2023-04-19 00:00:00';` 后的统计。

第一，表的状态：

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230423094647590.png)

第二，表的实际行数：173

第三，要清理的行数：0

第四，表查询的执行计划：

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230423094758811.png)

### 清理碎片

下面是执行 `OPTIMIZE TABLE courier_consume_fail_message;` 后的统计。清理碎片耗时：2.268s

第一，表的状态：

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230423095111282.png)

第二，表查询的执行计划：

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230423095201979.png)

## 生产环境的执行结果

### 清理前

第一，表的状态：

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230423091617646.png)

第二，表的实际行数：1366415

第三，要清理的行数：1363819

第四，表查询的执行计划：

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230423092905316.png)

### 清理数据

### 清理碎片
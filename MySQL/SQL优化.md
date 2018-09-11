## SQL优化

1. 对查询进行优化，应尽量避免全表扫描，首先应考虑在where及order by涉及的列上建立索引。

2. 应尽量避免在where字句中对字段进行null值判断，否则将导致引擎放弃使用索引而进行全表扫描，如：select id from t where num is null可以在num上设置默认值0,确保表中num列没有null值，然后这样查询：select id from t where num = 0。

3. 应尽量避免在where子句中使用 !=或<>操作符，否则引擎将放弃使用索引而进行全表扫描。

4. 应尽量避免在where子句中使用or来连接条件，否则将导致引擎放弃使用索引而进行全表扫描，如：

   ```sql
   select id from t where num = 10 or num = 20
   ```

   可以这样查询：

   ```sql
   select id from t where number = 10 union all select id from t where num = 20
   ```

5. in 和 not in也要慎用，否则会导致全表扫描，如：

   ```sql
   select id from t where num in(1,2,3) 对于连续的数值，能用between就不要用in了：select id from t where num between 1 and 3
   ```

6. 下面的查询也将导致全表扫描：

   ```sql
   select id from t where name like '李%'
   ```

   若要提高效率，可以考虑全文检索。

7. 如果在where子句中使用参数，也会导致全表扫描。因为SQL只有在运行时才会解析局部变量，但优化程序不能将访问计划的选择推迟到运行时；它必须在编译时继续选择。然而，如果在编译时建立访问计划，变量的值还是未知的，因而无法作为索引选择的输入项。如下面语句将进行全表扫描：

   ```sql
   select id from t where num=@num
   ```

   可以改为强制查询使用索引：

   ```sql
   select id from t with(idex(索引名)) where num=@num
   ```

8. 应尽量避免在where子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。如：

   ```sql
   select id from num/2 = 100 应改为： select id from t where num = 100*2
   ```

9. 
## mysql执行计划参数详解

1. ID：MySQL Query Optimizer选定的执行计划中查询的序列号。

2. Select_type：所使用的查询类型，主要有以下这几种查询类型。

   + DEPENDENT SUBQUERY：子查询内层的第一个SELECT，依赖于外部查询的结果集。
   + DEPENDENT UNION：子查询中的UNION，且为UNION中从第二个SELECT开始的后面所有SELECT，同样依赖于外部查询的结果集。
   + PRIMARY：子查询中的最外层查询，注意并不是主键查询。
   + SIMPLE：除子查询或UNION之外的其他查询。
   + SUBQUERY：子查询内层查询的第一个SELECT，结果不依赖于外部查询结果集。
   + UNCACHEABLE SUBQUERY：结果集无法缓存的子查询。
   + UNION：UNION语句中第二个SELECT开始后面的所有SELECT，第一个SELECT为PRIMARY。
   + UNION RESULT：UNION中的合并结果。

3. Table：显示这一步所访问的数据库中的表的名称。

4. Type：告诉我们对表使用的访问方式，主要包含如下集中类型。

   + all：全表扫描。
   + const：读常量，最多只会有一条记录匹配，由于是常量，实际上只须要读一次。
   + eq_ref：最多只会有一条匹配结果，一般是通过主键或唯一键索引来访问。
   + fulltext：进行全文索引检索。
   + index：全索引扫描。
   +  index_merge：查询中同时使用两个（或更多）索引，然后对索引结果进行合并（merge），再读取表数据。
   + index_subquery：子查询中的返回结果字段组合是一个索引（或索引组合），但不是一个主键或唯一索引。
   +  rang：索引范围扫描。
   + ref：Join语句中被驱动表索引引用的查询。
   + ref_or_null：与ref的唯一区别就是在使用索引引用的查询之外再增加一个空值的查询。
   + system：系统表，表中只有一行数据。
   + unique_subquery：子查询中的返回结果字段组合是主键或唯一约束。

5. Possible_keys：该查询可以利用的索引。如果没有任何索引可以使用，就会显示成null，

   这项内容对优化索引时的调整非常重要。

6. Key：MySQL Query Optimizer从possible_keys中所选择使用的索引。

7. Key_len：被选中使用索引的索引键长度。

8. Ref：列出是通过常量（const），还是某个表的某个字段（如果是join）来过滤（通过key）的。

9. Rows：MySQL Query Optimizer通过系统收集的统计信息估算出来的结果集记录条数。

10. Extra：查询中每一步实现的额外细节信息，主要会是以下内容。

    + Distinct：查找distinct值，当mysql找到了第一条匹配的结果时，将停止该值的查询，转为后面其他值查询。
    + Full scan on NULL key：子查询中的一种优化方式，主要在遇到无法通过索引访问null值的使用。
    + Impossible WHERE noticed after reading const tables：MySQL Query Optimizer通过收集到的统计信息判断出不可能存在结果。
    + No tables：Query语句中使用FROM DUAL或不包含任何FROM子句。
    + Not exists：在某些左连接中，MySQL Query Optimizer通过改变原有Query的组成而使用的优化方法，可以部分减少数据访问次数。
    + Range checked for each record (index map: N)：通过MySQL官方手册的描述，当MySQL Query Optimizer没有发现好的可以使用的索引时，如果发现前面表的列值已知，部分索引可以使用。对前面表的每个行组合，MySQL检查是否可以使用range或index_merge访问方法来索取行。
    + SELECT tables optimized away：当我们使用某些聚合函数来访问存在索引的某个字段时，MySQL Query Optimizer会通过索引直接一次定位到所需的数据行完成整个查询。当然，前提是在Query中不能有GROUP BY操作。如使用MIN()或MAX()的时候。
    + Using filesort：当Query中包含ORDER BY操作，而且无法利用索引完成排序操作的时候，MySQL Query Optimizer不得不选择相应的排序算法来实现。
    + Using index：所需数据只需在Index即可全部获得，不须要再到表中取数据。
    + Using index for group-by：数据访问和Using index一样，所需数据只须要读取索引，当Query中使用GROUP BY或DISTINCT子句时，如果分组字段也在索引中，Extra中的信息就会是Using index for group-by。
    + Using temporary：当MySQL在某些操作中必须使用临时表时，在Extra信息中就会出现Using temporary。主要常见于GROUP BY和ORDER BY等操作中。
    + Using where：如果不读取表的所有数据，或不是仅仅通过索引就可以获取所有需要的数据，则会出现Using where信息。
    + Using where with pushed condition：这是一个仅仅在NDBCluster存储引擎中才会出现的信息，而且还须要通过打开Condition Pushdown优化功能才可能被使用。控制参数为engine_condition_pushdown。
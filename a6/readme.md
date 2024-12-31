# 一、openGauss 执行计划的查看与分析
## 实验步骤
1. **查询1**： 查询零部件表中零售价小于920，且供应商key为5的零件key。
2. **查询2**： 查询客户表中账户余额小于1000且客户国家为ALGERIA的客户名称和电话。

## 实验代码
```sql
-- 查询1
EXPLAIN  
SELECT p_partkey  
FROM part  
WHERE p_retailprice < 920 AND p_partkey IN (SELECT ps_partkey FROM partsupp WHERE ps_suppkey = 5);

-- 查询2
EXPLAIN  
SELECT c_name, c_phone  
FROM customer  
WHERE c_acctbal < 1000 AND c_nationkey = (SELECT n_nationkey FROM nation WHERE n_name = 'ALGERIA');
```

---

## 执行计划
### 查询1
#### 执行计划
```sql
QUERY PLAN
------------------------------------------------------------------------------------------
 Nested Loop Semi Join  (cost=0.00..1999.07 rows=78 width=4)
   ->  Seq Scan on part  (cost=0.00..1352.00 rows=281 width=4)
         Filter: (p_retailprice < 920::numeric)
   ->  Index Only Scan using partsupp_pkey on partsupp  (cost=0.00..7.19 rows=21 width=4)
         Index Cond: ((ps_partkey = part.p_partkey) AND (ps_suppkey = 5))
(5 rows)
```

#### 关系代数的实现方式
1. 使用了半连接（Semi Join），结合了顺序扫描和索引扫描，适合在数据量较大的情况下优化性能。通过索引扫描`partsupp`表，减少了需要扫描的数据行数。

#### 执行成本
1. 外层扫描`part`表的成本：`0.00..1352.00`
2. 内层索引扫描`partsupp`表的成本：`0.00..7.19`
3. 总体成本：`0.00..1999.07`，表示该查询的执行成本相对较低，主要是因为`partsupp`表上的索引扫描很高效。

---

### 查询2
#### 执行计划
```sql
QUERY PLAN
----------------------------------------------------------------
 Seq Scan on customer  (cost=1.31..1170.31 rows=5495 width=35)
   Filter: ((c_acctbal < 1000::numeric) AND (c_nationkey = $0))
   InitPlan 1 (returns $0)
     ->  Seq Scan on nation  (cost=0.00..1.31 rows=1 width=4)
           Filter: (n_name = 'ALGERIA'::bpchar)
(5 rows)
```

#### 关系代数的实现方式
1. 使用了两次顺序扫描：一次扫描`nation`表查找`ALGERIA`，然后对`customer`表进行扫描，利用内层查询返回的`nationkey`过滤外层结果。由于没有合适的索引，这个查询的执行效率较低。

#### 执行成本
1. 外层扫描`customer`表的成本：`cost=1.31..1170.31`
2. 内层扫描`nation`表的成本：`cost=0.00..1.31`
3. 总体成本：`cost=1.31..1170.31`，这个查询的总体成本较高，主要是由于`customer`表的大量数据需要扫描。

---

# 二、观察视图查询和 WITH 临时视图查询的执行计划
## 实验步骤
1. 在 `nation` 表上创建视图 `nation_view`。
2. 使用 WITH 临时视图 `nation_tempview`。
3. 比较视图、WITH 临时视图和直接查询的执行计划和执行时间。

## 实验代码
```sql
-- 创建视图
CREATE VIEW nation_view AS
SELECT * FROM nation;

-- 通过视图查询
EXPLAIN ANALYZE
SELECT n_name FROM nation_view;

-- 使用 WITH 临时视图查询
EXPLAIN ANALYZE
WITH nation_tempview AS (
    SELECT * FROM nation
)
SELECT * FROM nation_tempview;

-- 直接查询
EXPLAIN ANALYZE  
SELECT n_name  
FROM nation;
```

---

## 执行计划
###  通过视图查询 
#### 执行计划
```sql
QUERY PLAN
----------------------------------------------------------------------------------------------------
 Seq Scan on nation  (cost=0.00..1.25 rows=25 width=104) (actual time=0.009..0.015 rows=25 loops=1)
 Total runtime: 0.070 ms
```

#### 分析
1. **执行方式：**顺序扫描 `nation` 表。
2. **成本：**`cost=0.00..1.25`，表扫描效率较高。
3. **总运行时间：**`0.070 ms`，性能较优。
4. **结论：**对于`nation`表这种数据量较小的表，顺序扫描是非常高效的，因此即使是通过视图查询，执行时间也非常快。

---

### 使用 WITH 临时视图查询
#### 执行计划
```sql
QUERY PLAN
-------------------------------------------------------------------------------------------------------------
 CTE Scan on nation_tempview  (cost=1.25..1.75 rows=25 width=434) (actual time=0.024..0.047 rows=25 loops=1)
   CTE nation_tempview
     ->  Seq Scan on nation  (cost=0.00..1.25 rows=25 width=434) (actual time=0.017..0.024 rows=25 loops=1)
 Total runtime: 0.178 ms
```

#### 分析
1. **执行方式：**引入 CTE（临时视图）。
2. **成本：**`cost=1.25..1.75`，引入额外的扫描和数据存储开销。
3. **总运行时间：**`0.178 ms`，高于视图查询和直接查询。
4. **结论：**使用`WITH`临时视图查询时，相比直接查询和视图查询，会引入一些额外的开销。尽管CTE提供了查询结构上的便利（尤其是在复杂查询中），但在这种简单查询下，开销较大。

---

### 直接查询
#### 执行计划
```sql
QUERY PLAN
----------------------------------------------------------------------------------------------------
 Seq Scan on nation  (cost=0.00..1.25 rows=25 width=104) (actual time=0.009..0.015 rows=25 loops=1)
 Total runtime: 0.065 ms
```

#### 分析
1. **执行方式：**顺序扫描。
2. **成本：**`cost=0.00..1.25`，与视图查询一致。
3. **总运行时间：**`0.065 ms`，为最优性能。
+ **结论：**直接查询的执行时间和视图查询非常相似，几乎没有额外的性能损耗，特别是在查询非常简单时。

---

# 三、优化 SQL 语句
## 3.1.复合索引左前缀
### 实验步骤
1. 在 `lineitem_new` 表上创建复合索引。
2. 比较有索引和无索引的查询性能。

### 实验代码
```sql
-- 创建备份表
CREATE TABLE lineitem_new AS TABLE lineitem;

-- 创建复合索引
CREATE INDEX lineitem_index ON lineitem_new(l_quantity, l_tax, l_extendedprice);

-- 无索引查询l_quantity=24的全部数据
EXPLAIN ANALYZE  
SELECT *  
FROM lineitem  
WHERE l_quantity = 24;

-- 使用最左前缀索引查询l_quantity=24的全部数据
EXPLAIN ANALYZE  
SELECT *  
FROM lineitem_new  
WHERE l_quantity = 24;

-- 无索引查询l_tax=0.02的全部数据
EXPLAIN ANALYZE  
SELECT *  
FROM lineitem  
WHERE l_tax=0.02;

-- 使用最左前缀索引查询l_tax=0.02的全部数据
EXPLAIN ANALYZE  
SELECT *  
FROM lineitem_new  
WHERE l_tax=0.02;

```



### 执行计划
#### **无索引查询 **`**l_quantity = 24**`** 的全部数据**
##### **执****行计划：**
```plain
Seq Scan on lineitem  (cost=0.00..97408.30 rows=20950 width=129) (actual time=58.944..275.978 rows=18229 loops=1)
   Filter: (l_quantity = 24::numeric)
   Rows Removed by Filter: 885624
Total runtime: 276.975 ms
(4 rows)
```

##### **分析：**
+ **执行方式：无索引查询对**`**lineitem**`**表进行了顺序扫描（**`**Seq Scan**`**）。**
+ **过滤条件：**`**l_quantity = 24**`**。**
+ **行数：查询返回的行数为18229行，过滤掉了885624行不符合条件的记录。**
+ **实际执行时间：执行时间为 **`**276.975 ms**`**。这是因为顺序扫描需要遍历整个表，过滤掉不满足条件的行，过程较为耗时。**

#### ** 使用最左前缀索引查询 **`**l_quantity = 24**`** 的全部数据**
##### **执行计划：**
```plain
Bitmap Heap Scan on lineitem_new  (cost=512.73..20123.63 rows=18619 width=129) (actual time=6.430..40.468 rows=18229 loops=1)
   Recheck Cond: (l_quantity = 24::numeric)
   ->  Bitmap Index Scan on lineitem_index  (cost=0.00..508.07 rows=18619 width=0) (actual time=5.275..5.275 rows=18229 loops=1)
         Index Cond: (l_quantity = 24::numeric)
Total runtime: 41.252 ms
(5 rows)
```

##### **分析：**
+ **执行方式：使用了**`**Bitmap Heap Scan**`**与**`**Bitmap Index Scan**`**，首先通过索引扫描来找出满足条件的行，然后通过堆扫描获取数据。**
+ **索引条件****：**`**l_quantity = 24**`**。**
+ **实际执行时间：执行时间为 **`**41.252 ms**`**，显著低于无索引查询的执行时间。**

#### ** 无索引查询 **`**l_tax = 0.02**`** 的全部数据**
##### **执行计划：**
```plain
Seq Scan on lineitem  (cost=0.00..97408.30 rows=117803 width=129) (actual time=56.628..287.455 rows=100598 loops=1)
   Filter: (l_tax = .02)
   Rows Removed by Filter: 803255
Total runtime: 291.463 ms
(4 rows)
```

##### **分析：**
+ **执行方式：无索引查询对**`**lineitem**`**表进行了顺序扫描（**`**Seq Scan**`**）。**
+ **过滤条件：**`**l_tax = 0.02**`**。**
+ **行数：查询返回的行数为100598行，过滤掉了803255行不符合条件的记录。**
+ **实际执行时间：执行时间为 **`**291.463 ms**`**，也比索引查询的时间长，原因与**`**l_quantity = 24**`**查询相同，都是因为需要扫描整个表。**

#### **使用最左前缀索引查询 **`**l_tax = 0.02**`** 的全部数据**
##### **执行计划：**
```plain
Seq Scan on lineitem_new  (cost=0.00..30066.16 rows=97616 width=129) (actual time=0.044..144.631 rows=100598 loops=1)
   Filter: (l_tax = .02)
   Rows Removed by Filter: 803255
Total runtime: 148.464 ms
(4 rows)
```

##### **分析：**
+ **执行方式：使用了最左前缀索引（**`**lineitem_index**`**）进行扫描。**
+ **过滤条件：**`**l_tax = 0.02**`**。**
+ **实际执行时间：执行时间为 **`**148.464 ms**`**，比无索引查询快，但没有比最左前缀索引的**`**l_quantity = 24**`**查询更快。**

---

1. **无索引查询**
    - 执行时间较长，顺序扫描导致高开销。
2. **使用复合索引**
    - 查询性能显著提升，索引有效降低了扫描行数。

---

### 结果分析
#### ** 执行结果是否一样？**
+ `**l_quantity = 24**`** 查询的执行结果一致，无论是使用最左前缀索引，还是无索引查询，返回的结果行数都是一致的，都是18229行。**
+ `**l_tax = 0.02**`** 查询的执行结果也一致，无论是使用最左前缀索引，还是无索引查询，返回的结果行数都是一致的，都是100598行。**

#### **对比执行效果和执行速度**
+ **无索引查询 vs 使用最左前缀索引查询 **
    - `**l_quantity = 24**`**执行时间差异：无索引查询的执行时间为 **`276.975 ms`**，而使用索引查询的执行时间为 **`41.252 ms`**，索引查询快得多。**
    - `**l_tax = 0.02**`**执行时间差异：无索引查询的执行时间为 **`291.463 ms`**，使用索引查询的执行时间为 **`148.464 ms`**，同样索引查询较快，但差异没有**`l_quantity = 24`**的查询那么明显。**

#### **解释执行时间差异的原因**
+ **索引的使用：最左前缀索引对于查询条件中第一个列的过滤效果最为显著。对于**`l_quantity = 24`**和**`l_tax = 0.02`**这两个查询，索引帮助减少了需要扫描的行数，尤其在有大量不符合条件的行时，索引的作用尤为明显。**
+ **查询条件的不同：**
    - **对于**`**l_quantity = 24**`**，查询的结果集行数相对较少（18229行），使用索引可以显著提高查询性能。**
    - **对于**`**l_tax = 0.02**`**，虽然查询条件相对更复杂，涉及到浮动的数值，但是由于没有索引列的顺序，扫描的行数相对较多（100598行）。因此，索引的作用虽然存在，但在这种情况下，索引的效率提升比**`**l_quantity = 24**`**时要小一些。**
+ **Bitmap索引的使用：在最左前缀索引查询中，**`Bitmap Index Scan`**会首先在索引中查找符合条件的行，并生成一个位图（Bitmap），然后通过**`Bitmap Heap Scan`**在数据表中读取相关数据。这种方法减少了对数据表的重复扫描，使得查询速度更快。无索引查询则需要顺序扫描整个表，比较慢。**

---

### 结论
+ **执行结果****在两种查询中是相同的，无论是否使用索引。**
+ **执行时间****上，使用最左前缀索引的查询显著优于无索引查询，尤其在涉及大量数据时，索引查询的性能更为突出。**
+ **差异原因：索引通过减少扫描的数据量，加速了查询过程。顺序扫描则需要扫描整个表，导致执行时间较长。**`**Bitmap Index Scan**`**能显著提升查询效率，尤其是在匹配条件较多的情况下。**



## 3.2 多表连接操作，在连接属性上建立索引
### 实验步骤
1. 创建 `lineitem` 表的备份表 `lineitem_new`。  
2. 在 `lineitem_new` 上删除实验 3.1 创建的组合索引，并在 `l_suppkey` 和 `l_partkey` 上分别创建两个索引：  
    - 索引 1：`lineitem_index1`，针对 `l_suppkey`。  
    - 索引 2：`lineitem_index2`，针对 `l_partkey`。
3. 编写并执行两组 SQL 语句：  
    - 使用索引和不使用索引分别查询 `supplier` 表中供应商手机号。  
    - 使用索引和不使用索引分别查询 `part` 表中零件名称。
4. 对比两组查询的执行计划和查询执行的耗时。

### 实验代码
#### 索引创建
```sql
DROP INDEX IF EXISTS lineitem_index;
CREATE INDEX lineitem_index1 ON lineitem_new(l_suppkey);
CREATE INDEX lineitem_index2 ON lineitem_new(l_partkey);
```

#### 无索引查询供应商手机号
```sql
EXPLAIN ANALYZE  
SELECT DISTINCT l_suppkey, s_phone  
FROM lineitem, supplier  
WHERE l_suppkey = 10 AND l_suppkey = supplier.s_suppkey;
```

#### 使用索引查询供应商手机号
```sql
EXPLAIN ANALYZE  
SELECT DISTINCT l_suppkey, s_phone  
FROM lineitem_new, supplier  
WHERE l_suppkey = 10 AND l_suppkey = supplier.s_suppkey;
```

#### 无索引查询零件名称
```sql
EXPLAIN ANALYZE  
SELECT DISTINCT l_partkey, p_name  
FROM lineitem, part  
WHERE l_partkey = 928 AND l_partkey = part.p_partkey;
```

#### 使用索引查询零件名称
```sql
EXPLAIN ANALYZE  
SELECT DISTINCT l_partkey, p_name  
FROM lineitem_new, part  
WHERE l_partkey = 928 AND l_partkey = part.p_partkey;
```

### 执行计划
#### 无索引查询供应商手机号
```sql
                                                             QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=97424.11..97424.12 rows=1 width=20) (actual time=235.376..235.376 rows=1 loops=1)
   Group By Key: lineitem.l_suppkey, supplier.s_phone
   ->  Nested Loop  (cost=0.00..97421.60 rows=503 width=20) (actual time=63.298..235.224 rows=450 loops=1)
         ->  Index Scan using supplier_pkey on supplier  (cost=0.00..8.27 rows=1 width=20) (actual time=0.037..0.039 rows=1 loops=1)
               Index Cond: (s_suppkey = 10)
         ->  Seq Scan on lineitem  (cost=0.00..97408.30 rows=503 width=4) (actual time=63.255..235.093 rows=450 loops=1)
               Filter: (l_suppkey = 10)
               Rows Removed by Filter: 903403
 Total runtime: 235.540 ms
(9 rows)
```

#### 使用索引查询供应商手机号
```sql
                                                               QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=1570.10..1570.11 rows=1 width=20) (actual time=4.265..4.265 rows=1 loops=1)
   Group By Key: lineitem_new.l_suppkey, supplier.s_phone
   ->  Nested Loop  (cost=11.75..1567.91 rows=439 width=20) (actual time=0.400..4.044 rows=450 loops=1)
         ->  Index Scan using supplier_pkey on supplier  (cost=0.00..8.27 rows=1 width=20) (actual time=0.013..0.015 rows=1 loops=1)
               Index Cond: (s_suppkey = 10)
         ->  Bitmap Heap Scan on lineitem_new  (cost=11.75..1555.25 rows=439 width=4) (actual time=0.379..3.880 rows=450 loops=1)
               Recheck Cond: (l_suppkey = 10)
               ->  Bitmap Index Scan on lineitem_index1  (cost=0.00..11.64 rows=439 width=0) (actual time=0.218..0.218 rows=450 loops=1)
                     Index Cond: (l_suppkey = 10)
 Total runtime: 4.401 ms
(10 rows)
```

#### 无索引查询零件名称
```sql
                                                         QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=97416.96..97416.97 rows=1 width=42) (actual time=236.517..236.518 rows=1 loops=1)
   Group By Key: lineitem.l_partkey, part.p_name
   ->  Nested Loop  (cost=0.00..97416.83 rows=26 width=42) (actual time=116.263..236.486 rows=14 loops=1)
         ->  Index Scan using part_pkey on part  (cost=0.00..8.27 rows=1 width=42) (actual time=0.761..0.763 rows=1 loops=1)
               Index Cond: (p_partkey = 928)
         ->  Seq Scan on lineitem  (cost=0.00..97408.30 rows=26 width=4) (actual time=115.489..235.704 rows=14 loops=1)
               Filter: (l_partkey = 928)
               Rows Removed by Filter: 903839
 Total runtime: 236.661 ms
(9 rows)
```

#### 使用索引查询零件名称
```sql
                                                              QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=103.02..103.03 rows=1 width=42) (actual time=0.413..0.414 rows=1 loops=1)
   Group By Key: lineitem_new.l_partkey, part.p_name
   ->  Nested Loop  (cost=4.53..102.90 rows=23 width=42) (actual time=0.188..0.370 rows=14 loops=1)
         ->  Index Scan using part_pkey on part  (cost=0.00..8.27 rows=1 width=42) (actual time=0.018..0.020 rows=1 loops=1)
               Index Cond: (p_partkey = 928)
         ->  Bitmap Heap Scan on lineitem_new  (cost=4.53..94.40 rows=23 width=4) (actual time=0.160..0.328 rows=14 loops=1)
               Recheck Cond: (l_partkey = 928)
               ->  Bitmap Index Scan on lineitem_index2  (cost=0.00..4.52 rows=23 width=0) (actual time=0.131..0.131 rows=14 loops=1)
                     Index Cond: (l_partkey = 928)
 Total runtime: 0.595 ms
(10 rows)
```

### 结果分析
#### 无索引与有索引查询供应商手机号
**查询目标**：列出在 `lineitem` 表中某个 `l_suppkey`（供应商 ID）对应的所有供应商的手机号。

##### 无索引查询：
+ **查询计划**：
    - 使用 `Nested Loop` 连接 `supplier` 和 `lineitem`。
    - 在 `supplier` 表上使用主键索引进行扫描（`supplier_pkey`），并通过 `s_suppkey = 10` 查找供应商。
    - 对于 `lineitem` 表，执行顺序扫描（`Seq Scan`），然后根据 `l_suppkey = 10` 筛选记录。
    - 运行时间：235.540 ms。

##### 有索引查询：
+ **查询计划**：
    - 使用 `Nested Loop` 连接 `supplier` 和 `lineitem_new`。
    - 在 `supplier` 表上同样使用主键索引进行扫描（`supplier_pkey`），通过 `s_suppkey = 10` 查找供应商。
    - 对于 `lineitem_new` 表，使用 `Bitmap Heap Scan`，并通过 `l_suppkey = 10` 条件进行索引扫描（`Bitmap Index Scan`），因为已经在 `l_suppkey` 上建立了索引。
    - 运行时间：4.401 ms。

##### 执行时间差异的原因：
+ **无索引查询**使用了全表扫描（`Seq Scan`），因此需要检查整个 `lineitem` 表中是否有符合条件的记录。这导致了查询时间较长（235.540 ms）。
+ **有索引查询**使用了 `Bitmap Index Scan`，这大大减少了扫描的范围，避免了对整个 `lineitem` 表的顺序扫描，从而提高了查询效率，显著缩短了执行时间（4.401 ms）。
+ 主要的性能提升来自索引的使用，它使得查询能够直接通过索引定位到符合条件的行，而无需逐行扫描整个表。

#### 无索引与有索引查询零件名称
**查询目标**：列出在 `lineitem` 表中某个 `l_partkey`（零件 ID）对应的所有零件名称。

##### 无索引查询：
+ **查询计划**：
    - 使用 `Nested Loop` 连接 `part` 和 `lineitem`。
    - 在 `part` 表上使用主键索引进行扫描（`part_pkey`），通过 `p_partkey = 928` 查找零件。
    - 对于 `lineitem` 表，执行顺序扫描（`Seq Scan`），然后根据 `l_partkey = 928` 筛选记录。
    - 运行时间：236.661 ms。

##### 有索引查询：
+ **查询计划**：
    - 使用 `Nested Loop` 连接 `part` 和 `lineitem_new`。
    - 在 `part` 表上同样使用主键索引进行扫描（`part_pkey`），通过 `p_partkey = 928` 查找零件。
    - 对于 `lineitem_new` 表，使用 `Bitmap Heap Scan`，并通过 `l_partkey = 928` 条件进行索引扫描（`Bitmap Index Scan`），因为已经在 `l_partkey` 上建立了索引。
    - 运行时间：0.595 ms。

##### 执行时间差异的原因：
+ **无索引查询**同样使用了顺序扫描（`Seq Scan`），需要扫描整个 `lineitem` 表，查找与给定 `l_partkey` 匹配的记录。由于没有索引支持，这导致了较高的执行时间（236.661 ms）。
+ **有索引查询**使用了 `Bitmap Index Scan`，通过索引快速定位到符合条件的行，并使用 `Bitmap Heap Scan` 提取实际数据，显著提升了查询效率。由于索引加速了数据访问，查询时间减少到了 0.595 ms。

### 实验结论：
+ **无索引查询**由于必须执行全表扫描（`Seq Scan`），查询时间较长。
+ **有索引查询**使用了索引扫描（`Bitmap Index Scan`）来加速数据访问，减少了需要扫描的行数，从而显著缩短了执行时间。
+ 在这两个查询中，执行时间的差异主要是由是否存在合适的索引来决定的，索引能有效减少数据扫描的范围，提高查询效率。

### 结论
在多表连接查询中，针对连接属性建立索引能够有效减少查询的时间开销，显著提升性能。

---

## 3.3 索引对小表查询的作用
### 实验步骤
1. 创建 `supplier` 表的不带主键索引的备份表 `supplier_new`。  
2. 在 `supplier` 表的 `s_suppkey` 上创建索引。  
3. 执行查询：  
    - 不强制使用索引：查询是否会自动使用索引。  
    - 强制使用索引：通过设置 `enable_seqscan=false` 禁用顺序扫描，强制使用索引进行查询。
4. 比较两种查询的执行计划和执行时间。

### 实验代码
#### 创建备份表
```sql
CREATE TABLE supplier_new AS TABLE supplier;
```

#### 不强制使用索引查询
```sql
EXPLAIN  
SELECT s_suppkey  
FROM supplier;
```

#### 强制使用索引查询
```sql
SET enable_seqscan = OFF;  
EXPLAIN  
SELECT s_suppkey  
FROM supplier;
```

### 执行计划
#### 不强制使用索引查询
```sql
                         QUERY PLAN
------------------------------------------------------------
 Seq Scan on supplier  (cost=0.00..62.00 rows=2000 width=4)
(1 row)
```

#### 强制使用索引查询
```sql
                                   QUERY PLAN
--------------------------------------------------------------------------------
 Bitmap Heap Scan on supplier  (cost=42.75..104.75 rows=2000 width=4)
   ->  Bitmap Index Scan on supplier_pkey  (cost=0.00..42.25 rows=2000 width=0)
(2 rows)
```

### 结果分析
#### **不强制使用索引时**：
+ 查询优化器决定不使用索引，而是执行全表扫描（`Seq Scan`）。
+ 这种情况通常发生在查询的结果集较大，或者索引无法有效过滤掉大量数据时。在这种情况下，执行全表扫描可能比使用索引更加高效。

#### **强制使用索引时：**
+ 强制索引扫描后，查询计划显示使用了 `Bitmap Index Scan` 和 `Bitmap Heap Scan`。
+ 通过强制索引扫描，你迫使查询优化器使用索引，虽然在某些情况下这可能不是最优选择，因为索引扫描会涉及额外的成本（例如，在执行 `Bitmap Heap Scan` 时需要访问实际数据）。

#### **对比和分析：**
+ **不强制使用索引时**，由于没有强制要求使用索引，优化器选择了全表扫描（`Seq Scan`），这通常发生在查询范围较大或索引不够有效时。
+ **强制使用索引时**，索引扫描虽然能提高定位的精确度，但在某些情况下，它的成本反而较高，因为查询需要额外的步骤（`Bitmap Heap Scan`）来访问数据。

### 结论
+ **索引不一定总是优于全表扫描**。查询优化器通常会根据查询的实际情况（如数据量、过滤条件等）选择最优的执行路径。
+ **强制使用索引**虽然能确保索引被使用，但在某些情况下，这可能反而导致较高的执行成本。因此，在实际使用中，应根据查询的特点来决定是否强制使用索引

---

## 3.4 查询条件中函数对索引的影响
### 实验步骤
1. 创建 `lineitem_new` 表，并在 `l_discount` 和 `l_quantity` 上分别创建索引。  
2. 编写两条等价查询语句：  
    - 一条在查询条件中使用函数。  
    - 一条通过改写条件避免使用函数。
3. 执行两条语句，比较其执行计划和执行时间。

### 实验代码
#### 创建索引
```sql
CREATE INDEX lineitem_index1 ON lineitem_new(l_discount);
CREATE INDEX lineitem_index2 ON lineitem_new(l_quantity);
```

#### 使用函数的查询
```sql
EXPLAIN  
SELECT DISTINCT B.l_orderkey  
FROM lineitem_new AS A, lineitem_new AS B  
WHERE A.l_extendedprice = 16473.51  
  AND A.l_discount = 0.04  
  AND ABS(A.l_quantity - B.l_quantity) < 20;
```

#### 不使用函数的查询
```sql
EXPLAIN  
SELECT DISTINCT B.l_orderkey  
FROM lineitem_new AS A, lineitem_new AS B  
WHERE A.l_extendedprice = 16473.51  
  AND A.l_discount = 0.04  
  AND B.l_quantity > A.l_quantity - 20  
  AND B.l_quantity < A.l_quantity + 20;
```

### 执行计划
#### 使用函数的查询
```sql
                                             QUERY PLAN
-----------------------------------------------------------------------------------------------------
 HashAggregate  (cost=1000002818715.57..1000002820813.88 rows=209831 width=4)
   Group By Key: b.l_orderkey
   ->  Nested Loop  (cost=10000001505.54..1000002817962.36 rows=301284 width=4)
         Join Filter: (abs((a.l_quantity - b.l_quantity)) < 20::numeric)
         ->  Bitmap Heap Scan on lineitem_new a  (cost=1505.54..21491.93 rows=1 width=5)
               Recheck Cond: (l_discount = .04)
               Filter: (l_extendedprice = 16473.51)
               ->  Bitmap Index Scan on lineitem_index1  (cost=0.00..1505.54 rows=81226 width=0)
                     Index Cond: (l_discount = .04)
         ->  Seq Scan on lineitem_new b  (cost=10000000000.00..1000002780653.00 rows=903853 width=9)
(10 rows)
```

#### 不使用函数的查询
```sql
                                                          QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=45661.59..46665.87 rows=100428 width=4)
   Group By Key: b.l_orderkey
   ->  Nested Loop  (cost=3643.29..45410.52 rows=100428 width=4)
         ->  Bitmap Heap Scan on lineitem_new a  (cost=1505.54..21491.93 rows=1 width=5)
               Recheck Cond: (l_discount = .04)
               Filter: (l_extendedprice = 16473.51)
               ->  Bitmap Index Scan on lineitem_index1  (cost=0.00..1505.54 rows=81226 width=0)
                     Index Cond: (l_discount = .04)
         ->  Bitmap Heap Scan on lineitem_new b  (cost=2137.74..22914.30 rows=100428 width=9)
               Recheck Cond: ((l_quantity > (a.l_quantity - 20::numeric)) AND (l_quantity < (a.l_quantity + 20::numeric)))
               ->  Bitmap Index Scan on lineitem_index2  (cost=0.00..2112.63 rows=100428 width=0)
                     Index Cond: ((l_quantity > (a.l_quantity - 20::numeric)) AND (l_quantity < (a.l_quantity + 20::numeric)))
(12 rows)
```

### 结果分析
#### **使用函数时的执行计划**
+ **执行方式**：
    - 在 `WHERE` 子句中使用了 `abs(a.l_quantity - b.l_quantity) < 20` 函数。由于该函数的存在，查询优化器无法利用 `l_quantity` 上的索引来进行高效的查找。
    - 在执行计划中，查询优化器选择了 `Seq Scan`，即全表扫描，针对 `lineitem_new` 表的 `b` 表进行扫描，而没有使用索引。这导致了查询性能的显著降低。
    - 虽然在 `a` 表上能有效使用索引，但由于 `b` 表的查询条件包含了函数，导致不能有效利用索引，进而导致了全表扫描（`Seq Scan`）的使用。

#### **不使用函数时的执行计划**
+ **执行方式**：
    - 使用了 `BETWEEN a.l_quantity - 20 AND a.l_quantity + 20` 条件，而没有使用函数。这时，查询优化器能够利用 `l_quantity` 列上的索引（`lineitem_index2`），因为索引能够直接处理范围查询。
    - 在执行计划中，查询优化器选择了 `Bitmap Index Scan`，并使用了 `Bitmap Heap Scan` 来根据索引定位数据，显著提升了查询性能。

#### **性能差异的原因**
+ **使用函数的查询**：由于 `abs()` 函数的存在，查询优化器无法利用索引对 `l_quantity` 列进行高效的扫描，因此选择了全表扫描（`Seq Scan`）。
+ **不使用函数的查询**：查询条件可以直接通过索引扫描处理（`Bitmap Index Scan`），因此查询性能较好。

### 结论
+ 使用函数（如 `abs()`）会导致查询条件失效，优化器无法使用索引，最终可能选择全表扫描，从而导致性能下降。
+ 通过优化查询语句，避免在 `WHERE` 子句中使用函数，可以提高查询的性能，并使索引得以充分利用。

---

## 3.5 多表嵌入式 SQL 查询
### 实验步骤
1. 编写两条等价查询语句：  
    - 一条使用嵌套查询。  
    - 一条使用连接查询。
2. 执行两条语句，比较其执行计划和执行时间。  
3. 对实验结果进行分析，确定两种方法在性能上的差异。

### 实验代码
#### 嵌套查询
```sql
EXPLAIN ANALYZE  
SELECT A.p_name  
FROM part AS A  
WHERE A.p_partkey IN (  
  SELECT B.ps_partkey  
  FROM partsupp AS B  
  WHERE B.ps_suppkey = 1002  
);
```

#### 连接查询
```sql
EXPLAIN ANALYZE  
SELECT A.p_name  
FROM part AS A, partsupp AS B  
WHERE B.ps_suppkey = 1002  
  AND A.p_partkey = B.ps_partkey;
```

### 执行计划
#### 嵌套查询
```sql
                                                                 QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------
 Nested Loop Semi Join  (cost=0.00..3273.74 rows=78 width=38) (actual time=0.042..41.381 rows=80 loops=1)
   ->  Index Scan using part_pkey on part a  (cost=0.00..1903.25 rows=40000 width=42) (actual time=0.012..5.820 rows=40000 loops=1)
   ->  Index Only Scan using partsupp_pkey on partsupp b  (cost=0.00..0.68 rows=21 width=4) (actual time=30.008..30.008 rows=80 loops=40000)
         Index Cond: ((ps_partkey = a.p_partkey) AND (ps_suppkey = 1002))
         Heap Fetches: 80
 Total runtime: 41.493 ms
(6 rows)
```

#### 连接查询
```sql
                                                                QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.00..3666.56 rows=78 width=38) (actual time=0.050..5.205 rows=80 loops=1)
   ->  Index Only Scan using partsupp_pkey on partsupp b  (cost=0.00..3112.37 rows=78 width=4) (actual time=0.022..4.722 rows=80 loops=1)
         Index Cond: (ps_suppkey = 1002)
         Heap Fetches: 80
   ->  Index Scan using part_pkey on part a  (cost=0.00..7.09 rows=1 width=42) (actual time=0.373..0.401 rows=80 loops=80)
         Index Cond: (p_partkey = b.ps_partkey)
 Total runtime: 5.314 ms
(7 rows)
```

### 结果分析
####  查询执行计划对比  
#####  嵌套查询执行计划  
+ **执行计划分析**：
+ 在嵌套查询中，查询优化器执行了一个 **Nested Loop Semi Join**，即首先通过 `part` 表扫描（`Index Scan`），然后通过 `partsupp` 表对每个零件查询是否存在供应商 `ps_suppkey = 1002`。
+ `partsupp` 表的扫描是通过 **Index Only Scan** 来实现的，它直接利用了 `ps_partkey` 和 `ps_suppkey` 列的索引。每个 `part` 表中的条目都需要对 `partsupp` 表执行一次扫描，导致执行计划中有多次 `Heap Fetches`。
+ **性能瓶颈**：
+ 执行时间为 **41.493 ms**，嵌套查询使用了 `Index Only Scan` 和 `Heap Fetches`，导致对于每个 `part` 表的记录都需要查找对应的 `partsupp` 表记录，可能会导致较高的 I/O 开销。

#####  连接查询执行计划  
+ **执行计划分析**：
+ 在连接查询中，查询优化器首先通过 `partsupp` 表上的索引（`Index Only Scan`）扫描所有供应商 `ps_suppkey = 1002` 的记录。
+ 然后，它通过 **Index Scan** 在 `part` 表中查找相应的零件 `p_partkey`。
+ 查询优化器使用了有效的索引扫描，没有出现额外的 `Heap Fetches`，因此查询速度较快。
+ **性能瓶颈**：
+ 执行时间为 **5.314 ms**，连接查询直接利用了索引，减少了额外的 I/O 开销，因此查询时间比嵌套查询要低得多。

####  性能差异的原因  
##### 嵌套查询（Subquery）：
+ **效率较低**：嵌套查询通常会对外层查询的每一行执行一次内层查询，因此在涉及到多个记录时，查询可能会变得非常低效。
+ **多次索引查找**：在执行嵌套查询时，`part` 表的每一条记录都需要访问 `partsupp` 表来查找是否存在符合条件的记录，这可能导致较高的 I/O 操作（即每次对 `partsupp` 表执行查找）。

##### 连接查询（Join）：
+ **更高效的索引使用**：连接查询通过索引扫描在 `partsupp` 和 `part` 表之间建立直接的连接，避免了对每一条记录都执行子查询。直接连接使得查询能够更有效地利用索引，减少了不必要的 I/O 操作。
+ **索引优化**：在连接查询中，使用了对 `ps_suppkey` 和 `p_partkey` 的索引，这样能够快速找到匹配的记录。

### 结论
+ 尽量避免使用嵌套查询，优先选择连接查询以提升查询性能。

---

## 3.6 where 查询条件中复合查询条件 OR 对索引的影响
### 实验步骤
1. 在 `lineitem_new` 表的 `l_quantity` 上创建索引。  
2. 编写查询语句：  
    - 一条使用 `OR` 条件。  
    - 另一条将 `OR` 条件改写为两条查询通过 `UNION` 连接。
3. 比较两种查询方式的执行计划和查询性能。

### 实验代码
#### 创建索引
```sql
CREATE INDEX lineitem_index ON lineitem_new(l_quantity);
```

#### 含有 `OR` 的查询
```sql
EXPLAIN  
SELECT *  
FROM lineitem_new  
WHERE l_tax = 0.06 AND (l_quantity = 36 OR l_discount = 0.09);
```

#### 改写为 `UNION` 的查询
```sql
EXPLAIN  
(SELECT *  
 FROM lineitem_new  
 WHERE l_tax = 0.06 AND l_quantity = 36)  
UNION  
(SELECT *  
 FROM lineitem_new  
 WHERE l_tax = 0.06 AND l_discount = 0.09);
```

### 执行计划
#### 含有 `OR` 的查询
```sql
                                       QUERY PLAN
----------------------------------------------------------------------------------------
 Seq Scan on lineitem_new  (cost=10000000000.00..1000003458542.75 rows=10825 width=129)
   Filter: ((l_tax = .06) AND ((l_quantity = 36::numeric) OR (l_discount = .09)))
(2 rows)
```

#### 改写为 `UNION` 的查询
```sql
                                                                                                                         QUERY PLAN


------------------------------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=1000003253050.43..1000003253160.49 rows=11006 width=129)
   Group By Key: public.lineitem_new.l_orderkey, public.lineitem_new.l_partkey, public.lineitem_new.l_suppkey, public.lineitem_new.l_linenumber,
 public.lineitem_new.l_quantity, public.lineitem_new.l_extendedprice, public.lineitem_new.l_discount, public.lineitem_new.l_tax, public.lineitem
_new.l_returnflag, public.lineitem_new.l_linestatus, public.lineitem_new.l_shipdate, public.lineitem_new.l_commitdate, public.lineitem_new.l_rec
eiptdate, public.lineitem_new.l_shipinstruct, public.lineitem_new.l_shipmode, public.lineitem_new.l_comment
   ->  Append  (cost=341.10..1000003252610.19 rows=11006 width=129)
         ->  Bitmap Heap Scan on lineitem_new  (cost=341.10..19920.63 rows=2010 width=129)
               Recheck Cond: (l_quantity = 36::numeric)
               Filter: (l_tax = .06)
               ->  Bitmap Index Scan on lineitem_index  (cost=0.00..340.60 rows=18167 width=0)
                     Index Cond: (l_quantity = 36::numeric)
         ->  Seq Scan on lineitem_new  (cost=10000000000.00..1000003232579.50 rows=8996 width=129)
               Filter: ((l_tax = .06) AND (l_discount = .09))
(10 rows)
```

### 结果分析
#### ** 含有 OR 的查询  **
+ **执行计划分析：**
    - **Seq Scan**：查询计划显示，尽管有条件 `l_quantity = 36` 和 `l_discount = 0.09`，但由于包含了 `OR`，查询执行计划选择了顺序扫描（`Seq Scan`），而没有利用任何索引。
    - **原因**：由于 `OR` 条件可能导致查询无法利用现有的单列索引。如果某个条件（如 `l_quantity = 36`）可以使用索引，而另一个条件（如 `l_discount = 0.09`）不能，则数据库通常会选择顺序扫描（`Seq Scan`），因为 `OR` 会使得查询的两部分条件被独立地处理，从而无法利用一个单一的索引。

#### **使用 UNION 的查询  **
+ **执行计划分析：**
    - **第一个子查询**利用了 `Bitmap Heap Scan` 和 `Bitmap Index Scan`，能够有效地利用索引（`l_quantity = 36`）。
    - **第二个子查询**则是对 `l_discount = 0.09` 进行了顺序扫描（`Seq Scan`），因为 `l_discount` 可能没有索引或者索引的使用不如 `l_quantity` 高效。

#### **性能差异**：
+ **原始查询（含 OR 条件）**：
+ 由于 OR 条件可能导致查询无法有效使用索引，查询会执行顺序扫描（`Seq Scan`），这会导致性能较差，尤其是在数据量较大时。
+ **改写为 UNION 的查询**：
+ 将查询拆成两个独立的查询，可以分别利用不同的条件（`l_quantity` 和 `l_discount`）来使用索引，从而提高查询效率。尤其是对于 `l_quantity` 使用了索引，查询的执行会显著加速。
+ 通过 `UNION` 合并结果，可以避免对 `OR` 条件的处理，使得每个查询都能利用其特定的索引。

### 结论
+ **含 OR 的查询**：由于包含 OR 条件，查询计划可能会选择顺序扫描（`Seq Scan`），不能有效利用索引，从而导致性能较差。
+ **使用 UNION 的查询**：将查询拆分为两个子查询并通过 UNION 合并，每个查询都能分别利用适当的索引，从而提高了查询效率。

---

## 3.7 聚集运算中的索引设计
### 实验步骤
1. 在 `lineitem_new` 表上创建适合分组和聚集操作的索引：  
    - 针对 `GROUP BY` 的分组属性建立索引。  
    - 针对聚集运算的属性建立索引。
2. 分别在无索引表和有索引表上执行相同的查询。  
3. 强制使用索引后再次执行查询，观察性能变化。

### 实验代码
#### 创建索引
```sql
CREATE INDEX lineitem_index1 ON lineitem_new(l_extendedprice);
CREATE INDEX lineitem_index2 ON lineitem_new(l_suppkey);
```

#### 无索引查询
```sql
EXPLAIN  
SELECT l_suppkey, AVG(l_extendedprice) AS avg_price  
FROM lineitem  
GROUP BY l_suppkey;
```

#### 有索引查询
```sql
EXPLAIN  
SELECT l_suppkey, AVG(l_extendedprice) AS avg_price  
FROM lineitem_new  
GROUP BY l_suppkey;
```

#### 强制使用索引查询
```sql
SET enable_seqscan = OFF;  
EXPLAIN  
SELECT l_suppkey, AVG(l_extendedprice) AS avg_price  
FROM lineitem_new  
GROUP BY l_suppkey;
```

### 执行计划
#### 无索引查询
```sql
                               QUERY PLAN
-------------------------------------------------------------------------
 HashAggregate  (cost=100005.36..100030.36 rows=2000 width=44)
   Group By Key: l_suppkey
   ->  Seq Scan on lineitem  (cost=0.00..94811.24 rows=1038824 width=12)
(3 rows)
```

#### 有索引查询
```sql
                                 QUERY PLAN
----------------------------------------------------------------------------
 HashAggregate  (cost=32325.79..32350.79 rows=2000 width=44)
   Group By Key: l_suppkey
   ->  Seq Scan on lineitem_new  (cost=0.00..27806.53 rows=903853 width=12)
(3 rows)
```

#### 强制使用索引查询
```sql
                                             QUERY PLAN
-----------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=0.00..892878.29 rows=2000 width=44)
   Group By Key: l_suppkey
   ->  Index Scan using lineitem_index2 on lineitem_new  (cost=0.00..888334.02 rows=903853 width=12)
(3 rows)
```

### 结果分析
#### 无索引查询
+ **执行计划分析**：
    - **Seq Scan**：查询计划显示，数据库执行顺序扫描（`Seq Scan`）来处理 `lineitem` 表，因为没有合适的索引来优化分组操作。扫描表中的所有数据，计算每个供应商的平均价格。
    - **HashAggregate**：由于没有索引，数据库使用了哈希聚合（`HashAggregate`）来按 `l_suppkey` 进行分组并计算平均值。
    - 执行时间：由于没有索引，扫描整个表需要较长的时间，尤其是表数据量较大时。

#### 有索引查询
+ **执行计划分析**：
    - **Seq Scan**：即使我们创建了索引，查询仍然使用了顺序扫描（`Seq Scan`）。虽然 `l_suppkey` 有索引，但是在这种情况下数据库可能认为顺序扫描更高效，因为它可能需要读取整个表的数据来进行聚合操作，特别是当表中的数据量较大时，索引可能没有明显的优势。
    - **HashAggregate**：数据库仍然使用了哈希聚合（`HashAggregate`）来按 `l_suppkey` 分组并计算平均值。即使有索引，数据库并没有选择使用索引扫描（`Index Scan`）来代替顺序扫描。

#### 强制使用索引查询
+ **执行计划分析**：
    - **Index Scan**：由于强制启用索引扫描，数据库选择了使用 `lineitem_index2` 索引来扫描 `lineitem_new` 表。通过索引扫描，可以有效地按 `l_suppkey` 进行分组，避免了顺序扫描（`Seq Scan`）带来的性能损失。
    - **GroupAggregate**：与无索引查询类似，数据库使用哈希聚合来按 `l_suppkey` 进行分组并计算平均值，但索引扫描提供了更高效的数据访问方式。
    - 执行时间：强制使用索引通常会提升查询性能，尤其是当表中有大量数据时。

#### 对比执行计划：
| 查询类型 | 执行计划 | 说明 |
| --- | --- | --- |
| **无索引查询** | `Seq Scan`<br/> + `HashAggregate` | 执行顺序扫描，按 `l_suppkey`<br/> 进行哈希聚合。数据量大时性能差。 |
| **有索引查询** | `Seq Scan`<br/> + `HashAggregate` | 即使有索引，数据库选择了顺序扫描，可能因为需要对整个表进行聚合。 |
| **强制使用索引查询** | `Index Scan`<br/> + `GroupAggregate` | 强制使用索引扫描，避免顺序扫描，能够更高效地利用索引。 |


+ **执行速度对比：**
    - **无索引查询**：由于没有索引，查询性能较差，尤其是当表数据量较大时，顺序扫描会导致性能瓶颈。
    - **有索引查询**：虽然创建了索引，数据库仍然选择顺序扫描，导致性能没有显著提升。
    - **强制使用索引查询**：通过强制使用索引扫描，查询性能得到明显提升。索引帮助避免了顺序扫描，从而加速了查询。

### 结论
+ **无索引查询**：由于没有索引，数据库需要扫描整个表来执行聚合操作，性能较差，特别是在数据量大时。
+ **有索引查询**：即使创建了索引，如果查询条件没有完全匹配索引或数据库认为顺序扫描更高效，查询可能仍然会执行顺序扫描，导致性能没有提升。
+ **强制使用索引查询**：强制启用索引扫描通常会提高查询效率，尤其是在大表聚合时，通过索引扫描能够避免顺序扫描带来的性能损失。

---

## 3.8 Select 子句中有无 distinct 的区别
### 实验步骤
1. 编写两条查询语句：  
    - 一条不使用 `DISTINCT`。  
    - 一条使用 `DISTINCT`。
2. 比较两种方式的执行计划和查询效率。

### 实验代码
#### 无 `DISTINCT` 查询
```sql
EXPLAIN ANALYZE  
SELECT o_orderkey  
FROM orders  
WHERE o_orderstatus = 'O'  
  AND o_orderpriority = '4-NOT SPECIFIED';
```

#### 有 `DISTINCT` 查询
```sql
EXPLAIN ANALYZE  
SELECT DISTINCT o_orderkey  
FROM orders  
WHERE o_orderstatus = 'O'  
  AND o_orderpriority = '4-NOT SPECIFIED';
```

### 执行计划
#### 无 `DISTINCT` 查询
```sql
                                                          QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on orders  (cost=10000000000.00..1000000878600.00 rows=29224 width=4) (actual time=0.072..46.682 rows=29224 loops=1)
   Filter: ((o_orderstatus = 'O'::bpchar) AND (o_orderpriority = '4-NOT SPECIFIED'::bpchar))
   Rows Removed by Filter: 270777
 Total runtime: 48.135 ms
(4 rows)
```

#### 有 `DISTINCT` 查询
```sql
                                                               QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------
 Unique  (cost=0.00..13670.34 rows=29224 width=4) (actual time=47.678..112.071 rows=29224 loops=1)
   ->  Index Scan using orders_pkey on orders  (cost=0.00..13597.28 rows=29224 width=4) (actual time=47.673..108.759 rows=29224 loops=1)
         Filter: ((o_orderstatus = 'O'::bpchar) AND (o_orderpriority = '4-NOT SPECIFIED'::bpchar))
         Rows Removed by Filter: 270777
 Total runtime: 113.043 ms
(5 rows)
```

### 结果分析
#### 无 `DISTINCT` 查询
+ **执行计划分析**：
    - **Seq Scan**：查询执行计划显示，数据库对 `orders` 表进行了顺序扫描（`Seq Scan`）。这是因为没有索引可以有效地用于 `o_orderstatus` 和 `o_orderpriority` 的过滤条件，或者数据库评估认为顺序扫描在此情况下更为高效。
    - **Filter**：使用了过滤条件 `o_orderstatus = 'O'` 和 `o_orderpriority = '4-NOT SPECIFIED'` 来筛选符合条件的记录。
    - **Rows Removed by Filter**：从原始数据集中，过滤掉了 270,777 条不符合条件的记录。
+ **执行时间**：查询扫描整个表并筛选符合条件的记录，执行时间为 48.135 毫秒。

#### 有 `DISTINCT` 查询
+ **执行计划分析**：
    - **Unique**：查询计划中使用了 `Unique` 操作符，这是因为 `DISTINCT` 需要确保结果中没有重复的记录，因此会进行额外的去重操作。
    - **Index Scan**：数据库选择了使用 `orders_pkey` 索引进行扫描。与无 `DISTINCT` 查询相比，查询计划使用了索引扫描（`Index Scan`），这通常比顺序扫描更高效，尤其是对于带有条件过滤的查询。
    - **Filter**：使用了和之前相同的过滤条件 `o_orderstatus = 'O'` 和 `o_orderpriority = '4-NOT SPECIFIED'`。
    - **Rows Removed by Filter**：过滤掉了 270,777 条不符合条件的记录。
+ **执行时间**：尽管使用了索引扫描，但 `DISTINCT` 操作导致查询的执行时间较长，达到了 113.043 毫秒。去重操作消耗了更多时间。

####  对比执行计划  
| 查询类型 | 执行计划 | 说明 |
| --- | --- | --- |
| **无 DISTINCT 查询** | `Seq Scan`<br/> + `Filter` | 使用顺序扫描处理表，过滤符合条件的记录。没有去重操作。 |
| **有 DISTINCT 查询** | `Index Scan`<br/> + `Unique`<br/> + `Filter` | 使用索引扫描，额外进行去重操作（`Unique`<br/>）。执行时间更长。 |


+ **执行时间对比**
    - **无 DISTINCT 查询**：使用顺序扫描，执行时间为 48.135 毫秒。没有去重操作，所以相对较快。
    - **有 DISTINCT 查询**：使用索引扫描，但增加了去重操作（`DISTINCT`），执行时间为 113.043 毫秒。去重操作导致查询时间大大增加，尽管索引扫描本身通常较快，但 `DISTINCT` 操作增加了额外的开销。

### 结论
+ **无 DISTINCT 查询**：由于没有去重操作，查询执行时间较短，适合在无需去重的场合使用。
+ **有 DISTINCT 查询**：使用 `DISTINCT` 会导致查询执行时间显著增加，尤其是当去重操作需要额外的排序或去重步骤时。在某些情况下，尽管有索引扫描，但去重操作的额外开销使得整体查询效率较低。

---

## 3.9 union、union all 的区别
### 实验步骤
1. 编写两条查询语句：  
    - 一条使用 `UNION`。  
    - 一条使用 `UNION ALL`。
2. 比较两种方式的查询结果和执行性能。

### 实验代码
#### 使用 `UNION` 查询
```sql
EXPLAIN ANALYZE  
SELECT p_partkey FROM part  
UNION  
SELECT ps_partkey FROM partsupp;
```

#### 使用 `UNION ALL` 查询
```sql
EXPLAIN ANALYZE  
SELECT p_partkey FROM part  
UNION ALL  
SELECT ps_partkey FROM partsupp;
```

### 执行计划
#### 使用 `UNION` 查询
```sql
                                                                  QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=12112.52..14112.52 rows=200000 width=4) (actual time=191.195..194.706 rows=40000 loops=1)
   Group By Key: part.p_partkey
   ->  Append  (cost=0.00..11612.52 rows=200000 width=4) (actual time=0.150..170.226 rows=200004 loops=1)
         ->  Index Only Scan using part_pkey on part  (cost=0.00..1903.25 rows=40000 width=4) (actual time=0.148..8.650 rows=40000 loops=1)
               Heap Fetches: 40000
         ->  Bitmap Heap Scan on partsupp  (cost=2612.27..7709.27 rows=160000 width=4) (actual time=4.239..152.128 rows=160004 loops=1)
               ->  Bitmap Index Scan on partsupp_pkey  (cost=0.00..2572.27 rows=160000 width=0) (actual time=3.947..3.947 rows=160004 loops=1)
 Total runtime: 196.416 ms
(8 rows)
```

#### 使用 `UNION ALL` 查询
```sql
                                                                  QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------
 Result  (cost=0.00..9612.52 rows=200000 width=4) (actual time=0.041..45.067 rows=200004 loops=1)
   ->  Append  (cost=0.00..9612.52 rows=200000 width=4) (actual time=0.039..34.400 rows=200004 loops=1)
         ->  Index Only Scan using part_pkey on part  (cost=0.00..1903.25 rows=40000 width=4) (actual time=0.038..5.956 rows=40000 loops=1)
               Heap Fetches: 40000
         ->  Bitmap Heap Scan on partsupp  (cost=2612.27..7709.27 rows=160000 width=4) (actual time=3.232..19.204 rows=160004 loops=1)
               ->  Bitmap Index Scan on partsupp_pkey  (cost=0.00..2572.27 rows=160000 width=0) (actual time=2.945..2.945 rows=160004 loops=1)
 Total runtime: 50.268 ms
(7 rows)
```

### 结果分析
#### 使用 `UNION` 查询
+ **执行计划分析**：
    - **HashAggregate**：`UNION` 会去除重复的记录，因此执行计划中使用了 `HashAggregate` 来进行去重操作。
    - **Append**：执行计划中使用了 `Append`，将两个子查询的结果合并。
    - **Index Only Scan**：`part` 表使用了索引扫描。
    - **Bitmap Heap Scan**：`partsupp` 表使用了位图堆扫描，首先通过索引扫描找到符合条件的记录。
    - **Total runtime**：执行时间为 196.416 毫秒。
+ **执行计划和效率分析：**
    - `UNION` 操作会去除重复的记录，因此需要额外的去重步骤（`HashAggregate`）。去重操作增加了额外的开销，因此查询执行时间较长。

#### 使用 `UNION ALL` 查询
+ **执行计划分析**：
    - **Result**：`UNION ALL` 没有进行去重，因此只使用了 `Result` 操作来合并两个子查询的结果。
    - **Append**：同样使用了 `Append` 操作来合并两个子查询的结果。
    - **Index Only Scan**：`part` 表使用了索引扫描。
    - **Bitmap Heap Scan**：`partsupp` 表使用了位图堆扫描，首先通过索引扫描找到符合条件的记录。
    - **Total runtime**：执行时间为 50.268 毫秒。
+ **执行计划和效率分析：**
    - `UNION ALL` 不进行去重操作，因此没有 `HashAggregate` 步骤，执行效率较高。只需要合并两个查询的结果，减少了去重带来的开销，因此执行时间显著低于 `UNION`。

#### 执行计划对比
| 查询类型 | 执行计划 | 执行时间 | 说明 |
| --- | --- | --- | --- |
| **UNION 查询** | `HashAggregate`<br/> + `Append`<br/> + `Index Only Scan` | 196.416 ms | 需要去重操作，增加了额外的开销，执行时间较长。 |
| **UNION ALL 查询** | `Result`<br/> + `Append`<br/> + `Index Only Scan` | 50.268 ms | 不进行去重操作，执行时间较短。 |


+ **执行时间对比：**
    - **UNION 查询**：执行时间较长，主要因为它需要进行去重（`HashAggregate`）。虽然在两个表的查询结果中可能有重复的 `p_partkey`，但去重操作的增加了额外的计算开销。
    - **UNION ALL 查询**：执行时间较短，因为它不进行去重操作，直接将两个查询的结果合并，减少了计算量。

### 结论
+ **UNION**：适用于需要去重的场景，但由于去重操作带来的额外计算开销，会导致执行时间更长。
+ **UNION ALL**：适用于无需去重的场景，因为它跳过了去重步骤，通常执行速度更快。

---

## 3.10 from 中存在多余的关系表，即查询非最简化
### 实验步骤
1. 编写两条等价查询语句：  
    - 一条在 `FROM` 子句中加入多余的关系表，并在 `WHERE` 子句中增加对应的连接条件。  
    - 一条移除多余的关系表。
2. 执行查询并比较两种方式的执行计划和性能。

### 实验代码
#### 无多余关系表
```sql
EXPLAIN ANALYZE  
SELECT DISTINCT orders.o_custkey  
FROM lineitem, orders  
WHERE lineitem.l_orderkey = orders.o_orderkey;
```

#### 有多余关系表
```sql
EXPLAIN ANALYZE  
SELECT DISTINCT orders.o_custkey  
FROM lineitem, orders, part  
WHERE lineitem.l_orderkey = orders.o_orderkey  
  AND lineitem.l_partkey = part.p_partkey;
```

### 执行计划
#### 无多余关系表
```sql
                                                                        QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=96807.84..96991.50 rows=18366 width=4) (actual time=358.020..359.230 rows=19998 loops=1)
   Group By Key: orders.o_custkey
   ->  Merge Join  (cost=1.16..94210.78 rows=1038824 width=4) (actual time=0.050..271.497 rows=903853 loops=1)
         Merge Cond: (orders.o_orderkey = lineitem.l_orderkey)
         ->  Index Scan using orders_pkey on orders  (cost=0.00..12097.28 rows=300000 width=8) (actual time=0.010..53.182 rows=300001 loops=1)
         ->  Index Only Scan using lineitem_pkey on lineitem  (cost=0.00..68379.14 rows=1038824 width=4) (actual time=0.034..114.542 rows=903853 loops=1)
               Heap Fetches: 0
 Total runtime: 360.127 ms
(8 rows)
```

#### 有多余关系表
```sql
                                                              QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=138763.96..138947.62 rows=18366 width=4) (actual time=684.124..685.281 rows=19998 loops=1)
   Group By Key: orders.o_custkey
   ->  Hash Join  (cost=12788.00..136166.90 rows=1038824 width=4) (actual time=129.458..595.131 rows=903853 loops=1)
         Hash Cond: (lineitem.l_partkey = part.p_partkey)
         ->  Hash Join  (cost=11036.00..120131.07 rows=1038824 width=8) (actual time=117.737..455.655 rows=903853 loops=1)
               Hash Cond: (lineitem.l_orderkey = orders.o_orderkey)
               ->  Seq Scan on lineitem  (cost=0.00..94811.24 rows=1038824 width=8) (actual time=53.132..234.957 rows=903853 loops=1)
               ->  Hash  (cost=7286.00..7286.00 rows=300000 width=8) (actual time=63.706..63.706 rows=300001 loops=1)
                      Buckets: 524288  Batches: 1  Memory Usage: 11719kB
                     ->  Seq Scan on orders  (cost=0.00..7286.00 rows=300000 width=8) (actual time=0.003..33.202 rows=300001 loops=1)
         ->  Hash  (cost=1252.00..1252.00 rows=40000 width=4) (actual time=11.320..11.320 rows=40000 loops=1)
                Buckets: 65536  Batches: 1  Memory Usage: 1407kB
               ->  Seq Scan on part  (cost=0.00..1252.00 rows=40000 width=4) (actual time=0.012..6.847 rows=40000 loops=1)
 Total runtime: 686.363 ms
(14 rows)
```

### 结果分析
#### 无多余关系表
+ **执行计划分析：**
    - **Merge Join**：使用了 `Merge Join` 对 `orders` 和 `lineitem` 表进行连接。
    - **Index Scan** 和 **Index Only Scan**：对 `orders` 表和 `lineitem` 表都使用了索引扫描。
    - **Total runtime**：执行时间为 360.127 毫秒。

#### 有多余关系表
+ **执行计划分析：**
    - **Hash Join**：两个 `Hash Join` 操作。首先是将 `lineitem` 和 `orders` 表进行哈希连接，然后再将 `lineitem` 和 `part` 表进行哈希连接。
    - **Seq Scan**：对 `orders`、`lineitem` 和 `part` 表进行顺序扫描。
    - **Total runtime**：执行时间为 686.363 毫秒。

#### 执行结果和时间对比
| 查询类型 | 执行计划 | 执行时间 | 说明 |
| --- | --- | --- | --- |
| **查询 1**（多余关系表） | 使用了多个 `Hash Join`<br/> 和 `Seq Scan` | 686.363 ms | 增加了多余的 `part`<br/> 表，导致查询变慢。 |
| **查询 2**（最简化查询） | 使用了 `Merge Join`<br/> 和 `Index Scan` | 360.127 ms | 仅连接了 `orders`<br/> 和 `lineitem`<br/> 表，查询更快。 |


+ **执行时间对比**：
    - **查询 1**（多余关系表）：由于增加了 `part` 表，查询中执行了更多的连接操作（两个 `Hash Join`），导致查询时间显著增加。总执行时间为 686.363 毫秒。
    - **查询 2**（最简化查询）：仅连接了 `orders` 和 `lineitem` 表，查询执行更高效。总执行时间为 360.127 毫秒。
+ **执行计划对比**：
    - **查询 1** 使用了多余的 `part` 表，导致执行计划中增加了两个 `Hash Join` 操作。这不仅增加了计算开销，还导致了更多的数据处理。
    - **查询 2** 使用了 `Merge Join`，这是连接两个表时更高效的方式，特别是当两个表的连接条件有索引时，`Merge Join` 通常比 `Hash Join` 更快速。

### 结论
+ **多余关系表的影响**：
    - 增加不必要的表会导致更多的连接操作，这增加了执行计划的复杂性，特别是当这些表需要通过 `Hash Join` 或 `Merge Join` 进行连接时，计算量显著增加。
    - 在执行计划中，使用更多的表和更复杂的连接操作会增加查询的执行时间。
+ **最简化查询的优势**：
    - 最简化的查询只连接了必要的表，执行计划简单，使用了高效的 `Merge Join`，因此查询速度较快。


## 教材上的查询语句分析
查询1的执行计划如下，其中：cost=5,rows=20,predicate information中有一次索引access，一次全表搜索filter。
```
1- Original
-----------
Plan hash value: 3808327043

 
---------------------------------------------------------------------------------------------------
| Id  | Operation                     | Name              | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |                   |     1 |    23 |     5  (20)| 00:00:01 |
|   1 |  HASH GROUP BY                |                   |     1 |    23 |     5  (20)| 00:00:01 |
|   2 |   NESTED LOOPS                |                   |    19 |   437 |     4   (0)| 00:00:01 |
|   3 |    NESTED LOOPS               |                   |    20 |   437 |     4   (0)| 00:00:01 |
|*  4 |     TABLE ACCESS FULL         | DEPARTMENTS       |     2 |    32 |     3   (0)| 00:00:01 |
|*  5 |     INDEX RANGE SCAN          | EMP_DEPARTMENT_IX |    10 |       |     0   (0)| 00:00:01 |
|   6 |    TABLE ACCESS BY INDEX ROWID| EMPLOYEES         |    10 |    70 |     1   (0)| 00:00:01 |
---------------------------------------------------------------------------------------------------
 
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$1
   4 - SEL$1 / D@SEL$1
   5 - SEL$1 / E@SEL$1
   6 - SEL$1 / E@SEL$1
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   4 - filter("D"."DEPARTMENT_NAME"='IT' OR "D"."DEPARTMENT_NAME"='Sales')
   5 - access("D"."DEPARTMENT_ID"="E"."DEPARTMENT_ID")
 Column Projection Information (identified by operation id):
-----------------------------------------------------------
 
   1 - (#keys=1) "DEPARTMENT_NAME"[VARCHAR2,30], COUNT("E"."SALARY")[22], COUNT(*)[22], 
       SUM("E"."SALARY")[22]
   2 - (#keys=0) "D"."DEPARTMENT_ID"[NUMBER,22], "DEPARTMENT_NAME"[VARCHAR2,30], 
       "E"."SALARY"[NUMBER,22]
   3 - (#keys=0) "D"."DEPARTMENT_ID"[NUMBER,22], "DEPARTMENT_NAME"[VARCHAR2,30], 
       "E".ROWID[ROWID,10]
   4 - "D"."DEPARTMENT_ID"[NUMBER,22], "DEPARTMENT_NAME"[VARCHAR2,30]
   5 - "E".ROWID[ROWID,10]
   6 - "E"."SALARY"[NUMBER,22]
 
Note
-----
   - this is an adaptive plan
```
   查询2的执行计划如下，其中cost=7,rows=106,predicate information中的有一次索引搜索access，两次全表搜索filter。
   ```
   1- Original
-----------
Plan hash value: 2128232041

 
----------------------------------------------------------------------------------------------
| Id  | Operation                      | Name        | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |             |     1 |    23 |     7  (29)| 00:00:01 |
|*  1 |  FILTER                        |             |       |       |            |          |
|   2 |   HASH GROUP BY                |             |     1 |    23 |     7  (29)| 00:00:01 |
|   3 |    MERGE JOIN                  |             |   106 |  2438 |     6  (17)| 00:00:01 |
|   4 |     TABLE ACCESS BY INDEX ROWID| DEPARTMENTS |    27 |   432 |     2   (0)| 00:00:01 |
|   5 |      INDEX FULL SCAN           | DEPT_ID_PK  |    27 |       |     1   (0)| 00:00:01 |
|*  6 |     SORT JOIN                  |             |   107 |   749 |     4  (25)| 00:00:01 |
|   7 |      TABLE ACCESS FULL         | EMPLOYEES   |   107 |   749 |     3   (0)| 00:00:01 |
----------------------------------------------------------------------------------------------
 
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$1
   4 - SEL$1 / D@SEL$1
   5 - SEL$1 / D@SEL$1
   7 - SEL$1 / E@SEL$1
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter("DEPARTMENT_NAME"='IT' OR "DEPARTMENT_NAME"='Sales')
   6 - access("D"."DEPARTMENT_ID"="E"."DEPARTMENT_ID")
       filter("D"."DEPARTMENT_ID"="E"."DEPARTMENT_ID")
 
Column Projection Information (identified by operation id):
-----------------------------------------------------------
 
   1 - "DEPARTMENT_NAME"[VARCHAR2,30], COUNT("E"."SALARY")[22], COUNT(*)[22], 
       SUM("E"."SALARY")[22]
   2 - (#keys=1) "DEPARTMENT_NAME"[VARCHAR2,30], COUNT("E"."SALARY")[22], 
       COUNT(*)[22], SUM("E"."SALARY")[22]
   3 - (#keys=0) "D"."DEPARTMENT_NAME"[VARCHAR2,30], "E"."SALARY"[NUMBER,22]
   4 - "D"."DEPARTMENT_ID"[NUMBER,22], "D"."DEPARTMENT_NAME"[VARCHAR2,30]
   5 - "D".ROWID[ROWID,10], "D"."DEPARTMENT_ID"[NUMBER,22]
   6 - (#keys=1) "E"."DEPARTMENT_ID"[NUMBER,22], "E"."SALARY"[NUMBER,22]
   7 - "E"."SALARY"[NUMBER,22], "E"."DEPARTMENT_ID"[NUMBER,22]

-------------------------------------------------------------------------------
```
通过上面的两个执行计划的比较，我们可以看出查询1的查询语句比查询2的语句更优。因为查询1只有一次全表搜索，查询2有两次；并且查询1是先过滤后汇总（where语句），参与汇总计算的数据量少。而查询2是先汇总后过滤（having子句），参与汇总与计算的数据量多。

通过sqldeveloper中的优化指导，可看出仍有优化的必要。下图是给出的优化建议：
![图片加载失败](https://github.com/mxbox2/oracle/blob/master/test1/%E4%BC%98%E5%8C%96%E6%8C%87%E5%AF%BC.jpg)
在执行计划中给出了优化建议：
```
1- Index Finding (see explain plans section below)
--------------------------------------------------
  通过创建一个或多个索引可以改进此语句的执行计划。

  Recommendation (estimated benefit: 59.99%)
  ------------------------------------------
  - 考虑运行可以改进物理方案设计的访问指导或者创建推荐的索引。
    create index HR.IDX$$_00290001 on HR.DEPARTMENTS("DEPARTMENT_NAME","DEPARTM
    ENT_ID");

  Rationale
  ---------
    创建推荐的索引可以显著地改进此语句的执行计划。但是, 使用典型的 SQL 工作量运行 "访问指导"
    可能比单个语句更可取。通过这种方法可以获得全面的索引建议案, 包括计算索引维护的开销和附加的空间消耗。
 ```
## 自己的查询语句
各个部门平均工资和人数，按照部门名字升序排列：
![图片加载失败](https://github.com/mxbox2/oracle/blob/master/test1/my.jpg?raw=true)
该查询语句的执行计划为：
```
1- Original
-----------
Plan hash value: 1817899749

 
--------------------------------------------------------------------------------------
| Id  | Operation           | Name           | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT    |                |    27 |   621 |     5  (20)| 00:00:01 |
|   1 |  SORT GROUP BY      |                |    27 |   621 |     5  (20)| 00:00:01 |
|*  2 |   HASH JOIN         |                |   106 |  2438 |     4   (0)| 00:00:01 |
|   3 |    INDEX FULL SCAN  | IDX$$_006F0001 |    27 |   432 |     1   (0)| 00:00:01 |
|   4 |    TABLE ACCESS FULL| EMPLOYEES      |   107 |   749 |     3   (0)| 00:00:01 |
--------------------------------------------------------------------------------------
 
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$1
   3 - SEL$1 / DEPT@SEL$1
   4 - SEL$1 / EMP@SEL$1
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("EMP"."DEPARTMENT_ID"="DEPT"."DEPARTMENT_ID")
 
Column Projection Information (identified by operation id):
-----------------------------------------------------------
 
   1 - (#keys=1) "DEPT"."DEPARTMENT_NAME"[VARCHAR2,30], COUNT(*)[22], 
       COUNT("EMP"."SALARY")[22], SUM("EMP"."SALARY")[22]
   2 - (#keys=1) "DEPT"."DEPARTMENT_NAME"[VARCHAR2,30], 
       "EMP"."SALARY"[NUMBER,22]
   3 - "DEPT"."DEPARTMENT_ID"[NUMBER,22], 
       "DEPT"."DEPARTMENT_NAME"[VARCHAR2,30]
   4 - "EMP"."SALARY"[NUMBER,22], "EMP"."DEPARTMENT_ID"[NUMBER,22]

-------------------------------------------------------------------------------
```

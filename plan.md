# Reading Oracle execution plans

Каждый шаг плана проще всего описывать как некую функцию, вложенные шаги которой так же являются операциями-функциями, запускаемыми/останавливаемые этой функцией и постепенно возвращающими в нее свои результаты.

Т.е. каждая строка плана - это вызов соответствующей функции. 
Собственно, V$SQL_PLAN.OPERATION - это название функции, а V$SQL_PLAN.OPTIONS - это ее входной параметр, определяющий ее алгоритм/подфункцию.

***
Например, операция "TABLE ACCESS" имеет следующие варианты:

````
BY GLOBAL INDEX ROWID BATCHED
BY INDEX ROWID
BY INDEX ROWID BATCHED
BY LOCAL INDEX ROWID BATCHED
BY USER ROWID
CLUSTER
FULL
SAMPLE
````

А у INDEX:
````
FAST FULL SCAN
FULL SCAN
FULL SCAN (MIN/MAX)
RANGE SCAN
RANGE SCAN (MIN/MAX)
RANGE SCAN DESCENDING
SAMPLE FAST FULL SCAN
SKIP SCAN
UNIQUE SCAN
````

Т.к. алгоритмы совершенно разные, то и указывать только OPERATION без OPTIONS особого смысла не имеет, поэтому DBMS_XPLAN как и другие инструменты их конкатенируют, например `TABLE ACCESS FULL`, `TABLE ACCESS BY INDEX ROWID`, и тп. 
И мы, говоря о FULL TABLE SCAN, имеем ввиду функцию `TABLE ACCESS` с параметром `FULL`.
Соответствие названий некоторых операций и реальных сишных функций можно увидеть в скрипте у Танела Подера:
[os_explain](http://blog.tanelpoder.com/files/scripts/tools/unix/os_explain)

Короткий пример из os_explain:

```
s/qerbm/MINUS: /g;
s/qercb/CONNECT BY: /g;
s/qerco/COUNT: /g;
s/qerfu/UPDATE: /g;
s/qerfx/FIXED TABLE: /g;
s/qergr/GROUP BY ROLLUP: /g;
s/qergs/GROUP BY SORT: /g;
s/qerhj/HASH JOIN: /g;
s/qeril/IN-LIST: /g;
s/qerix/INDEX: /g;
s/qerjot/NESTED LOOP JOIN: /g;
s/qerjo/NESTED LOOP OUTER: /g;
s/qerns/GROUP BY NO SORT: /g;
s/qeroc/OBJECT COLLECTION ITERATOR: /g;
s/qerso/SORT: /g;
s/qertb/TABLE ACCESS: /g;
s/qerua/UNION-ALL: /g;
s/qergh/HASH GROUP BY: /g
```

Больше описаний функций можно найти в проекте Frits Hoogland [orafun.info](http://orafun.info/)
***

При необходимости каждая строка плана запускает, останавливает и обрабатывает дочерние операции. А каким именно образом и сколько раз она их запускает зависит от самой операции и ее параметров.

Например, HASH JOIN сначала вычитывает все результаты первой дочерней операции(build table) и только потом запускает вторую (prob table), a NESTED LOOPS получает строку из первой дочерней и для нее запускает вторую дочернюю, затем запускает снова первую, чтобы получить очередную строку и для нее снова вторую и тд...

Или еще более простой пример с select * from dual where 1=0:

```sql

PLAN:
---------------------------------------------------------------------------
| Id  | Operation          | Name | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |      |     1 |     2 |     0   (0)|          |
|*  1 |  FILTER            |      |       |       |            |          |
|   2 |   TABLE ACCESS FULL| DUAL |     1 |     2 |     2   (0)| 00:00:01 |
---------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter(NULL IS NOT NULL)

```

В данном случае операция FILTER не запускает дочерний TABLE ACCESS FULL, т.к. условие ложно.

***
Пример запроса и плана:
  
```sql
 SELECT E.employee_id, 
        E.last_name,
        d.department_name
 FROM employees E, 
      departments D
 WHERE E.department_id = D.department_id 
       AND upper(E.first_name) =  'GUY';

---------------------------------------------------------------------------------------------
Plan hash value: 3488509485                                                                  
                                                                                             
---------------------------------------------------------------------------------------------
| Id  | Operation                     | Name        | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |             |     1 |    43 |     3   (0)| 00:00:01 |
|   1 |  NESTED LOOPS                 |             |       |       |            |          |
|   2 |   NESTED LOOPS                |             |     1 |    43 |     3   (0)| 00:00:01 |
|   3 |    TABLE ACCESS BY INDEX ROWID| EMPLOYEES   |     1 |    27 |     2   (0)| 00:00:01 |
|*  4 |     INDEX RANGE SCAN          | FIRST_NAME  |     1 |       |     1   (0)| 00:00:01 |
|*  5 |    INDEX UNIQUE SCAN          | DEPT_ID_PK  |     1 |       |     0   (0)| 00:00:01 |
|   6 |   TABLE ACCESS BY INDEX ROWID | DEPARTMENTS |     1 |    16 |     1   (0)| 00:00:01 |
---------------------------------------------------------------------------------------------
                                                                                             
Predicate Information (identified by operation id):                                          
---------------------------------------------------                                          
                                                                                             
   4 - access(UPPER("FIRST_NAME")='GUY')                                                     
   5 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
```

0. Запускается SELECT, который запускает NESTED LOOPS из шага 1;
1. NESTED LOOPS шага 1, запускает NESTED LOOPS из шага 2 и для каждой возвращенной строки(ROWIDs) оттуда выполняет шаг 6, т.е. достает оттуда строки по найденным ROWID из шага 2;
2. NESTED LOOPS шага 2 запускает процедуру из шага 3 (TABLE ACCESS BY INDEX ROWID) и по возвращенным строкам оттуда выполняет шаг 5, т.е. фильтрует эти строки по INDEX UNIQUE SCAN индекса DEPT_ID_PK
3. Шаг 3 - TABLE ACCESS BY INDEX ROWID - запускает шаг 4(INDEX RANGE SCAN) и по возвращенным оттуда ROWID достает строки из таблицы EMPLOYEES
4. Шаг 4 сканирует индекс FIRST_NAME через IRS(index range scan) по предикату: access(UPPER("FIRST_NAME")='GUY')

***

Доп.материалы:

- [Execution Plans: Part 1 Finding plans](http://allthingsoracle.com/execution-plans-part-1-finding-plans/)
- [Execution Plans: Part 2: Things to see](http://allthingsoracle.com/execution-plans-part-2-things-to-see/) 
- [Execution Plans: Part 3: “The Rule”](http://allthingsoracle.com/execution-plans-part-3-the-rule/) 
- [Execution Plans: Part 4: Precision and Timing](http://allthingsoracle.com/execution-plans-part-4-precision-and-timing/) 
- [Execution Plans: part 5: First Child Variations](http://allthingsoracle.com/execution-plans-part-5-first-child-variations/) 
- [Execution Plans: Part 6: Pushed Subqueries](http://allthingsoracle.com/execution-plans-part-6-pushed-subqueries/) 
- [Execution Plans: Part 7: Query Blocks and Inline Views](http://allthingsoracle.com/execution-plans-part-7-query-blocks-and-inline-views/) 
- [Execution Plans: Part 8: Cost, time, etc.](http://allthingsoracle.com/execution-plans-part-8-cost-time-etc/) 
- [Execution Plans: Part 9: Multiplication](http://allthingsoracle.com/execution-plans-part-9-multiplication/) 
- [Execution Plans: Part 10: Guesswork](http://allthingsoracle.com/execution-plans-part-10-guesswork/)
- [Tracing Oracle SQL plan execution with DTrace](https://tanelpoder.com/2009/04/24/tracing-oracle-sql-plan-execution-with-dtrace/)
- [https://tanelpoder.com/2008/06/15/advanced-oracle-troubleshooting-guide-part-6-understanding-oracle-execution-plans-with-os_explain/](https://tanelpoder.com/2008/06/15/advanced-oracle-troubleshooting-guide-part-6-understanding-oracle-execution-plans-with-os_explain/)
- [скрипт os_explain с маппингом функций к названиям операций](https://github.com/tanelpoder/tpt-oracle/blob/master/tools/unix/os_explain)

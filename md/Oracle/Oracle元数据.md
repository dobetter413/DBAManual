# Oracle 

## 表信息统计

1. 表大小统计
   ```
    select t.segment_name, t.segment_type, sum(t.bytes / 1024 / 1024) "占用空间(M)"
    from dba_segments t
    where t.segment_type='TABLE'
    and t.segment_name ='GDMS_OPINION_COMEFORM'
    group by OWNER, t.segment_name, t.segment_type;
   ```
2. 统计Oracle 数据库用户所有表的大小
   ```
    SELECT OWNER as "username",sum(BYTES)/1024/1024/1024 as "alltablesize(GB)" FROM DBA_SEGMENTS  WHERE SEGMENT_NAME in (select t2.OBJECT_NAME from dba_objects t2 where t2.OBJECT_TYPE = 'TABLE') group by OWNER order by 2 desc;
   ```

3. 查看每个表空间的大小
    ```
     Select Tablespace_Name,Sum(bytes)/1024/1024  as "tbs(M)"From Dba_Segments Group By Tablespace_Name order by 2 desc;
    ```


## Oracle 对象表
1. 查看对象类型
`select OBJECT_NAME,OBJECT_TYPE from all_objects where OWNER='COSPACE' and OBJECT_NAME in ('ACT_TMP','V_WORKFLOW_TODO_TASK','V_WORKFLOW_PROCDEF');`

2. 查看库下所有对象
```
select OBJECT_NAME,OBJECT_TYPE from all_objects where OWNER='GOVERNMENT' and OBJECT_TYPE not in ('INDEX','LOB');
```
3. 统计库下各对象数
```
select OBJECT_TYPE,count(OBJECT_NAME) from all_objects where OWNER='GOVERNMENT' and OBJECT_TYPE not in ('INDEX','LOB') group by OBJECT_TYPE;
```
4. 某一类对象列表
```
select OBJECT_NAME from all_objects where OWNER='GOVERNMENT' and OBJECT_TYPE='TABLE';
```
5. 重名对象
```
select OBJECT_NAME,count(OBJECT_NAME) from all_objects where OWNER='GOVERNMENT' and OBJECT_TYPE not in ('INDEX','LOB') group by OBJECT_NAME having count(OBJECT_NAME)>1;
```

## 字段类型查询
1. 查看字段长度超过200 的表名和字段
```
select TABLE_NAME,COLUMN_NAME,DATA_TYPE,DATA_LENGTH FROM user_tab_columns WHERE DATA_LENGTH >200;
```

2. 查看指定用户字段信息
   + 查看指定用户下包含LOB 类型的字段
    ```
    select TABLE_NAME,COLUMN_NAME,DATA_TYPE FROM all_tab_columns WHERE DATA_TYPE like '%LOB%' and owner='SXML_CY';
    ```

  + 查看指定用户下，含有LOB 字段的表和每个表LOB字段的个数
    ```
    select TABLE_NAME,count(COLUMN_NAME) FROM all_tab_columns WHERE DATA_TYPE like '%LOB%' and owner='SXML_CY' group by TABLE_NAME;
    ```


## 字符集配置
select userenv("language") from dual;;



## 表DDL

1. 查看指定用户下的建表语句
   `select dbms_metadata.get_ddl('TABLE','tablename','username') from dual;`
2. 查看指定用户下的约束DDL语句
   `select dbms_metadata.get_ddl('CONSTRAINT','tablename','username') from dual;`
3. 查看指定用户下的索引DDL语句
   `select dbms_metadata.get_ddl('INDEX','tablename','username') from dual;`

## 约束元数据信息

1. 主键约束
    + 查看指定表的主键列
    ```
    SELECT CU.OWNER,CU.CONSTRAINT_NAME,CU.TABLE_NAME,CU.COLUMN_NAME,CU.POSITION FROM DBA_CONS_COLUMNS CU, DBA_CONSTRAINTS AU
    WHERE CU.CONSTRAINT_NAME = AU.CONSTRAINT_NAME
    AND AU.CONSTRAINT_TYPE = 'P'
    AND AU.TABLE_NAME = 'T_ITEM_PROCESS_SUB' 
    AND CU.OWNER='SXML_CY'; 
   SELECT CU.OWNER,CU.CONSTRAINT_NAME,CU.TABLE_NAME,CU.COLUMN_NAME,CU.POSITION FROM ALL_CONS_COLUMNS CU, ALL_CONSTRAINTS AU
    WHERE CU.CONSTRAINT_NAME = AU.CONSTRAINT_NAME
    AND AU.CONSTRAINT_TYPE = 'P'
    AND AU.TABLE_NAME = 'T_ITEM_PROCESS_SUB' 
    AND CU.OWNER='SXML_CY'; 
    ```
    + 查看含有主键的表
    ```
    SELECT CU.OWNER,CU.CONSTRAINT_NAME,CU.TABLE_NAME,CU.COLUMN_NAME,CU.POSITION  FROM DBA_CONS_COLUMNS CU, DBA_CONSTRAINTS AU
    WHERE CU.CONSTRAINT_NAME = AU.CONSTRAINT_NAME
    AND AU.CONSTRAINT_TYPE = 'P'
    AND CU.OWNER='SXML_CY'; 
    ```
    + 查看不含主键的表
    ```
        select owner,table_name
          from all_tables a
         where not exists (select *
          from DBA_constraints b
         where b.constraint_type = 'P'
           and a.table_name = b.table_name)  and owner='SXML_CY';
    ```



### 附录
1. 无LOB 有主键表
```
/*无LOB 有主键*/
SELECT
 AAA.TABLE_NAME
FROM
 (
 SELECT
  table_name
 FROM
  (
  SELECT
   TABLE_NAME
  FROM
   all_tab_columns a
  WHERE
   NOT EXISTS (
   SELECT
    table_name
   FROM
    all_tab_columns b
   WHERE
    DATA_TYPE LIKE '%LOB%'
    AND owner = 'COSPACE'
    AND a.table_name = b.table_name
    AND a.owner = b.owner)
   AND a.owner = 'COSPACE'
  GROUP BY
   TABLE_NAME)aa
 JOIN all_objects bb ON
  aa.table_name = bb.object_name
  AND bb.object_type = 'TABLE'
  AND bb.owner = 'CAOMENG') AAA,
 (
 SELECT
  CU.OWNER,
  CU.TABLE_NAME,
  CU.CONSTRAINT_NAME
 FROM
  DBA_CONS_COLUMNS CU,
  DBA_CONSTRAINTS AU
 WHERE
  CU.CONSTRAINT_NAME = AU.CONSTRAINT_NAME
  AND AU.CONSTRAINT_TYPE = 'P'
  AND CU.OWNER = 'CAOMENG'
  AND CU.POSITION = 1) BBB
WHERE
 AAA.TABLE_NAME = BBB.TABLE_NAME GROUP BY AAA.TABLE_NAME;
```

2. 无LOB 有主键表
```
/*无LOB 无主键*/
SELECT
 AAA.TABLE_NAME
FROM 
 (
 SELECT
  table_name
 FROM
  (
  SELECT
   TABLE_NAME
  FROM
   all_tab_columns a
  WHERE
   NOT EXISTS (
   SELECT
    table_name
   FROM
    all_tab_columns b
   WHERE
    DATA_TYPE LIKE '%LOB%'
    AND owner = 'COSPACE'
    AND a.table_name = b.table_name
    AND a.owner = b.owner)
   AND a.owner = 'COSPACE'
  GROUP BY
   TABLE_NAME)aa
 JOIN all_objects bb ON
  aa.table_name = bb.object_name
  AND bb.object_type = 'TABLE'
  AND bb.owner = 'CAOMENG') AAA,
 (
 SELECT
  A.OWNER,
  A.TABLE_NAME
 FROM
  ALL_TABLES A
 WHERE
  NOT EXISTS 
 (
  SELECT
   B.OWNER,
   B.TABLE_NAME
  FROM
   DBA_CONSTRAINTS B
  WHERE
   B.CONSTRAINT_TYPE = 'P'
   AND A.TABLE_NAME = B.TABLE_NAME
   AND A.OWNER = B.OWNER)
  AND A.OWNER = 'CAOMENG') BBB
WHERE
 AAA.TABLE_NAME = BBB.TABLE_NAME GROUP BY AAA.TABLE_NAME;

```

3. 有LOB 有主键表 
```
/*有LOB 有主键*/
SELECT AAA.TABLE_NAME 
FROM 
 (
 SELECT
 TABLE_NAME
FROM
 all_tab_columns a
JOIN all_objects b ON
 a.owner = b.owner
 AND a.table_name = b.object_name
WHERE
 b.object_type = 'TABLE'
 AND a.DATA_TYPE LIKE '%LOB%'
 AND a.owner = 'COSPACE'
GROUP BY
 TABLE_NAME
 ) AAA,
 (
 SELECT
 CU.OWNER,
 CU.TABLE_NAME,
 CU.CONSTRAINT_NAME
FROM
 DBA_CONS_COLUMNS CU,
 DBA_CONSTRAINTS AU
WHERE
 CU.CONSTRAINT_NAME = AU.CONSTRAINT_NAME
 AND AU.CONSTRAINT_TYPE = 'P'
 AND CU.OWNER = 'CAOMENG'
 AND CU.POSITION=1
 ) BBB
WHERE
 AAA.TABLE_NAME = BBB.TABLE_NAME GROUP BY AAA.TABLE_NAME;
```

4. 有LOB 无主键表
```
/*有LOB 无主键*/
SELECT AAA.TABLE_NAME 
FROM 
 (
 SELECT
 TABLE_NAME
FROM
 all_tab_columns a
JOIN all_objects b ON
 a.owner = b.owner
 AND a.table_name = b.object_name
WHERE
 b.object_type = 'TABLE'
 AND a.DATA_TYPE LIKE '%LOB%'
 AND a.owner = 'COSPACE'
GROUP BY
 TABLE_NAME
 ) AAA,
 (
 SELECT 
 A.OWNER,
 A.TABLE_NAME
FROM 
 ALL_TABLES A
WHERE
 NOT EXISTS 
 (
 SELECT
  B.OWNER,
  B.TABLE_NAME
 FROM
  DBA_CONSTRAINTS B
 WHERE
  B.CONSTRAINT_TYPE = 'P'
  AND A.TABLE_NAME = B.TABLE_NAME
  AND A.OWNER = B.OWNER)
 AND A.OWNER = 'CAOMENG'
 ) BBB
WHERE
 AAA.TABLE_NAME = BBB.TABLE_NAME GROUP BY AAA.TABLE_NAME;
```
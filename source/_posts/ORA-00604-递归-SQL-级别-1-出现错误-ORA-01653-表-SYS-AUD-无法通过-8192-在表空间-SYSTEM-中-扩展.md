---
title: ' ORA-00604: 递归 SQL 级别 1 出现错误 ORA-01653: 表 SYS.AUD$ 无法通过 8192 (在表空间 SYSTEM 中) 扩展'
date: 2018-10-25 16:35:58
tags: Oracle 运维
categories: Technology
---

今日遇到OA数据库报错，报错内容如下：

```
ORA-00604: 递归 SQL 级别 1 出现错误
ORA-01653: 表 SYS.AUD$ 无法通过 8192 (在表空间 SYSTEM 中) 扩展
ORA-02002: 写入审计线索时出错
ORA-00604: 递归 SQL 级别 1 出现错误
```

这种情况是遇到了表空间已满的情况，由于Windows版32位的Oracle限制每个表空间存储文件最大32G，所以必须手动增加表空间存储文件进行扩容。步骤如下：

1. 本地模式登录数据库

   ```sql
   C:\Users\Administrator>sqlplus / as sysdba
   
   SQL*Plus: Release 11.2.0.1.0 Production on Thu Oct 25 16:16:49 2018
   
   Copyright (c) 1982, 2010, Oracle.  All rights reserved.
   
   
   Connected to:
   Oracle Database 11g Enterprise Edition Release 11.2.0.1.0 - Production
   With the Partitioning, OLAP, Data Mining and Real Application Testing options
   
   ```

   

2. 查看表空间情况

   ```plsql
   SQL> SELECT UPPER(F.TABLESPACE_NAME) "表空间名", D.TOT_GROOTTE_MB "表空间大小(M)
   ", D.TOT_GROOTTE_MB - F.TOTAL_BYTES "已使用空间(M)", TO_CHAR(ROUND((D.TOT_GROOTT
   E_MB - F.TOTAL_BYTES) / D.TOT_GROOTTE_MB * 100,2),'990.99') "使用比", F.TOTAL_BY
   TES "空闲空间(M)", F.MAX_BYTES "最大块(M)" FROM (SELECT TABLESPACE_NAME, ROUND(S
   UM(BYTES) / (1024 * 1024), 2) TOTAL_BYTES, ROUND(MAX(BYTES) / (1024 * 1024), 2)
   MAX_BYTES FROM SYS.DBA_FREE_SPACE GROUP BY TABLESPACE_NAME) F, (SELECT DD.TABLES
   PACE_NAME, ROUND(SUM(DD.BYTES) / (1024 * 1024), 2) TOT_GROOTTE_MB FROM SYS.DBA_D
   ATA_FILES DD GROUP BY DD.TABLESPACE_NAME) D WHERE D.TABLESPACE_NAME = F.TABLESPA
   CE_NAME ORDER BY 4 DESC;
   
   表空间名                       表空间大小(M) 已使用空间(M) 使用比  空闲空间(M)
   ------------------------------ ------------- ------------- ------- -----------
    最大块(M)
   ----------
   SYSTEM                                 65482      65409.19   99.89       72.81
           47
   
   OA                                      1248       1184.94   94.95       63.06
           63
   
   SYSAUX                                   920        852.06   92.62       67.94
           59
   
   
   表空间名                       表空间大小(M) 已使用空间(M) 使用比  空闲空间(M)
   ------------------------------ ------------- ------------- ------- -----------
    最大块(M)
   ----------
   UNDOTBS1                                1000         420.5   42.05       579.5
          353
   
   USERS                                      5          1.31   26.20        3.69
         3.69
   ```

   

3. 查看表空间文件名和是否自动增长

   ```plsql
   SQL> SELECT FILE_NAME,TABLESPACE_NAME,AUTOEXTENSIBLE FROM dba_data_files;
   
   FILE_NAME
   --------------------------------------------------------------------------------
   
   TABLESPACE_NAME                AUT
   ------------------------------ ---
   D:\APP\ADMINISTRATOR\ORADATA\ORCL\USERS01.DBF
   USERS                          YES
   
   D:\APP\ADMINISTRATOR\ORADATA\ORCL\UNDOTBS01.DBF
   UNDOTBS1                       YES
   
   D:\APP\ADMINISTRATOR\ORADATA\ORCL\SYSAUX01.DBF
   SYSAUX                         YES
   
   
   FILE_NAME
   --------------------------------------------------------------------------------
   
   TABLESPACE_NAME                AUT
   ------------------------------ ---
   D:\APP\ADMINISTRATOR\ORADATA\ORCL\SYSTEM01.DBF
   SYSTEM                         YES
   
   D:\APP\ADMINISTRATOR\ORADATA\ORCL\OA.DBF
   OA                             YES
   
   D:\APP\ADMINISTRATOR\ORADATA\ORCL\SYSTEM02.DBF
   SYSTEM                         NO
   
   
   6 rows selected.
   
   ```

   

4. 增加表空间的存储文件，system03.dbf

   ```plsql
   SQL> ALTER TABLESPACE SYSTEM ADD DATAFILE 'D:\app\Administrator\oradata\orcl\SYS
   TEM03.DBF' SIZE 32752M;
   
   Tablespace altered.
   
   ```

   

5. 执行完成后查看表空间的使用情况

   ```plsql
   SQL> SELECT UPPER(F.TABLESPACE_NAME) "表空间名", D.TOT_GROOTTE_MB "表空间大小(M)
   ", D.TOT_GROOTTE_MB - F.TOTAL_BYTES "已使用空间(M)", TO_CHAR(ROUND((D.TOT_GROOTT
   E_MB - F.TOTAL_BYTES) / D.TOT_GROOTTE_MB * 100,2),'990.99') "使用比", F.TOTAL_BY
   TES "空闲空间(M)", F.MAX_BYTES "最大块(M)" FROM (SELECT TABLESPACE_NAME, ROUND(S
   UM(BYTES) / (1024 * 1024), 2) TOTAL_BYTES, ROUND(MAX(BYTES) / (1024 * 1024), 2)
   MAX_BYTES FROM SYS.DBA_FREE_SPACE GROUP BY TABLESPACE_NAME) F, (SELECT DD.TABLES
   PACE_NAME, ROUND(SUM(DD.BYTES) / (1024 * 1024), 2) TOT_GROOTTE_MB FROM SYS.DBA_D
   ATA_FILES DD GROUP BY DD.TABLESPACE_NAME) D WHERE D.TABLESPACE_NAME = F.TABLESPA
   CE_NAME ORDER BY 4 DESC;
   
   表空间名                       表空间大小(M) 已使用空间(M) 使用比  空闲空间(M)
   ------------------------------ ------------- ------------- ------- -----------
    最大块(M)
   ----------
   OA                                      1248       1184.94   94.95       63.06
           63
   
   SYSAUX                                   920        852.06   92.62       67.94
           59
   
   SYSTEM                                 98234      65474.19   66.65    32759.81
         3968
   
   
   表空间名                       表空间大小(M) 已使用空间(M) 使用比  空闲空间(M)
   ------------------------------ ------------- ------------- ------- -----------
    最大块(M)
   ----------
   UNDOTBS1                                1000         460.5   46.05       539.5
          353
   
   USERS                                      5          1.31   26.20        3.69
         3.69
   
   ```

   可见SYSTEM表空间已经扩大了，任务完成。
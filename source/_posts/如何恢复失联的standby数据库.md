---
title: 如何恢复失联的standby数据库
date: 2018-04-27 16:26:38
tags: 
    - dataguard
    - restore
catagories: 
    - oracle
---

> DMS的DG备库失联很久了，导致将standby数据库重新打卡时，发现apply出现了gap，即有一段归档日志已经没有了，但是却还没有应用。这篇文章讲述一下如何恢复这个备库。


DMS的备库基本上用不到，平时只是作为一个数据库备份，所以可以随意操作。由于出现了Gap，所以需要重做standby备库，但是原本我学习的建立备库的方式是通过在主库RMAN上复制到备库。此次操作的原则就是要不影响主库，所以尝试使用主库的RMAN文件进行恢复。  
**！！！！注意点**：下面所有的操作都假设主库和备库的文件夹结构是**一模一样**的，如果不是，请参考此文<https://blogs.oracle.com/database4cn/rmandataguardstandby>中间一段的说明进行配置！
<!--more-->
# 1. 恢复standby控制文件
由于standby控制文件"失修"已久，所以需要从主库那边复制一个控制文件过来（控制文件会记录文件夹结构，如果备库和主库不一样，在恢复前一定要修改数据文件位置）。在`主库`的RMAN中执行下面的语句（这也是主库唯一需要执行的语句，对主库影响忽略不计）：
```sql
RMAN> BACKUP CURRENT CONTROLFILE FOR STANDBY FORMAT '/tmp/ForStandbyCTRL.bck';
```
将上面的控制文件传输到备库。上述语句中的**FOR STANDBY**不能漏掉。然后将所有的备份包括归档日志也传输到备库（本次不需要，因为所有的备份在备库上都有一个拷贝）。在备库中执行下面的语句添加备份文件：
```sql
CATALOG START WITH 'XXXXXXXX';
```
上面的XXX就是备份的文件夹。弄好之后在RMAN中执行下面的语句导入控制文件：
```sql
RMAN> SHUTDOWN IMMEDIATE ;
RMAN> STARTUP NOMOUNT;
RMAN> RESTORE STANDBY CONTROLFILE FROM '/tmp/ForStandbyCTRL.bck';
RMAN> SHUTDOWN;
RMAN> STARTUP MOUNT;
```
导入控制文件需要nomount，恢复数据库需要mount，这里提醒一下。另外我这里踩了一个坑，恢复控制文件的时候没有输入STANDBY关键字，导致最后备库变成了一个独立的库，这里提醒一下，一定是**RESTORE STANDBY CONTROLFILE**。

# 2. 恢复数据库

下面开始恢复数据库，恢复数据库之前先查询一下归档日志到哪个SCN或者Sequence了（本次肯定是不完全恢复了，因为控制文件和备份都不是一起的，但是对最终结果没有影响，因为所有丢失的数据都会通过apply的方式找回）。用下面的语句查询：
```sql
RMAN> LIST BACKUP OF ARCHIVELOG ALL;
```

我的输出如下(只截取了最后一部分)：
```
BS Key  Size       Device Type Elapsed Time Completion Time
------- ---------- ----------- ------------ ---------------
8134    81.08M     DISK        00:00:01     26-APR-18      
        BP Key: 41094   Status: AVAILABLE  Compressed: NO  Tag: WEEKLY INC1 BACKUP
        Piece Name: /u1/db/oracle/rmanbak/20180426_inc1_02t18vah_1_1.bkp

  List of Archived Logs in backup set 8134
  Thrd Seq     Low SCN    Low Time  Next SCN   Next Time
  ---- ------- ---------- --------- ---------- ---------
  1    25746   1250471243 26-APR-18 1250475702 26-APR-18
  1    25747   1250475702 26-APR-18 1250479196 26-APR-18
  1    25748   1250479196 26-APR-18 1250481461 26-APR-18
```
可以看到最后的sequence是25748，然后执行不完全恢复，命令如下：
```sql
run  {
    set until sequence 25749;
    restore database;
    recover database;
    }
```

经过一段时间等待之后大功告成，然后进入SQLPLUS中将备库开启。开启的命令如下：
```sql
SHU IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE OPEN READ ONLY;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT FROM SESSION;
```
然后查看一下备库是否起来：
```
SQL> select open_mode from v$database;

OPEN_MODE
--------------------
READ ONLY WITH APPLY

SQL> SELECT sequence#, first_time, next_time, applied FROM v$archived_log ORDER BY sequence#;

 SEQUENCE# FIRST_TIM NEXT_TIME APPLIED
---------- --------- --------- ---------
     25746 26-APR-18 26-APR-18 YES
     25747 26-APR-18 26-APR-18 YES
     25748 26-APR-18 26-APR-18 YES
     25749 26-APR-18 26-APR-18 YES
     25750 26-APR-18 26-APR-18 YES
     25751 26-APR-18 26-APR-18 YES
     25752 26-APR-18 26-APR-18 YES
     25753 26-APR-18 26-APR-18 YES
     25754 26-APR-18 26-APR-18 YES
     25755 26-APR-18 26-APR-18 YES
     25756 26-APR-18 26-APR-18 YES

```
过一段时间看到全部APPLY的时候就表示已经成功恢复了。

# 3. 总结

本文介绍了如何利用备份重建STANDBY数据库，当然，如果能够在主库进行操作，那么我们只需要将丢失的那部分数据通过增量备份的方式传输给备库就可以了，这样的话备库就不需要像上面一样进行整库还原了（DMS整库还原很慢）。此次由于不想影响主库所以没有尝试这个操作，下次可以试试。
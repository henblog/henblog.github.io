---
title: Oracle归档日志总结
date: 2022-08-01 17:21:00
author: hen
top: false
hide: false
cover: false
mathjax: false
summary: 总结Oracle使用场景以及如何使用
categories: Oracle
tags:
  - Oracle
  - 日志
  - 数据库
---

# Oracle归档日志总结

## 介绍

归档日志(Archive Log)是**非活动的重做日志备份**.通过使用归档日志,可以保留所有重做历史记录,当数据库处于`ARCHIVELOG`模式并进行日志切换时,后台进程ARCH会将重做日志的内容保存到归档日志中.当数据库出现介质失败时,使用数据文件备份,归档日志和重做日志可以完全恢复数据库。

## 归档模式开启与关闭

- 查询是否开启归档

  ```cmd
  [oracle@osc ~]$ sqlplus / as sysdba
  
  SQL*Plus: Release 11.2.0.4.0 Production on Mon Nov 12 17:36:13 2018
  
  Copyright (c) 1982, 2013, Oracle.  All rights reserved.
  
  Connected to:
  Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
  With the Partitioning, OLAP, Data Mining and Real Application Testing options
  
  SQL> archive log list; 
  Database log mode              No Archive Mode
  Automatic archival             Disabled
  Archive destination            USE_DB_RECOVERY_FILE_DEST
  Oldest online log sequence     1124
  Current log sequence           1126
  ```

  `No Archive Mode`、`Disabled` 表示未开启归档

- 开启归档

  ```sql
  SQL> shutdown immediate;
  Database closed.
  Database dismounted.
  ORACLE instance shut down.
  SQL>  startup mount;
  ORACLE instance started.
  
  Total System Global Area 2421825536 bytes
  Fixed Size                  2255632 bytes
  Variable Size             620758256 bytes
  Database Buffers         1778384896 bytes
  Redo Buffers               20426752 bytes
  Database mounted.
  SQL> alter database archivelog;
  
  Database altered.
  
  SQL> alter database open;
  
  Database altered.
  ```

- 关闭归档



## 归档日志空间不足解决方法

如果归档空间不足，将无法正常登入ORACLE，通过错误日志 `oracle_ora_错误码.trc` 

```sql
show parameter background_dump_dest --获取错误日志
```

- `ORA-01034`
- `ORA-27101`
- `ORA-00257: archiver error. Connect internal only, until freed`

由于归档日志太多，占用了全部日志存储空间。

### 解决方法

1、先cmd命令连接到数据库：有多个数据库需要指定连接的实例SID

```sql
sqlplus /@orcl as sysdba --最高权限连接到orcl实例
```

2、连接到实例后，确定下是否是我们所需要处理的数据库实例：

```sql
select instance_name from v$instance; --查看当前连接的数据库的sid
```

3、确认是后，先关闭例程，再启动例程（相当于初始化环境，排除干扰）

```sql
shutdown abort;

startup mount; 
```

4、查看下归档日志空间情况：

```sql
select * from  v$recovery_file_dest;
```

#### 1.关闭归档

如果业务不需要用到归档日志，可以选择关闭归档日志

#### 2.增大归档空间

```sql
sqlplus / as sysdba
shutdown abort     ----关闭进程
startup mount       ---- 装载数据库
select * from v$recovery_file_dest; ---查询归档日志
db_recovery_file_dest_size=10737418240; --设置归档日志空间为10G
Exit ---到这里空间大小已经设置完成
```

#### 3.删除归档日志

**注意提前备份**

```sql
sqlplus /nolog  --登录Oracle

SQL> connect /as sysdba  --连接用户

show parameter recover;  --查询日志目录位置

-- 将日志目录指向的归档文件删除

rman target sys/pass  --用RMAN维护控制文件

RMAN> crosscheck archivelog all;  --检查一些无用的archivelog

RMAN> delete archivelog until time 'sysdate-1' ;  --删除截止到前一天的所有archivelog
```

#### Windows自动删除归档日志，设置定时任务：

1. 创建一个删除归档日志的脚本（delete_arch.txt）：

    ```txt
    connect target /
    run{
    DELETE ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-7'; 
    //删除7天前的归档日志,怕哪天DG有问题，有日志没有及时应用
    crosscheck archivelog all;
    delete expired archivelog all;
    }
    ```

2.创建批处理任务（delete_archive.bat）

```txt
rman cmdfile=e:\delete_arch.txt
```

3.创建一个windows任务定时调用批处理任务

    **开始 => 所有程序 => 附件 => 系统工具 => 任务计划**

新建个任务计划了，然后根据要求配置下即可。

### 优化建议：

对于出现这种情况，归根结底是磁盘无法放下过多的归档日志，可以考虑：

1、挂接单独的磁盘组存储归档日志，并同步进行备份（推荐）。

2、对当前磁盘组进行扩容。

3、定期进行人工检查，删除部分归档日志以保证磁盘空间保持有剩余状态，或者设置自动删除较早的归档日志。
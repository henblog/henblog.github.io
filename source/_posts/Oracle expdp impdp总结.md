---
title: Oracle expdp/impdp总结
date: 2022-07-08 09:25:00
author: hen
top: true
hide: false
cover: true
mathjax: false
summary: 本文提供了expdp/impdp的快速上手和参数，还有一些小技巧和经验
categories: Oracle
tags:
  - oracle
  - 数据迁移
---

# Oracle expdp/impdp总结

## 介绍

数据泵（expdp，impdp）是Oracle 10g时引入的新技术，兼容了之前的数据导出导入工具（exp，imp）大部分功能，并进一步完善，提供了很多新功能以满足复杂的业务需求。区别于传统的exp，imp工具，数据泵相关命令需在数据库**服务端**执行。

数据泵属于逻辑迁移，可跨操作系统版本，跨数据库版本。高版本兼容低版本，高版本向低版本导数据，导出时需添加低版本的版本号。

## 快速开始

本篇适用于常用的导出导入命令，如果你只想将本库的数据导出到文件然后将文件导入到库，本篇可以作为参考。

本例将密码为**test**的用户**C##TEST**中的**testcar**表按照**条件**导出到指定目录，并导入后**更名**为**newtestcar**。

1. 在操作系统创建文件目录用来保存导出的dump文件(Windows系统直接图形化建立路径即可)；

   ```plsql
   mkdir -p d:\test\dump
   ```

2. 将目录指定为数据库的目录路径对象；

   ```plsql
   create directory dptest02 as 'd:\test\dump';
   ```

3. 为用户赋予读写目录对象的权限；

   ```plsql
   grant read,write on directory dptest02 to C##TEST;
   ```

4. 导出；

   ```basic
   expdp C##TEST/test@orcl directory=dptest02 tables=testcar query=\"where car_buy_time between to_date('2018-01-01 0:00:00','yyyy-mm-dd HH24:mi:ss') and to_date('2019-12-31 23:59:59','yyyy-mm-dd HH24:mi:ss')\" dumpfile=testcar%U.dmp parallel=10 job_name=jobtest01
   ```

5. 导入；

   ```basic
   impdp C##TEST/test directory=dp_test01 dumpfile=testcar%U.dmp remap_table=C##TEST.testcar:newtestcar
   ```

总文件113G ,条件导出expdp按照条件导出49G用时25分，impdp导入用时26分，并行10个进程，共231938000行；

## 准备工作

### 1、目标新库上的操作

（1）创建临时表空间

```plsql
create temporary tablespace 用户临时表空间名称 
tempfile '/u01/tablespaces/user_temp.dbf' 
size 50m 
autoextend on 
next 50m maxsize 20480m 
extent management local; 
```

备注：根据实际情况调整表空间大小等参数以及规划临时表空间、回滚表空间等相关规划。

（2）创建数据表空间

```plsql
create tablespace 用户表空间名称 
datafile '/u01/tablespaces/user_data.dbf' 
size 50m 
autoextend on 
next 50m maxsize 20480m 
extent management local; 
```

（3）建立用户，并指定默认表空间

```
create user 用户名称 identified by 密码 
default tablespace 用户表空间名称 
temporary tablespace 用户临时表空间名称;
```

（4）给用户授予权限

```
grant connect,resource to 用户名1;
grant create database link to 用户名;
```

注意：赋权给多个用户的情况下，各个用户名称间用,分隔即可。

（5）登录需要创建dblink的用户，创建dblink

```plsql
CREATE DATABASE LINK DBLink名称 
CONNECT TO 用户 IDENTIFIED BY 密码 
USING '(DESCRIPTION =(ADDRESS_LIST =(ADDRESS =(PROTOCOL = TCP)(HOST = XXX.XXX.XXX.XXX)(PORT = 1521)))(CONNECT_DATA =(SERVICE_NAME = 实例名)))';
```

注意：创建DBLINK默认是用户级别的，对当前用户有效。只有当需要对所有用户有效时，再建立公有的DBlink对象（pulic参数）。



### 2、创建数据备份目录（源库和目标库）

备份目录需要使用操作系统用户创建一个真实的目录，然后登录oracle dba用户，创建逻辑目录，指向该路径并使用户有对其读写权限。这样oracle才能识别这个备份目录。

（1）在操作系统上建立真实的目录

```
$ mkdir -p d:\test\dump
```

（2）登录oracle管理员用户

```qlsql
SQL> conn /as sysdba
```

（3）创建逻辑目录

```sql
SQL> create directory dp_name as 'd:\test\dump';
```

查看目录是否已经创建成功：

```sql
SQL> select * from dba_directories;
```

5、用sys管理员给指定用户赋予在该目录的操作权限

```plsql
SQL> grant read,write on directory dp_name to 用户;
```



## 常用语句

- expdp导出

```
1)导出表
expdp  tables=dbmon.lihaibo_exp dumpfile=sms.dmp DIRECTORY=dump_dir;
2)并发导出parallel，指定job名
我们需要特别注意一点，parallel 一定要与 dumpfile=...%U.dmp结合 使用，或者有多个表需要同时导出。单表，或者其他诸如network_link方式，指定parallel,也无法开启并发进程
expdp scott/tiger@orcl directory=dpdata1 dumpfile=scott3%U.dmp parallel=4 job_name=scott3
3)全表
expdp scott/tiger@orcl TABLES=emp,dept dumpfile=expdp.dmp DIRECTORY=dpdata1;
4)导出表，并指定表中的内容
expdp scott/tiger@orcl directory=dpdata1 dumpfile=expdp.dmp Tables=emp query="WHERE deptno=20";
5)导出表空间
expdp system/manager DIRECTORY=dpdata1 DUMPFILE=tablespace.dmp TABLESPACES=temp,example;
6)导出全库
expdp system/manager DIRECTORY=dpdata1 DUMPFILE=full.dmp FULL=y;
```

- impdp导入

```
1) 全用户导入
impdp scott/tiger DIRECTORY=dpdata1 DUMPFILE=expdp.dmp SCHEMAS=scott;
2) 用户对象迁移
impdp system/manager DIRECTORY=dump_dir DUMPFILE=expdp.dmp TABLES=scott.dept REMAP_SCHEMA=scott:system; （SCOTT为原用户，system为目标用户）
3) 导入指定表空间
impdp system/manager DIRECTORY=dump_dir DUMPFILE=tablespace.dmp TABLESPACES=example;
4) 全库导入
impdb system/manager DIRECTORY=dump_dir DUMPFILE=full.dmp FULL=y;
5) 表已存在的处理
impdp system/manager DIRECTORY=dump_dir DUMPFILE=expdp.dmp SCHEMAS=system TABLE_EXISTS_ACTION=append;
6) 表空间迁移
impdp system/manager directory=dump_dir dumpfile=remap_tablespace.dmp logfile=remap_tablespace.log remap_tablespace=A:B (A为原表空间名，B为指定的目标表空间名)
```

## expdp/impdp参数说明

- 帮助：
> ```
> impdp -help
> expdp help=y
> ```

- ATTACH

> ```
> 作用
>     当我们使用ctrl+C 退出交互式命令时，可使用attach参数重新进入到交互模式
> 语法
>     ATTACH=[schema_name.]job_name
>     Schema_name用户名,job_name任务名
> 示例
>     Expdp scott/tiger ATTACH=scott.export_job
> ```

- CONTENT

> ```
> 作用
>     限制了导出的内容，包括三个级别：全部／数据／元数据（结构）
> 语法
>    CONTENT={ALL | DATA_ONLY | METADATA_ONLY}
>    ALL           -- 导出所有数据，包括元数据及数据
>    DATA_ONLY     -- 只导出数据
>    METADATA_ONLY -- 只包含元数据，就是创建语句
> 示例
>    Expdp scott/tiger DIRECTORY=dump DUMPFILE=a.dump CONTENT=METADATA_ONLY
> ```

- **DIRECTORY**

> ```
> 作用
> 此路径可以理解为实际绝对路径在oracle数据库里的别名，是导出文件的存储位置
>     路径的创建： create directory &DIRECTORY_NAME AS '&PATH';
>     查看已存在路径： select  * from dba_directories;
> 语法
>     directory=[directory_name]
> 示例
>     Expdp scott/tiger DIRECTORY=dump_dir DUMPFILE=lhb.dump
> ```

- **DUMPFILE**

> ```
> 作用
>     此参数用户命名导出文件，默认是 expdat.dmp. 文件的存储位置如果在文件名前没有指定directory,则会默认存储到directory参数指定的路径下。
> 语法
>     DUMPFILE=[dump_dir:]file_name
> 示例
>     Expdp scott/tiger DIRECTORY=dump_dir DUMPFILE=dump_dir1:a.dmp
> ```

- ESTIMATE

> ```
> 在使用Expdp进行导出时，Expdp需要计算导出数据大小容量，Oracle可以通过两种方式进行容量估算，一种是通过数据块(blocks)数量、一种是通过统计信息中记录的内容(statistics)估算.
> 
> 语法结构：
>     EXTIMATE={BLOCKS | STATISTICS}
> 示例：
>     Expdp scott/tiger TABLES=emp ESTIMATE=STATISTICS DIRECTORY=dump_dir DUMPFILE=halberd.dump
>     Expdp scott/tiger TABLES=emp ESTIMATE=BLOCKS DIRECTORY=dump_dir DUMPFILE=halberd.dump
> ```

- EXTIMATE_ONLY

> ```
> 作用
>     此参数用于统计导出的数据量大小及统计过程耗时长短。
> 语法
>     EXTIMATE_ONLY={Y | N}
> 示例
>     Expdp scott/tiger ESTIMATE_ONLY=y NOLOGFILE=y directory=dump_dir schemas=halberd
> ```

- EXCLUDE

> ```
> 作用
>     此参数用于排除不需要导出的内容，如我们进行全库导出，但是不需要导出用户scott，此时需要在exlude后先指定排除类型为schema,再指定具体的schema。具体使用方法见include参数. EXCLUDE与include的使用方法是一样的
> 语法
>     EXCLUDE=object_type[:name_clause] [,object_type[:name_clause] ]
>     name_clause
>         "='object_name'"
>         "in ('object_name'[,'object_name',....])"
>         "in (select_clause) "
>     Object_type对象类型，如：table,view,procedure,schema等
>     name_clause指定名称的语句,如果不具体指定是哪个对象，则此类所有对象都不导出, select 语句中表名不要加用户名。用户名，通过schemas 指定。
> 
> 示例
>     expdp scott/tiger DIRECTORY=dump_dir DUMPFILE=halberd.dup EXCLUDE=VIEW
>     expdp scott/tiger DIRECTORY=dump_dir DUMPFILE=halberd.dup EXCLUDE=TABLE:\" IN\(\'TEMP\',\'GRADE\'\)\"
>     EXCLUDE=TABLE:"='APPLICATION_AUDIT'"
> ```

- FILESIZE

> ```
> 作用
>     用于指定单个导出的数据文件的最大值，与％U一起使用。比如，我们需要导出100G的数据，文件全部存储到一个文件内，在文件传输时，会耗费大量的时间，此时我们就可以使用这个参数，限制每个文件的大小，在传输导出文件时，就可以多个文件同时传送，大大的节省了文件传输时间。提高了工作的效率。
> 语法
>   FILESIZE=integer[B | K | M | G]
> 示例
>    Expdp scott/tiger DIRECTORY=dump_dir DUMPFILE=halberd%U.dup FILESIZE=20g
> ```

- FLASHBACK_SCN／FLASHBACK_TIME

> ```
> 作用
>     基于undo 及scn号(时间点)进行的数据导出。使用此参数设置会进行flashback query的功能，查询到对应指定的SCN时的数据，然后进行导出。只要UNDO不被覆盖，无论数据库是否重启，都可以进行导出. flashback_time参数与flashback_scn的原理是一样的。在导出的数据里保持数据的一致性是很有必要的。这个。。我想，没谁傻忽忽的把这两个参数一起使用吧？所以我就不提醒你两个参数不可以同时使用了。
> 语法
>    FLASHBACK_SCN=scn_value
>    FLASHBACK_TIME 有多种设定值的格式：
>    flashback_time=to_timestamp (localtimestamp)
>    flashback_time=to_timestamp_tz (systimestamp)
>    flashback_time="TO_TIMESTAMP (""25-08-2003 14:35:00""， ""DD-MM-YYYY HH24:MI:SS"")"  使用此格式可能会遇到ORA-39150错误。
> 示例
>    Expdp scott/tiger DIRECTORY=dump_dir DUMPFILE=halberd.dmp FLASHBACK_SCN= 12345567789
>    Expdp scott/tiger DIRECTORY=dump_dir DUMPFILE=halberd.dmp FLASHBACK_TIME= to_timestamp (localtimestamp)
> ```

- FULL

> ```
> 作用
>    指定导出内容为全库导出。这里需要特别注意的是,expdp 不能导出sys用户对象。即使是全库导出也不包含sys用户。
> 语法
>    FULL={Y | N}
> 示例
>    expdp \'\/ as sysdba\' directory=dump_dir full=y
> ```

- **<span id = "include">INCLUDE</span>**

> ```
> 作用
>     限制范围，指定自己想要的内容，比如要导出某个用户的某张表。
> 语法
>     INCLUDE = object_type[:name_clause],object_type[:name_clause]
> 示例
>     impdp dbmon/dbmon_123 directory=dump_dir network_link=zjzwb2 SCHEMAS=AICBS remap_schema=aicbs:aicbsb include=table:\"IN\(SELECT TABLE_NAME FROM dbmon.TABLES_TOBE_MASKED\)\"  LOGFILE=zjzwb.log transform=segment_attributes:n
>     PARFILE中设置:
>         INCLUDE=table:"in(select table_name from dba_tables where owner='AA')"
>         INCLUDE=TABLE:"IN('TEST1','TEST2')"
>     SHELL环境设置:
>         INCLUDE=TABLE:\"IN\(SELECT TABLE_NAME FROM DBA_TABLES WHERE OWNER=\'AA\'\)\"
>         INCLUDE=TABLE:\"IN\(\'TEST1\',\'TEST2\'\)\"
> 说明
>     当导入命令在目标端发起时，select 子句所涉及的表要在源端，并且dblink 所使用的用户有访问的权限。
> ```

- JOB_NAME

> ```
> 作用
>     指定任务名，如果不指定的话，系统会默认自动命名：SYS_EXPORT_mode_nn
> 语法
>     JOB_NAME=&JOB_NAME
> 其他
>     查看有哪些expdp/impdp job，可以通过dba_datapump_jobs查看，其实你通过v$session.action也可以查看到
>     大多与attach参数一起使用，重新进行expdp交互命令时使用。
> ```

- **LOGFILE**

> ```
> 作用： 指定导出日志名称。默认是：expdp.log
> 语法
>     LOGFILE=[DIRECTORY:]file_name   , 如果参数值里没有指定路径，会默认使用directory参数值所指向的路径。
>     directory ： 存储路径,
>     file_name :日志文件名
> 示例
>     expdp scott/tiger DIRECTORY=dump_dir DUMPFILE=halberd.dmp logfile=halberd.log
>     impdp scott/tiger DIRECTORY=dump_dir DUMPFILE=halberd.dmp logfile=halberd.log
> ```

- **NETWORK_LINK**

> ```
> 作用
>     此参数只有在导入（impdp）时使用，可通过本地数据库里的db_link连接到其他数据库A，将数据库A的数据直接导入到本地数据库。中间可节省导出数据文件，传送数据文件的过程。很方便。但是要特别注意，不同版本之间可能会存在问题，比如源库为10g,目标库为11g。使用network_link参数会报错。至于 12C 与低版本之间是否有问题尚未尝试。
> 语法
>     network_link=[db_link]
> 示例
>     impdp scott/tiger DIRECTORY=dump_dir DUMPFILE=halberd.dmp NETWORK_LINK=to_tjj SCHEMAS=halberd logfile=halberd.log
> ```

- NOLOGFILE

> ```
> 作用
>     不写导入导出日志，这个笔者是灰常灰常滴不建议设置为“Y”滴。
> 语法
>     nologfile=[y|n]
> ```

- PARALLEL

> ```
> 作用
>     指定导出／导入时使用多少个并发，默认是1,如果是单个表则在dumpfile中使用...%U.dmp
> 语法
>     parallel=[digit]
> 示例
>     expdp \'\/ as sysdba\' directory=dump_dir schemas=halberd dumpfile=halberd%U.dmp parallel=8 logfile=halberd.log
> ```

- PARFILE

> ```
> 作用
>     参数文件，这个参数文件里，存储着一些参数的设置。比如上面说过的，parallel,network_link,等。导出时，可以使用此参数，expdp/impdp会自动读取文件中的参数设置，进行操作。
> 语法
>     PARFILE=[directory_path] file_name
> 示例
>    expdp \'\/ as sysdba\' parfile=halberd.par
>    cat halberd.par
>    directory=dump_dir                          
>    logfile=test.log                            
>    schemas=test                                
>    query="where create_date > last_day(add_months(sysdate,-1)) and create_date <= last_day(sysdate)" 
>    transform=segment_attributes:n                 
>    network_link=to_aibcrm
>    table_exists_action=append                    
>    impdp \'\/ as sysdba\' parfile=test.par
> ```

- **QUERY**

> ```plsql
> 作用
>     此参数指定在导入导出时的限制条件，和SQL语句中的 "where" 语句是一样儿一样儿滴
> 语法
>     QUERY=([schema.] [table_name:] query_clause, [schema.] [table_name:] query_clause,……)
>     CONTENT=METADATA_ONLY, EXTIMATE_ONLY＝Y,TRANSPORT_TABLESPACES.
> 示例
>    Expdp scott/tiger directory=dump dumpfiel=a.dmp Tables=emp query=\"where car_buy_time between to_date('2018-01-01 0:00:00','yyyy-mm-dd HH24:mi:ss') and to_date('2019-12-31 23:59:59','yyyy-mm-dd HH24:mi:ss')\"
> ```

- **SCHEMAS**

> ```
> 作用
>     指定导出／导入哪个用户
> 语法
>     schemas=schema_name[,schemaname,....]
> 示例
>     expdp \'\/ as sysdba\' directory=dump_dir schemas=halberd
> ```

- REMAP_SCHEMA

> ```
>  只在导入时使用
> 作用
>     当把用户A的对象导入到用户（其实应该叫schema，将就看吧）B时，使用此参数，可实现要求
> 格式
>     remap_schema=schema1: schema2
> 示例
>     impdp \'\/ as sysdba\' directory=dump_dir dumpfile=halberd.dmp logfile=halberd.log remap_schema=scott:halberd
> ```

- **TABLES**

> ```
> 作用
>     指定导出哪些表。
> 格式
>     TABLES=[schema.]table_name[:partition_name][,[schema.]table_name[:partition_name]]
> 说明
>     Schema 表的所有者；table_name表名；partition_name分区名.可以同时导出不同用户的不同的表
> 示例
>     expdp \'\/ as sysdba\' directory=dump_dir tables=emp.emp_no,emp.dept
> ```

- **REMAP_TABLE**
> ```
> 作用
>     只有在导入时使用，用于将导入的数据表更名。
> 用法
>     REMAP_TABLE=S.a:b
> 说明
>    S:表所在模式；a: 数据所在的原表空间; b： 目标表空间
> 示例
>     impdp \'\/ as sysdba\' directory=dump_dir tables=emp,dept  remap_table=C##TEST.emp:newemp
> ```
- TABLESPACES

> ```
> 作用
>     指定导出／导入哪个表空间。
> 语法
>     tablespaces=tablespace_name[,tablespace_name,....]
> 示例
>     expdp \'\/ as sysdba\' directory=dump_dir tablespace=user
> ```

- REMAP_TABLESPACE

> ```
> 作用
>     只有在导入时使用，用于进行数据的表空间迁移。 把前一个表空间中的对象导入到冒号后面的表空间
> 用法
>     remap_tablespace=a:b
> 说明
>    a: 数据所在的原表空间; b： 目标表空间
> 示例
>    impdp \'\/ as sysdba\' directory=dump_dir tables=emp.dept remap_tablespace=user:user1
> ```

- TRANSPORT_FULL_CHECK

> ```
> 检查需要进行传输的表空间与其他不需要传输的表空间之间的信赖关系，默认为N。当设置为“Y”时，会对表空间之间的信赖关系进行检查，如A（索引表空间）信赖于B（表数据表空间），那么传输A而不传输B，则会出错，相反则不会报错。
> ```

- TRANSPORT_TABLESPACES

> ```
> 作用
>     列出需要进行数据传输的表空间
> 格式
>      TRANSPORT_TABLESPACES＝tablespace1[,tablespace2,.............]
> ```

- TRANSFORM

> ```
> 作用
>   此参数只在导入时使用，是一个用于设定存储相关的参数,有时候也是相当方便的。假如数据对应的表空间都存在的话，就根本用不到这个参数，但是，假如数据存储的表空间不存在，使用此参数导入到用户默认表空间就可以了。更灵活的，可以使用remap_tablespace参数来指定。
> 格式
>     transform=transform_name:value[bject_type]
>     transform_name = [OID | PCTSPACE | SEGMENT_ATTRIBUTES | STORAGE]:[Y|N]
>     segment attributes:段属性包括物理属性、存储属性、表空间和日志,Y 值按照导出时的存储属性导入，N时按照用户、表的默认属性导入
>     storage:默认为Y，只取对象的存储属性作为导入作业的一部分
>     oid:  owner_id,如果指定oid=Y(默认)，则在导入过程中将分配一个新的oid给对象表，这个参数我们基本不用管。
>     pctspace:通过提供一个正数作为该转换的值，可以增加对象的分配尺寸，并且数据文件尺寸等于pctspace的值(按百分比)
> 示例
>     transform=segment_attributes:n --表示将用户所有对象创建到用户默认表空间，而不再考虑原来的存储属性。
> ```

- **VERSION**

> ```
> 此参数主要在跨版本之间进行导数据时使用，更具体一点，是在从高版本数据库导入到低版本数据库时使用,从低版本导入到高版本，这个参数是不可用的。默认值是:compatible。此参数基本在导出时使用，导入时基本不可用。
> VERSION={COMPATIBLE | LATEST | version_string}
> COMPATIBLE       ： 以参数compatible的值为准，可以通过show parameter 查看compatible参数的值
> LATEST           ： 以数据库版本为准
> version_string   ： 指定版本。如： version=10.2.0.1
> ```

- SAMPLE

> ```
> SAMPLE 抽样 给出导出表数据的百分比，参数值可以取.000001~100（不包括100）。不过导出过程不会和这里给出的百分比一样精确，是一个近似值。 
> 格式： SAMPLE=[[schema_name.]table_name:]sample_percent 
> 示例： SAMPLE="HR"."EMPLOYEES":50
> ```

- <span id="table_exists_action">table_exists_action</span>

> ```
> 此参数只在导入时使用。
> 作用：导入时，假如目标库中已存在对应的表，对于这种情况，提供三种不同的处理方式：append,truncate,skip,replace
> 格式： table_exists_action=［append | replace| skip |truncate］
> 说明： append :   追加数据到表中
>        truncate:  将目标库中的同名表的数据truncate掉。
>        skip ：      遇到同名表，则跳过，不进行处理，注意：使用此参数值时，与该表相关的所有操作都会skip掉。
>        replace:    导入过程中，遇到同名表，则替换到目标库的那张表（先drop,再创建）。
> 示例：  table_exists_action=replace
> ```

- SQLFILE

> ```
> 只在导入时使用！
> 作用： 使用此参数时，主要是将DMP文件中的metadata语句取出到一个单独的SQLfile中，而数据并不导入到数据库中
> 格式： sqlfile=&file_name.sql
> 示例： impdp \'\/ as sysdba\' directory=dump_dir dumpfile=halberd.dmp logfile=halberd.log sqlfile=halberd.sql
> ```

- legacy mode 

  在11g中，才有这种模式。这种模式里兼容了以前版本中的部分参数，如：consistent,reuse_dumpfiles等（其实我现在也就知道这两个参数，哈哈，以后再遇到再补充）

- consistent

> ```
> 这个是保持数据一致性的一个参数。在11g中使用时，如果设置 consistent=true,则会默认转换成 flashback_time参数，时间设置为命令开始执行的那个时间点。
> 格式： consistent=[true|false]
> ```

- reuse_dumpfiles

> ```
> 作用：重用导出的dmp文件 。假如第一次我们导失败了，虽然导出失败，但是dmp文件 还 是会生成的。在修改导出命令，第二次执行时，就可以 加上这个参数。
> 格式： reuse_dumpfile=[true|false]
> ```

- partition_options

> ```
> 1 NONE 不对分区做特殊处理。在系统上的分区表一样创建。
> 2 DEPARTITION 每个分区表和子分区表作为一个独立的表创建，名字使用表和分区（子分区）名字的组合。
> 3 MERGE 将所有分区合并到一个表 
> 注意：如果导出时使用了TRANSPORTABLE参数，这里就不能使用NONE和MERGE
> ```

## <span id="jiaohu">交互式命令</span>

1. 连接到对应的job: 

   ```sql
   impdp username/password attach=&job_name
   expdp username/password attach=&job_name
   --jobname可以在运行任务时指定或者使用sql：
   select * from DBA_DATAPUMP_JOBS;
   ```

2. 查看运行状态： status

3. 停止导入导出： kill_job(直接kill 掉进程，不自动退出交互模式)

4. 停止导入导出：stop_job（逐一停止job进程的运行，并退出交互模式）

5. 修改并发值： parallel

6. 退出交互模式： exit / exit_client(退出到日志模式，对job无影响)

## Small技巧

### 不生成文件直接导入目标数据库

- 创建public dblink；


```plsql
CREATE PUBLIC DATABASE LINK <pub_link_test1>
CONNECT TO <username> IDENTIFIED BY <"密码">
USING
'(DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = <localhost>)(PORT = 1521))
    )
    (CONNECT_DATA =
    (SERVER = DEDICATED)
      (SERVICE_NAME = ORCL)
    )
)';
```

- 对DIRECTORY对应目录有`` `` `READ` / `WRITE` / `UTL_FILE` 权限；


```plsql
CREATE DIRECTORY <dp_name> AS <'E:\dptest\test1'>;

GRANT READ,WRITE ON DIRECTORY <dp_name> TO <username>;

GRANT EXECUTE ON SYS.UTL_FILE TO <username>;
```

- 导入到数据库；


```plsql
impdp <username>/<password>@orcl directory=<dp_name> tables=<C##TEST.testcar要导入的表> parallel=10 network_link=<pub_link_test1> remap_table=<C##TEST.testcar:testfullcar改名> LOGFILE=impdp.log
```

个人测试分开导入导出共花费100min,直接导入数据库40min，速度直接提高2.5倍！

- 注意：

1. 在B服务器数据库创建到A服务器数据库的public db link；
2. 在system下创建Directory，并赋予其读写权限，同时赋予SYS.UTL_FILE的执行权限；
3. 执行脚本参数位置。

### 导入多张表

使用**[INCLUDE](#include)**参数

### 如果存在表将导入文件追加到文件尾/跳过/删除

使用**[table_exists_action](#table_exists_action)**参数：［append | replace| skip |truncate］

### 查看进度

在[交互式命令](#jiaohu)里面status可以查看导入百分比等详细信息，导出没有显示百分比；

### 错误处理

#### 1、ORA-39112

导出正常，导入数据时，只成功导入部分记录等数据，另外的部分提数ora 39112错误，经查是因为导出的用户数据中，有部分记录的表用的索引在另一表空间中，该空间还未创建，所以导致该失败。

解决方法：在导入时，添加参数：RANSFORM=segment_attributes:n ，配合table_exists_action=replace参数，重新导入即可。

RANSFORM=segment_attributes:n 在导入时，会将数据导入默认的表空间中。

 补充，造成该问题的可能原因：

```
1、在原来测试库中，目标schema和别的用户相互授权了，可是你导出的dmp中没有包含所有的用户，导入时对应用户没有创建。
2、再就是，表空间问题，测试库中的用户下的某个表的索引没有在他的默认表空间里，这样你要在目标端（这里就是生产环境），创建好对应的表空间，
就是说如果你在测试库把a用户的下的某个表的权限授给了b,那么你在把a用户用数据泵倒进生产库时，他会在生产库中检测有没有用户ｂ。也要做相同的操作。
```



#### 2、ORA-39346

导入过程中，遇到错误"  ORA-39346: data loss in character set conversion for object SCHEMA_EXPORT/PROCEDURE/PROCEDURE  "

oracle官方的描述如下：

- **Description:** data loss in character set conversion for object string
- **Cause:** Oracle Data Pump import converted a metadata object from the export database character set into the target database character set prior to processing the object. Some characters could not be converted to the target database character set and so the default replacement character was used.
- **Action:** No specific user action is required. This type of data loss can occur if the target database character set is not a superset of the export databases character set.

 翻译：不需要特定的用户操作。如果目标数据库字符集不是导出数据库字符集的超集，则可能发生此类数据丢失。

## 注意事项

- expdp和impdp是服务端的工具程序，他们只能在Oracle服务端使用，不能在客户端使用。
- impdp/expdp组合使用，不可以与exp/imp混用。
- 导数据之间的数据库字符集必须一致，否则可能出现乱码。
- 经过测试一次只能有一个job在运行

## 脚本（[转自](https://www.cnblogs.com/chinas/p/8300955.html#_label5)）

```sh
#!/bin/bash
#############################################################################
#脚本功能：
#脚本路径：
#使用方法：脚本名称 操作类型参数
#############################################################################
export NLS_LANG=american_america.AL32UTF8
export ORACLE_HOME=/u01/app/oracle/product/12.1.0/db_1
export ORACLE_SID=cyrtestdb
export PATH=$PATH:$ORACLE_HOME/bin:/usr/bin:/usr/local/bin:/bin:/usr/bin/X11:/usr/local/bin:.

v_date=`date +%Y%m%d`
v_logfile=`echo $(basename  $0) | awk -F "." '{print $1".log"}'`  #日志文件名称：脚本所在目录下，脚本名称.log

v_usr_1=""            #Oracle用户名、密码
v_pwd_1=""
v_usr_2=""
v_pwd_2=""            
v_db_instance=""    #数据库实例名

v_backup_dir=""        #Oracle备份目录(全局变量)
v_oradir_name=""    #
v_tmp_space1=""        #临时表空间、表空间、索引表空间
v_tmp_space2=""
v_space1=""
v_space2=""
v_idx_space1=""
v_idx_space2=""
v_max_size="5120m"    #表空间数据文件最大值
v_dblink=""            #dblink名称

#记录日志
record_log(){
    echo -e `date '+%Y-%m-%d %H:%M:%S'` $1 | tee -a ${v_logfile}
}

#用户数据导出
exp_usrdata(){
    v_exp_usr=$1
    v_exp_pwd=$2
    v_oradir_name=$3
    cd ${v_backup_dir}
    [[ -f ${v_exp_usr}"_"${v_date}".dmp" ]] && rm -rf ${v_exp_usr}"_"${v_date}".dmp"
    expdp ${v_exp_usr}/${v_exp_pwd} DIRECTORY=${v_oradir_name} DUMPFILE=${v_exp_usr}"_"${v_date}".dmp" SCHEMAS=${v_exp_usr} LOGFILE=${v_exp_usr}"_"${v_date}".log"
}

#在目标库上创建数据库备份目录
create_bankup_dir(){
    #创建操作系统物理路径
    [[ -d ${v_backup_dir} ]] && mkdir -p ${v_backup_dir}

    sqlplus -S / as sysdba >> ${v_logfile} <<EOF
    set heading off feedback off verify off
    spool tmp_space_flag.tmp
    grant read,write on directory '${v_oradir_name}' to '${v_usr_1}','${v_usr_2}';
    spool off
    exit;
EOF
    ##如果当前表空间不存在，则创建，否则退出当前函数
    if [[ `grep ${v_oradir_name} tmp_space_flag.tmp | wc -l` -eq 0 ]]; then
        record_log "创建备份目录"$1"开始"
        sqlplus -S / as sysdba >> ${v_logfile} <<EOF
        set heading off feedback off verify off
        grant read,write on directory '${v_oradir_name}' to '${v_usr_1}','${v_usr_2}';
        exit;
EOF
        record_log "创建备份目录"$1"结束"
    else
        record_log "创建备份目录"$1"已存在"
        return
    fi
    ##注意清理临时标志文件
    [[ -f ./tmp_space_flag.tmp ]] && rm -rf tmp_space_flag.tmp
}

#创建表空间
create_space(){
    v_space_name=$1
    #判断表空间类型（临时表空间或普通表空间）
    if [[ `grep TMP $1` -eq 1 ]]; then
        v_space_type="temporary"
    else
        v_space_type=""
    fi 
    sqlplus -S / as sysdba >> ${v_logfile} <<EOF
    set heading off feedback off verify off
    create '${v_space_type}' tablespace '${v_space_name}' tempfile '${v_backup_dir}/${v_space_name}.dbf' size 50m autoextend on next 128k maxsize '${v_max_size}' extent management local; 
    exit;
EOF
}

#判断表空间是否存在，若表空间不存在则创建
deal_spaces(){
    v_space_name=$1
    sqlplus -S / as sysdba <<EOF 
    set heading off feedback off verify off
    spool tmp_space_flag.tmp
    select tablespace_name from dba_tablespaces where tablespace_name='${v_space_name}';
    spool off
    exit;
EOF
    ##如果当前表空间不存在，则创建，否则退出当前函数
    if [[ `grep ${v_space_name} tmp_space_flag.tmp | wc -l` -eq 0 ]]; then
        record_log "创建表空间"$1"开始"
        create_space ${v_space_name}
        record_log "创建表空间"$1"结束"
    else
        record_log "表空间"$1"已存在"
        return
    fi
    ##注意清理临时标志文件
    [[ -f ./tmp_space_flag.tmp ]] && rm -rf tmp_space_flag.tmp

}

#在目标库上创建用户并赋权
create_usrs(){
    v_create_usr=$1        #参数1：用户名
    v_create_pwd=$2        #参数2：密码
    v_create_tmp_space=$3         #参数3：临时表空间名称
    v_create_space=$4        #参数4：表空间名称

    sqlplus -S / as sysdba >> ${v_logfile} <<EOF
    set heading off feedback off verify off
    create user '${v_create_usr}' identified by '${v_create_pwd}' default tablespace '${v_create_space}' temporary tablespace '${v_create_tmp_space}';
    grant connect,resource to '${v_create_usr}';
    grant exp_full_database to '${v_create_usr}';
    grant imp_full_database to '${v_create_usr}';
    grant unlimited tablespace to '${v_create_usr}';
    exit;
EOF
}

#用户数据导入
imp_usrdata(){
    v_imp_usr=$1
    v_imp_pwd=$2
    v_oradir_name=$3
    impdp ${v_imp_usr}/${v_imp_pwd} DIRECTORY=${v_oradir_name} DUMPFILE=${v_imp_usr}"_"${v_date}".dmp" SCHEMAS=${v_imp_usr} LOGFILE=${v_imp_usr}"_"${v_date}".log" table_exists_action=replace RANSFORM=segment_attributes:n
}

#删除用户
drop_user(){
    v_drop_usr=$1
    #删除用户及用户下的所有数据，删除表空间及表空间下的所有数据
    sqlplus -S / as sysdba >> ${v_logfile} <<EOF
    set heading off feedback off verify off
    drop user '${v_drop_usr}' cascade;
    exit
EOF
}

#删除表空间并删除表空间下的数据文件
drop_tablespace(){
    v_drop_space=$1
    #删除用户及用户下的所有数据，删除表空间及表空间下的所有数据
    sqlplus -S / as sysdba >> ${v_logfile} <<EOF
    set heading off feedback off verify off
    drop tablespace '${v_drop_space}' including contents and datafiles;
    exit
EOF
    ##操作系统上表空间下的数据文件
    [[ -f ${v_backup_dir}/${v_drop_space}.dbf ]] && rm -rf ${v_backup_dir}/${v_drop_space}.dbf
}

#创建dblink
create_dblink(){
    v_clink_usr=$1
    v_clink_pwd=$2

    #以管理员身份对创建dblink赋权
    sqlplus -S / as sysdba >> ${v_logfile} <<EOF
    set heading off feedback off verify off
    grant create database link to '${v_clink_usr}';
    exit;
EOF
    #以普通用户登录创建dblink
    sqlplus -S ${v_clink_usr}/${v_clink_pwd}@${v_db_instance} >> ${v_logfile} <<EOF
    set heading off feedback off verify off
    CREATE DATABASE LINK ${v_dblink} CONNECT TO '${v_clink_usr}' IDENTIFIED BY '${v_clink_pwd}' USING '{v_db_instance}';
    exit
EOF
}

#判断dblink是否存在
deal_dblink(){
    v_link=$1
    v_link_usr=$2
    v_link_pwd=$3
    sqlplus -S / as sysdba <<EOF 
    set heading off feedback off verify off
    spool tmp_space_flag.tmp
    select object_name from dba_objects where object_name = '${v_link}'; 
    spool off
    exit;
EOF
    ##如果当前dblink不存在，则创建，否则退出当前函数
    if [[ `grep ${v_space_name} tmp_space_flag.tmp | wc -l` -eq 0 ]]; then
        record_log "创建"$1"开始"
        create_dblink ${v_link_usr} ${v_link_pwd}
        record_log "创建"$1"结束"
    else
        record_log $1"已存在"
        return
    fi
    ##注意清理临时标志文件
    [[ -f ./tmp_space_flag.tmp ]] && rm -rf tmp_space_flag.tmp
}

#主函数
main(){
    v_start=`date +%s`
    if [[ $1 -eq "exp" ]]; then
         record_log "bl库导出开始..."
        exp_usrdata ${v_usr_1} ${v_pwd_1} ${v_oradir_name}
        record_log "bl库导出结束..."

        record_log "hx库导出开始..."
        exp_usrdata ${v_usr_2} ${v_pwd_2} ${v_oradir_name}
        record_log "hx库导出结束..."
    elif [[ $1 -eq "pre" ]]; then
        #1、创建备份目录
        create_bankup_dir
        #2、创建表空间
        for v_sp in ${v_tmp_space1} ${v_tmp_space2} ${v_space1} ${v_space2} ${v_idx_space1} ${v_idx_space2}; do
            deal_spaces ${v_sp}
        done
        #3、创建用户、赋权
        record_log "创建用户开始..."
        create_usrs ${v_usr_1} ${v_pwd_1} ${v_tmp_space1} ${v_space1}
        create_usrs ${v_usr_2} ${v_pwd_2} ${v_tmp_space2} ${v_space2}
        record_log "创建用户结束..."
        #4、为hx库创建dblink
        record_log "创建dblink开始..."
        deal_dblink ${v_dblink} ${v_usr_2} ${v_pwd_2}
        record_log "创建dblink结束..."

    elif [[ $1 -eq "imp" ]]; then
        record_log "bl库导入开始..."
        imp_usrdata ${v_usr_1} ${v_pwd_1} ${v_oradir_name}
        record_log "bl库导入结束..."

        record_log "hx库导入开始..."
        imp_usrdata ${v_usr_2} ${v_pwd_2} ${v_oradir_name}
        record_log "hx库导入结束..."
    elif [[ $1 -eq "clean" ]]; then
        read -t 5 -p "确认清除 0-否 1-是" v_num
        if [[ ${v_num} -eq 1 ]]; then
            record_log "清理数据文件开始..."
            for m in ${} ${}; do
                drop_user ${m}
            done
            for n in ${v_tmp_space1} ${v_tmp_space2} ${v_space1} ${v_space2} ${v_idx_space1} ${v_idx_space2}; do
                drop_tablespace ${n}
            done
            record_log "清理数据文件结束..."
        else
            exit
        fi
    else
        echo "Usage: sh script [exp|clean|pre|imp]"
        exit
    fi
    v_end=`date +%s`
    v_use_time=$[ v_end - v_start ]
    record_log "本次脚本运行时间："${v_use_time}"秒"
}
main
```
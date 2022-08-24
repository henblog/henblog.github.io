## 流量主题

### 路径分析

**求每条会话在页面间跳转的次数**

**指用户在APP或网站中的访问路径。**

通常使用桑基图：提供每种页面跳转(由 `source跳转起始页/targer终到页` 表示)的次数。

- 建表(粒度为会话所在页面的浏览时间)：

```sql
create external table test_ljfx(
    session_id string comment '会话id',
    page_id string comment '页面id',
    view_time string comment '浏览时间'
)comment '路径分析测试表'
row format delimited fields terminated by '\t'
location '/test/ljfx/';
```

- 使用数据：

```
vim ljfx.txt

ssi100  page100	2022-08-23 11:12
ssi100  page101 2022-08-23 11:13
ssi100  page102 2022-08-23 11:15
ssi105  page100 2022-08-23 11:18
ssi101  page100 2022-08-23 11:18
ssi102  page100 2022-08-23 11:20
ssi102  page103 2022-08-23 11:21
ssi102  page102 2022-08-23 11:22
ssi102  page103 2022-08-23 11:23

hdfs dfs -mkdir -p /test/ljfx
hdfs dfs -put ljfx.txt /test/ljfx
```

- 查询：

```
SELECT 
    `source`,
    `target`,
    count(*) page_cnt
from (
    SELECT 
        concat(rn,'-',page_id) `source`,
        concat(rn+1,'-',nvl(next_page,'pageend')) `target`
    from(
        select 
            page_id,
            lead(page_id,1,NULL) over(partition by session_id order by view_time) next_page,
            ROW_NUMBER() over(partition by session_id order by view_time) rn
        from test_ljfx
    )t 
)t2
group by source,target;
```

- 查询存表：

```
CREATE EXTERNAL TABLE ljfx_app
(
    `dt`          STRING COMMENT '统计日期',
    `recent_days` BIGINT COMMENT '最近天数,1:最近1天,7:最近7天,30:最近30天',
    `source`      STRING COMMENT '跳转起始页面ID',
    `target`      STRING COMMENT '跳转终到页面ID',
    `path_count`  BIGINT COMMENT '跳转次数'
) COMMENT '页面浏览路径分析'
    ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t';
--插入数据
insert overwrite table ljfx_app
select * from ljfx_app
union
SELECT 
    '2022-08-23' dt,
    recent_day ,
    `source`,
    `target`,
    count(*) path_count
from (
    SELECT 
        recent_day,
        concat(rn,'-',page_id) `source`,
        concat(rn+1,'-',nvl(next_page,'end')) `target`
    from (
        SELECT
            recent_day,
            page_id,
            lead(page_id,1,NULL) over(partition by session_id,recent_day order by view_time) next_page,
            row_number() over(partition by session_id,recent_day) rn
        from test_ljfx 
            lateral view explode(array(1,3,7)) tmp as recent_day
        where view_time >= date_add('2022-08-23',-recent_day+1)
    )t
)t1
group by recent_day,`source`,`target`
;
```

## 用户主题

### 用户变动统计

**求今日的流失用户数和回流用户数；**

| **统计周期** | **指标**   | **说明**                 |
| ------------ | ---------- | ------------------------ |
| **最近1日**  | 流失用户数 | 最后登录的日期为七天前   |
| **最近1日**  | 回流用户数 | 昨日为流失用户但今日登录 |

- 建表(粒度为用户迄今为止的登录信息)：

  ```sql
  CREATE EXTERNAL TABLE test_yhbd
  (
      `user_id`         STRING COMMENT '用户id',
      `login_date_last` STRING COMMENT '末次登录日期',
      `login_count_td`  BIGINT COMMENT '累计登录次数'
  ) COMMENT '用户变动表'
  PARTITIONED BY (`dt` STRING)
  row format delimited fields terminated by '\t'
  location '/test/yhbd/';
  ```

- 使用数据：

  ```
  vim yhbd23.txt
  
  u1	2022-08-23	2
  u2	2022-08-15	5
  u3	2022-08-16	4
  u4	2022-08-17	1
  u5	2022-08-21	3
  u6	2022-08-23	8
  
  vim yhbd22.txt
  
  u1	2022-08-11	1
  u2	2022-08-15	5
  u3	2022-08-16	4
  u4	2022-08-17	1
  u5	2022-08-21	3
  u6	2022-08-22	7
  
  hdfs dfs -mkdir -p /test/yhbd/dt='2022-08-22'
  hdfs dfs -mkdir -p /test/yhbd/dt='2022-08-23'
  hdfs dfs -put yhbd22.txt /test/yhbd/dt='2022-08-22'
  hdfs dfs -put yhbd23.txt /test/yhbd/dt='2022-08-23'
  
  --刷新分区
  ALTER TABLE test_yhbd ADD IF NOT EXISTS PARTITION (dt='2022-08-22');
  ALTER TABLE test_yhbd ADD IF NOT EXISTS PARTITION (dt='2022-08-23');
  ```

- 查询：

  ```sql
  SELECT 
      churn_cnt,
      back_cnt
  FROM (
      SELECT 
          count(1) churn_cnt
      from test_yhbd 
      where dt = '2022-08-23' 
      	and login_date_last < date_add('2022-08-23',-6)
  )churn join (
      SELECT 
          count(1) back_cnt
      from (
          SELECT 
              user_id
          from test_yhbd 
          where dt = DATE_ADD('2022-08-23',-1) 
          	and login_date_last < date_add('2022-08-22',-6)
      )yestd_churn
      join (
          select 
              user_id
          from test_yhbd 
          where dt = '2022-08-23' 
          	and login_date_last = '2022-08-23'
      )today_login on yestd_churn.user_id= today_login.user_id
  )back;
  ```

### 用户留存率

**求新增用户(7天内注册)的当日留存(今日活跃)率**

1. **新增留存分析**是分析某天的新增用户中，有多少人有后续的活跃行为。
2. **活跃留存分析**是分析某天的活跃用户中，有多少人有后续的活跃行为。
3. **留存率**是指留存用户数和新用户数的比值。

- 建表(粒度为)：

  ```sql
  create external table test_yhzc(
      user_id string comment '用户id',
      first_login string comment '注册时间'
  )comment '用户登录表'
  row format delimited fields terminated by '\t'
  location '/test/yhzc/'
  ;
  
  create external table test_yhdl(
      user_id string comment '用户id',
      last_login string comment '最后登录时间'
  )comment '用户登录表'
  row format delimited fields terminated by '\t'
  location '/test/yhdl/'
  ;
  ```

- 使用数据：

  ```shell
  vim yhdl.txt
  u1	2022-08-23
  u2	2022-08-23
  u3	2022-08-23
  u4	2022-08-24
  u5	2022-08-24
  u6	2022-08-24
  u7	2022-08-24
  u8	2022-08-23
  u9	2022-08-24
  
  vim yhzc.txt
  u1	2022-08-16
  u3	2022-08-18
  u4	2022-08-18
  u6	2022-08-21
  u8	2022-08-23
  u9	2022-08-23
  
  hdfs dfs -mkdir -p /test/yhdl
  hdfs dfs -mkdir -p /test/yhzc
  hdfs dfs -put yhdl.txt /test/yhdl
  hdfs dfs -put yhzc.txt /test/yhzc
  ```

- 查询：

  ```
  
  ```







- 建表(粒度为)：

  ```
  
  ```

- 使用数据：

  ```
  
  ```

- 查询：

  ```
  
  ```
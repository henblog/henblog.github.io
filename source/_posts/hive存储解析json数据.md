## hive存储解析处理json数据

 hive 处理json数据总体来说有两个方向的路走

1. **在建表时指定行格式**：在导入之前将json拆成各个字段，导入Hive表的数据是已经解析过得。这将需要使用第三方的SerDe。
2. **解析json数据插入数据**：将json以字符串的方式整个入Hive表，然后通过使用UDF函数解析已经导入到hive中的数据，比如使用LATERAL VIEW json_tuple的方法，获取所需要的列名。

数据如下：

```txt
{name:"h1", age:"18"}
{name:"zs", age:"13"}
{name:"ls", age:"45"}
```

### 在建表时指定

**每行必须是一个完整的JSON，一个JSON不能跨越多行，serde不会对多行的Json有效，文件必须是可以拆分的；**

#### 1.下载JAR

- 使用之前先[下载jar](http://www.congiu.net/hive-json-serde/)：

```
http://www.congiu.net/hive-json-serde/
```

- 如果要想在Hive中使用JsonSerde，需要把jar添加到hive类路径中：

add jar json-serde-1.3.7-jar-with-dependencies.jar;

**报错可以将jar文件权限转换为rwxrwxr--** [还是报错](https://blog.csdn.net/xingyue0422/article/details/86479990)

#### 2.建表

```hi&#39;v
CREATE EXTERNAL TABLE test_json_serde
(
    `name` STRING COMMENT '姓名',
    `age` STRING COMMENT '年龄'
) COMMENT '测试导入json表'
    ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
    STORED AS TEXTFILE;
```

#### 2.嵌套结构

##### 源数据：

```
{"country":"Switzerland","languages":["German","French","Italian"],"religions":{"catholic":[6,7]}}
{"country":"China","languages":["chinese"],"religions":{"catholic":[10,20],"protestant":[40,50]}}
```

##### 建表：

```
CREATE TABLE test_json_serde (
    country string,
    languages array<string>,
    religions map<string,array<int>>)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
--LOCATION '/yourjsonpath'  --此处可指定json文件所在路径
STORED AS TEXTFILE;
 
LOAD DATA LOCAL INPATH '/yourjson.txt' OVERWRITE INTO TABLE  test_json_serde ;
```

##### 使用：

```
select languages[0],religions['catholic'][0]  from test_json_serde；
```

#### 3.坏数据

- 格式错误的数据的默认行为是抛出异常。在查询时报错。

- 忽略坏数据：

  ```
  ALTER TABLE json_table SET SERDEPROPERTIES ( "ignore.malformed.json" = "true");
  ```


- 忽略会将有问题的记录全部变为NULL
- 注：如果json格式正确，但是不符合Hive范式，则不会跳过，依然会报错。

#### 4.将标量转为数组

这是一个常见的问题，某一个字段有时是一个标量，有时是一个数组，例如：

```
{ field: "hello", .. }
{ field: [ "hello", "world" ], ...
```


在这种情况下，如果将表声明为array<string>，如果SerDe找到一个标量，它将返回一个单元素的数组，从而有效地将标量提升为数组。 但是标量必须是正确的类型。

#### 5.映射Hive关键词

有时可能发生的是，JSON数据具有名为hive中的保留字的属性。 例如，您可能有一个名为“timestamp”的JSON属性，它是hive中的保留字，当发出CREATE TABLE时，hive将失败。 此SerDe可以使用SerDe属性将hive列映射到名称不同的属性。

```
--源数据
{"country":"Switzerland","exec_date":"2017-03-14 23:12:21"}
{"country":"China","exec_date":"2017-03-16 03:22:18"}
--建表
CREATE TABLE tmp_json_mapping (
    country string,
    dt string
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
WITH SERDEPROPERTIES ("mapping.dt"="exec_date") --映射
STORED AS TEXTFILE;
--查询
hive> select * from tmp_json_mapping;
OK
Switzerland	2017-03-14 23:12:21
China	2017-03-16 03:22:18
Time taken: 0.081 seconds, Fetched: 2 row(s)
```


“mapping.dt”，表示dt列读取JSON属性为exec_date的值。

### 解析json后插入

**不能处理复杂类型（如果hive表中字段为array,map等）;**

 读取json的函数:

```
select get_json_object(t.json,'$.id'), get_json_object(t.json,'$.total_number') from tmp_json_test t ; 
 
select t2.* from tmp_json_test t1 lateral view json_tuple(t1.json, 'id', 'total_number') t2 as c1, c2;
 
 方法一使用函数get_json_object  ， 方法二使用函数 json_tuple
```


---
title: kettle使用(2)-使用sql脚本和Java脚本
date: 2022-07-10 09:25:00
author: hen
top: false
hide: false
cover: true
mathjax: false
summary: 本文提供了在kettle中使用sql脚本和Java脚本、定时和连接池
categories: kettle
tags:
  - kettle
  - ETL
---

# kettle使用(2)-使用SQL和JAVA脚本

## kettle中执行SQL脚本

- 在转换中需要勾选 **执行每一行   变量替换**；
- SQL中如果字段是字符类型的需要加引号 **'?'** ;
- SQL中的变量用 **?/${var}** (测试有问题)代替;
- **?**为变量时：在**作为参数的字段**中顺序填入变量(需要一一对应)，定义参数需要将文件作为输入，里面的字段值就是参数；
- ${var}为变量时：在**转换属性**的**命名参数**中设置变量；

## kettle中执行Java脚本

添加变量：

```java
Type var = "randtext";
get(Fields.Out,"var").setValue(r,var);
//并在底部'字段'中声明变量var对应类型Type
```

获取输入的字段：

```java
Type var = get(Fields.In, "a_fieldname").getString(r);//表中字段名为a_fieldname的字段
```

输出字段：

```java
get(Fields.Out, "output_fieldname").setValue(r, var);
```

## kettle中设置定时

在作业的**start**组件设置

## 连接池

会在目标数据库生成对应表文件，右上角connect连接就可以了。

报错：

```
You don't seem to be getting a connection to the server. Please check the path you're using and make sure the server is up and running. 
```

解决方法：1.是否缺失jar包；2.相关服务是否开启；3.如果没有对应的R_的表，查看对应用户在资源库是否又建表权限，正确赋权后，删掉原来的数据库连接，重新配置repository即可；4.新建数据库重新配置
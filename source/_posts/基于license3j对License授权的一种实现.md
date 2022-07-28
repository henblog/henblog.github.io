---
title: 基于license3j实现简单的License验证
date: 2022-07-28 17:21:00
author: hen
top: false
hide: false
cover: false
mathjax: false
summary: 本文是基于License3j进行开发，实现了简单的License验证
categories: license
tags:
  - license
  - 授权
---

# 基于license3j实现简单的License验证

## 介绍

所谓license，百度百科译为`执照，许可证` 。也就是说，当我们需要对软件进行授权管理，但不想投入太多，只是希望借用License机制提供基本的限制或提醒功能。如果需要强悍而又稳定的`许可证机制`还是需要去找专业团队。

本文是基于License3j进行开发，实现了简单的License验证。[License3j是一个免费开源]([verhas/License3j: Free Licence Management Library (github.com)](https://github.com/verhas/License3j)) 的Java 库，你可以借助它在项目中实现基本的License验证功能。

**许可证管理本身并不能保证程序不会被盗，盗版或以任何非法方式使用。但是，许可证管理可能会增加非法使用该程序的难度，因此可能会促使用户成为客户。许可证管理还有另一个影响，即法律方面。如果有足够的许可证管理，非法用户成功声称其使用是基于缺乏许可条件或虚假了解的可能性较小。**

## 业务场景

我们需要将自己的软件装到用户的电脑中，但又不想让用户一直`白嫖`，所以需要一个权限机制来限制用户，要实现的功能包括以下几点：

1. 软件有使用天数，并且会提示什么时候到期；
2. 软件需要和硬件绑定，不能随意拷贝；
3. 软件在私网下使用，在软件入口处限制出入；

## 实现思路

为实现以上目标，得出以下流程图：

![1658997375608](../pic/基于license3j对License授权的一种实现/1658997375608.png)

1. 用户使用软件将自己的信息封装（可以包括但不限于`用户名`、`本机ID`）成文件，发送到开发商，下面实例中将`过期时间`默认为已经过期的时间以增加文件鲁棒性；
2. 开发商得到用户的信息后，将`过期时间`等信息添加后，生成文件发送给用户，用户得到文件后软件就可以使用了；

上面的过程相当于用户将`可以识别为他本人的id`发送给开发商，开发商使用**私钥**对用户的id进行加密，软件使用**公钥**识别以获得信息。

## 代码

- 在项目中导入依赖(注意是jdk11)

```xml
<dependency>
    <groupId>com.javax0.license3j</groupId>
    <artifactId>license3j</artifactId>
    <version>3.2.0</version>
</dependency>
```

- 用户端生成LICENSE文件

```java
import javax0.license3j.Feature;
import javax0.license3j.HardwareBinder;
import javax0.license3j.License;
import javax0.license3j.io.LicenseWriter;

import java.io.File;
import java.io.IOException;
import java.security.NoSuchAlgorithmException;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.UUID;

public class LicenseFactory {
    public static void main(String[] args) {
        // licence 文件验证
        File licensefile = new File("D:\\09_test\\license\\LICENSE");
        try {
            License license = new License();

            if (!licensefile.exists()) {
                //获取本机硬件id
                UUID machineId = new HardwareBinder().getMachineId();
                //新建特征
                Feature f1 = Feature.Create.stringFeature("company", "tl");
                Feature f2 = Feature.Create.intFeature("version", 0);
                Feature f3 = Feature.Create.stringFeature("user", "hen");
                //指定uuid
                license.setLicenseId(machineId);
                //增加特征
                license.add(f1);
                license.add(f2);
                license.add(f3);
                license.setExpiry(new SimpleDateFormat("yyyy-MM-dd HH:ss:mm.SSS").parse("1980-01-01 00:00:00.00"));
                System.out.println(license.toString());
                //license文件生成
                new LicenseWriter("D:\\09_test\\license\\LICENSE").write(license);
                System.out.println("license build succ!");
            }
        } catch (NoSuchAlgorithmException | IOException e) {
            e.printStackTrace();
        } catch (ParseException e) {
            e.printStackTrace();
        }
    }
}
```

- 开发商端对用户的LICENSE加工

```java
import javax0.license3j.Feature;
import javax0.license3j.License;
import javax0.license3j.io.LicenseReader;
import javax0.license3j.io.LicenseWriter;

import java.io.File;
import java.io.IOException;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;

public class LicenseParseFactory {
    public static void main(String[] args) {
        String pathName = "D:\\09_test\\license\\LICENSE";
        // licence 文件验证
        File licensefile = new File(pathName);
        try {
            if (!licensefile.exists()) {
                System.out.println("license build unsucc! please check your licensefile");
            } else {
                //读取license文件获得信息
                License ls = new LicenseReader(pathName).read();
                //定义结束时间
                String expiry = "2022-08-21 20:00:00.00";
                SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:ss:mm.SSS");
                Date end = df.parse(expiry);
                ls.setExpiry(end);

                Feature f1 = Feature.Create.dateFeature("endtime", end);
                Feature f2 = Feature.Create.intFeature("version", 1);

                ls.add(f1);
                ls.add(f2);
                System.out.println("ls.toString() = " + ls.toString());

                new LicenseWriter(pathName).write(ls);
                System.out.println("license.bin build succ!");
            }
        } catch (IOException | ParseException e) {
            e.printStackTrace();
        }
    }
}
```

- 用户将LICENSE添加到目录后读取信息

```java
import javax0.license3j.HardwareBinder;
import javax0.license3j.License;
import javax0.license3j.io.LicenseReader;

import java.io.File;
import java.io.IOException;
import java.security.NoSuchAlgorithmException;
import java.util.Date;

public class LicenseController {
    //LICENSE文件路径
    String pathName = "D:\\09_test\\license\\LICENSE";
    //license是否可用(是否过期
    private boolean licenseAble = false;
    //记录到期时间
    private int expire = -1;
    //是否为本机
    private Boolean isMachine = false;

    //判断是否可用
    public Boolean getLicenseAble() {
        // licence 文件验证
        File licensefile = new File(pathName);
        try {
            if (!licensefile.exists()) {
                //license文件丢失
                licenseAble = false;
            } else {
                //读取license文件获得信息
                License ls = new LicenseReader(pathName).read();
                licenseAble = !ls.isExpired();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return licenseAble;
    }

    //返回距离到期还有多少天
    public int getExpire() {
        if (this.getLicenseAble()) {
            try {
                License ls = new LicenseReader(pathName).read();
                Date end = ls.get("endtime").getDate();
                //计算当前时间与文件中的时间差
                Date start = new Date(System.currentTimeMillis());
                expire = (int) ((end.getTime() - start.getTime()) / (24 * 60 * 60 * 1000));
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return expire;
    }

    //是否是本机
    public Boolean getIsMachine() {
        if (this.getLicenseAble()) {
            try {
                //获取本机硬件id
                String machineId = new HardwareBinder().getMachineIdString();
                License ls = new LicenseReader(pathName).read();
                //判断本机硬件id和文件中id是否一致
                String lsMachineId = ls.getLicenseId().toString();
                isMachine = lsMachineId.equals(machineId);
            } catch (IOException e) {
                e.printStackTrace();
            } catch (NoSuchAlgorithmException e) {
                e.printStackTrace();
            }
        }
        return isMachine;
    }
}
```

-   在用户登录或软件开始处使用方法判断

```java
LicenseController lc = new LicenseController();
        System.out.println("lc.getIsMachine() = " + lc.getIsMachine());//判断mac地址有没有改变
        System.out.println("lc.getExpire() = " + lc.getExpire());//输出过期时间还有多久
        System.out.println("lc.getLicenseAble() = " + lc.getLicenseAble());//判断有没有过期
```

代码中很多常量以及用户输入的数据没有封装，可以读取配置文件然后获取信息，感兴趣的可以尝试一下。
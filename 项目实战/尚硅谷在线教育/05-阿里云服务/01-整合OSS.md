# **一、****对象存储OSS**

为了解决海量数据存储与弹性扩容，项目中我们采用云存储的解决方案- 阿里云OSS。 

## 1、开通“对象存储OSS”服务

（1）申请阿里云账号

（2）实名认证

（3）开通“对象存储OSS”服务

（4）进入管理控制台



![image-20200714190455001](G:\soft\Typora\img\image-20200714190455001.png)

![image-20200714190535033](G:\soft\Typora\img\image-20200714190535033.png)

## 2、创建Bucket

选择：标准存储、公共读、不开通

![img](G:\soft\Typora\img\c4464d0a-5a54-4987-b5c6-8ccfe0535d42.png)

## 3、上传默认头像

创建文件夹avatar，上传默认的用户头像

![img](G:\soft\Typora\img\f5878678-ada3-4723-b606-7c6f6124db6f.png)

## 4、创建RAM子用户

![img](G:\soft\Typora\img\e64cdcc4-4c79-4a7c-a918-df3eb8197fb0.png)

# **二、使用SDK**

![img](G:\soft\Typora\img\933236c2-ad40-4a6b-a65d-64c34ce98cd8.png)

## **1、创建Mavaen项目**

aliyun-oss

![image-20200714190552232](G:\soft\Typora\img\image-20200714190552232.png)

![image-20200714190614518](G:\soft\Typora\img\image-20200714190614518.png)



## 2、pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>service</artifactId>
        <groupId>com.zxq</groupId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>service-oss</artifactId>

    <dependencies>
        <dependency>
            <groupId>com.aliyun.oss</groupId>
            <artifactId>aliyun-sdk-oss</artifactId>
             <version>2.8.3</version>
        </dependency>
        <!-- 日期工具栏依赖 -->
        <dependency>
            <groupId>joda-time</groupId>
            <artifactId>joda-time</artifactId>
        </dependency>
    </dependencies>


</project>
```

## 3、找到编码时需要用到的常量值

（1）endpoint

（2）bucketName

（3）accessKeyId

（4）accessKeySecret

### application.properties

```properties
#服务端口
server.port=8002
#服务名
spring.application.name=service-oss

#环境设置：dev、test、prod
spring.profiles.active=dev

#阿里云 OSS
#不同的服务器，地址不同
aliyun.oss.file.endpoint=oss-cn-chengdu.aliyuncs.com
aliyun.oss.file.keyid=LTAI4G61uNwvM3gr7SkxC16h
aliyun.oss.file.keysecret=K0t2ihKX4dePMhtwpPLW8nX1jze8Nt
#bucket可以在控制台创建，也可以使用java代码创建
aliyun.oss.file.bucketname=edu-guli-180
```



## 4、测试创建Bucket的连接

```java
package com.zxq.ossservice.utils;

import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

/**
 * @author zxq
 * 当项目已启动，spring接口，spring加载之后，执行接口一个方法
 */
@Component
public class ConstantPropertiesUtils implements InitializingBean {

    /**
     * 读去配置文件内容
     */
    @Value("${aliyun.oss.file.endpoint}")
    private String endpoint;

    @Value("${aliyun.oss.file.keyid}")
    private String keyId;

    @Value("${aliyun.oss.file.keysecret}")
    private String keySecret;

    @Value("${aliyun.oss.file.bucketname}")
    private String bucketName;

    /**
     * 定义公开静态常量
     * */
    public static String END_POINT;
    public static String ACCESS_KEY_ID;
    public static String ACCESS_KEY_SECRET;
    public static String BUCKET_NAME;

    @Override
    public void afterPropertiesSet() throws Exception {
        END_POINT = endpoint;
        ACCESS_KEY_ID = keyId;
        ACCESS_KEY_SECRET = keySecret;
        BUCKET_NAME = bucketName;
    }
}

```



```java
package com.zxq.ossservice.service.impl;

import com.aliyun.oss.OSS;
import com.aliyun.oss.OSSClientBuilder;
import com.zxq.ossservice.service.OssService;
import com.zxq.ossservice.utils.ConstantPropertiesUtils;
import org.joda.time.DateTime;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.multipart.MultipartFile;

import java.io.InputStream;
import java.util.UUID;

/**
 * @Author zxq
 * @Date 2020/6/16 21:35
 * @Version 1.0
 */
@Service
public class OssServiceImpl implements OssService {
    @Override
    public String uploadAvatar( MultipartFile file) {
        // 工具类获取值
        String endpoint = ConstantPropertiesUtils.END_POINT;
        String accessKeyId = ConstantPropertiesUtils.ACCESS_KEY_ID;
        String accessKeySecret = ConstantPropertiesUtils.ACCESS_KEY_SECRET;
        String bucketName = ConstantPropertiesUtils.BUCKET_NAME;

        try {
            // 创建OSS实例。
            OSS ossClient = new OSSClientBuilder().build(endpoint, accessKeyId, accessKeySecret);

            //获取上传文件输入流
            InputStream inputStream = file.getInputStream();
            //获取文件名称
            String fileName = file.getOriginalFilename();

            //1 在文件名称里面添加随机唯一的值
            String uuid = UUID.randomUUID().toString().replaceAll("-","");
            // yuy76t5rew01.jpg
            fileName = uuid+fileName;

            //2 把文件按照日期进行分类
            //获取当前日期
            //   2019/11/12
            String datePath = new DateTime().toString("yyyy/MM/dd");
            //拼接
            //  2019/11/12/ewtqr313401.jpg
            fileName = datePath+"/"+fileName;

            //调用oss方法实现上传
            //第一个参数  Bucket名称
            //第二个参数  上传到oss文件路径和文件名称   aa/bb/1.jpg
            //第三个参数  上传文件输入流
            ossClient.putObject(bucketName,fileName,inputStream);

            // 关闭OSSClient。
            ossClient.shutdown();

            //把上传之后文件路径返回
            //需要把上传到阿里云oss路径手动拼接出来
            //  https://edu-guli-1010.oss-cn-beijing.aliyuncs.com/01.jpg
            String url = "https://"+bucketName+"."+endpoint+"/"+fileName;
            return url;
        }catch(Exception e) {
            e.printStackTrace();
            return null;
        }
    }
}

```

# 三、实例

## controller

```java
package com.zxq.third.party.controller;

import com.zxq.commonutils.R;
import com.zxq.third.party.service.OssService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

/**
 * @Author zxq
 * @Date 2020/6/16 21:22
 * @Version 1.0
 */
@RestController
@RequestMapping("third/oss")
public class OssController {

    @Autowired
    private OssService ossService;

    @PostMapping("avatar")
    public R uploadAvatar(@RequestParam MultipartFile file){

        return R.ok(ossService.uploadAvatar(file));
    }
}

```

## impl

```java
package com.zxq.third.party.service.impl;

import com.aliyun.oss.OSS;
import com.aliyun.oss.OSSClientBuilder;
import com.zxq.third.party.service.OssService;
import com.zxq.third.party.utils.ConstantPropertiesUtils;
import org.joda.time.DateTime;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.InputStream;
import java.util.UUID;

/**
 * @Author zxq
 * @Date 2020/6/16 21:35
 * @Version 1.0
 */
@Service
public class OssServiceImpl implements OssService {
    @Override
    public String uploadAvatar( MultipartFile file) {
        // 工具类获取值
        String endpoint = ConstantPropertiesUtils.END_POINT;
        String accessKeyId = ConstantPropertiesUtils.ACCESS_KEY_ID;
        String accessKeySecret = ConstantPropertiesUtils.ACCESS_KEY_SECRET;
        String bucketName = ConstantPropertiesUtils.BUCKET_NAME;

        try {
            // 创建OSS实例。
            OSS ossClient = new OSSClientBuilder().build(endpoint, accessKeyId, accessKeySecret);

            //获取上传文件输入流
            InputStream inputStream = file.getInputStream();
            //获取文件名称
            String fileName = file.getOriginalFilename();

            //1 在文件名称里面添加随机唯一的值
            String uuid = UUID.randomUUID().toString().replaceAll("-","");
            // yuy76t5rew01.jpg
            fileName = uuid+fileName;

            //2 把文件按照日期进行分类
            //获取当前日期
            //   2019/11/12
            String datePath = new DateTime().toString("yyyy/MM/dd");
            //拼接
            //  2019/11/12/ewtqr313401.jpg
            fileName = datePath+"/"+fileName;

            //调用oss方法实现上传
            //第一个参数  Bucket名称
            //第二个参数  上传到oss文件路径和文件名称   aa/bb/1.jpg
            //第三个参数  上传文件输入流
            ossClient.putObject(bucketName,fileName,inputStream);

            // 关闭OSSClient。
            ossClient.shutdown();

            //把上传之后文件路径返回
            //需要把上传到阿里云oss路径手动拼接出来
            //  https://edu-guli-1010.oss-cn-beijing.aliyuncs.com/01.jpg
            String url = "https://"+bucketName+"."+endpoint+"/"+fileName;
            return url;
        }catch(Exception e) {
            e.printStackTrace();
            return null;
        }
    }
}

```

## aplication.properties

```properties
#服务端口
server.port=8002
#服务名
spring.application.name=service-third-party

# mysql数据库连接
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/guli_edu?serverTimezone=GMT%2B8
spring.datasource.username=root
spring.datasource.password=520135

#redis配置
spring.redis.host=123.56.76.154
spring.redis.port=6379
spring.redis.password=123456
spring.redis.database= 0
spring.redis.timeout=5000

spring.redis.lettuce.pool.max-active=20
spring.redis.lettuce.pool.max-wait=-1
#最大阻塞等待时间(负数表示没限制)
spring.redis.lettuce.pool.max-idle=5
spring.redis.lettuce.pool.min-idle=0

# nacos服务地址
spring.cloud.nacos.discovery.server-addr=123.56.76.154:8848

#环境设置：dev、test、prod
spring.profiles.active=dev

#阿里云 OSS
#不同的服务器，地址不同
aliyun.oss.file.endpoint=oss-cn-chengdu.aliyuncs.com
aliyun.oss.file.keyid=LTAI4G61uNwvM3gr7SkxC16h
aliyun.oss.file.keysecret=K0t2ihKX4dePMhtwpPLW8nX1jze8Nt
#bucket可以在控制台创建，也可以使用java代码创建
aliyun.oss.file.bucketname=edu-guli-180

#阿里云 vod
#不同的服务器，地址不同
aliyun.vod.file.keyid=LTAI4G61uNwvM3gr7SkxC16h
aliyun.vod.file.keysecret=K0t2ihKX4dePMhtwpPLW8nX1jze8Nt

# 最大上传单个文件大小：默认1M
spring.servlet.multipart.max-file-size=1024MB
# 最大置总上传的数据大小 ：默认10M
spring.servlet.multipart.max-request-size=1024MB

#请求处理的超时时间
ribbon.ReadTimeout: 120000
#请求连接的超时时间
ribbon.ConnectTimeout: 30000

#返回json的全局时间格式
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
spring.jackson.time-zone=GMT+8
#mybatis日志
mybatis-plus.configuration.log-impl=org.apache.ibatis.logging.stdout.StdOutImpl
```


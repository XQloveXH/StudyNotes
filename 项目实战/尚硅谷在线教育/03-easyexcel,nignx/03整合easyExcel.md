# 一、Excel导入导出的应用场景

1、数据导入：减轻录入工作量

2、数据导出：统计信息归档

3、数据传输：异构系统之间数据传输

# 二、EasyExcel简介

## 1、EasyExcel特点

- Java领域解析、生成Excel比较有名的框架有Apache poi、jxl等。但他们都存在一个严重的问题就是非常的耗内存。如果你的系统并发量不大的话可能还行，但是一旦并发上来后一定会OOM或者JVM频繁的full gc。
- EasyExcel是阿里巴巴开源的一个excel处理框架，**以使用简单、节省内存著称**。EasyExcel能大大减少占用内存的主要原因是在解析Excel时没有将文件数据一次性全部加载到内存中，而是从磁盘上一行行读取数据，逐个解析。
- EasyExcel采用一行一行的解析模式，并将一行的解析结果以观察者的模式通知处理（AnalysisEventListener）。

# 三、实现EasyExcel对Excel写操作

## 1、创建一个普通的maven项目

项目名：excel-easydemo

## 2、pom中引入xml相关依赖

```xml-dtd
<dependencies>
<!-- https: //mvnrepository . com/ artifact/ com. alibaba/easyexcel -->
    <dependency>
    <groupId>com.alibaba</ groupId>
    <artifactId>easyexcel< /artifactId>
    <version>2.1.1</version>
    </ dependency>
</dependencies>

```

## 3、创建实体类 

**设置表头和添加的数据字段**

```java
package com.zxq.eduservice.entity.subject;

import lombok.Data;

import java.util.ArrayList;
import java.util.List;

/**
 * @Author zxq
 * @Date 2020/6/17 16:45
 * @Version 1.0
 */
@Data
public class OneSubject {

    private String id;

    private String title;

    private List<TwoSubject> children = new ArrayList<>();
}

```

```java
package com.zxq.eduservice.entity.subject;

import lombok.Data;

/**
 * @Author zxq
 * @Date 2020/6/17 16:46
 * @Version 1.0
 */
@Data
public class TwoSubject {

    private String id;

    private String title;
}

```



```java
package com.zxq.eduservice.entity.excel;

import com.alibaba.excel.annotation.ExcelProperty;
import lombok.Data;

/**
 * @Author zxq
 * @Date 2020/6/17 14:27
 * @Version 1.0
 */
@Data
public class SubjectData {

    @ExcelProperty(index = 0)
    private String oneSubjectName;


    @ExcelProperty(index = 1)
    private String twoSubjectName;
}

```





## 4 、实现写操作

**（1）创建方法循环设置要添加到Excel的数据**

```java
//循环设置要添加的数据，最终封装到list集合中
private static List<EduSubject> data() {
    List<EduSubject> list = new ArrayList<EduSubject>();
    for(int i=0;i<10;i++){
    EduSubject data = new EduSubject( );
    data.setSno(i);
    data.setSname("张三"+i);
    list.add(data);
    }
}
return list;

```

**（2）实现最终的添加操作（写法一）**

```java

public static void main(String[] args) throws Exception {
//写法1
String fileName = "F:\\11.xlsx";
//这里需要指定写用哪个class去写，然后写到第一个sheet， 名字为模板然后文件流会自动关闭
//如果这里想使用03则传入excelType参数即可
EasyExcel.write(fileName，DemoData.class).sheet("写入方法- ").dowrite(data());
}
```

**（3）实现最终的添加操作（写法二）**

```java
public static void main(String[] args) throws Exception {
    //写法2，方法二需要手动关闭流
    string fileName = "F:\\112.xlsx";
    //这里需要指定写用哪个class去写
    ExcelWriter excelWriter = EasyExcel.write(fileName, DemoData.class).build();
    Writesheet writesheet = EasyExcel.writerSheet("写入方法二").build();
    excelwriter.write(data(),writeSheet);
    ///千万别忘记finish会帮忙关闭流
    excelWriter.finish();
}

```

# 四.实现EasyExcel对Excel读操作

1-3步与写操作一样

## 4.实现写操作

### (1)创建读取操作的监听器

```java
package com.zxq.eduservice.listener;

import com.alibaba.excel.context.AnalysisContext;
import com.alibaba.excel.event.AnalysisEventListener;
import com.baomidou.mybatisplus.core.conditions.query.QueryWrapper;
import com.zxq.eduservice.entity.EduSubject;
import com.zxq.eduservice.entity.SubjectData;
import com.zxq.eduservice.service.EduSubjectService;
import com.zxq.servicebase.exception.GuliException;

/**
 * @Author zxq
 * @Date 2020/6/17 14:32
 * @Version 1.0
 */
public class ExcelReadListener extends AnalysisEventListener<SubjectData> {

    //因为SubjectExcelListener不能交给spring进行管理，需要自己new，不能注入其他对象
    //不能实现数据库操作
    public EduSubjectService subjectService;
    public ExcelReadListener() {}
    public ExcelReadListener(EduSubjectService subjectService) {
        this.subjectService = subjectService;
    }

    //读取excel内容，一行一行进行读取
    @Override
    public void invoke(SubjectData subjectData, AnalysisContext analysisContext) {
        if(subjectData == null) {
            throw new GuliException(500,"文件数据为空");
        }

        //一行一行读取，每次读取有两个值，第一个值一级分类，第二个值二级分类
        //判断一级分类是否重复
        EduSubject existOneSubject = this.existOneSubject(subjectService, subjectData.getOneSubjectName());
        if(existOneSubject == null) { //没有相同一级分类，进行添加
            existOneSubject = new EduSubject();
            existOneSubject.setParentId("0");
            existOneSubject.setTitle(subjectData.getOneSubjectName());//一级分类名称
            subjectService.save(existOneSubject);
        }

        //获取一级分类id值
        String pid = existOneSubject.getId();

        //添加二级分类
        //判断二级分类是否重复
        EduSubject existTwoSubject = this.existTwoSubject(subjectService, subjectData.getTwoSubjectName(), pid);
        if(existTwoSubject == null) {
            existTwoSubject = new EduSubject();
            existTwoSubject.setParentId(pid);
            existTwoSubject.setTitle(subjectData.getTwoSubjectName());//二级分类名称
            subjectService.save(existTwoSubject);
        }
    }

    //判断一级分类不能重复添加
    private EduSubject existOneSubject(EduSubjectService subjectService,String name) {
        QueryWrapper<EduSubject> wrapper = new QueryWrapper<>();
        wrapper.eq("title",name);
        wrapper.eq("parent_id","0");
        EduSubject oneSubject = subjectService.getOne(wrapper);
        return oneSubject;
    }

    //判断二级分类不能重复添加
    private EduSubject existTwoSubject(EduSubjectService subjectService,String name,String pid) {
        QueryWrapper<EduSubject> wrapper = new QueryWrapper<>();
        wrapper.eq("title",name);
        wrapper.eq("parent_id",pid);
        EduSubject twoSubject = subjectService.getOne(wrapper);
        return twoSubject;
    }

    @Override
    public void doAfterAllAnalysed(AnalysisContext analysisContext) {

    }
}

```

### (2)调用实现最终的读取

```java
public static void main(String[] args) throws Exception {
//写法1:
string fileName = "F:\\01.xlsx";
//这里需要指定读用哪个class去读，然后读取第一个sheet 文件流会自动关闭
EasyExcel.read(fileName, ReadData.class, new Excellistener()).sheet().doRead();
//写法2:
InputStream in = new BufferedInputstream(new FileInputStream("F:\\01.xlsx"));
ExcelReader excelReader = EasyExcel.read(in, ReadData.class, new ExcelListener( )).build
Readsheet readsheet = EasyExcel.readsheet(0).build();
excelReader.read(readsheet);
//这里千万别忘记关闭，读的时候会创建临时文件，到时磁盘会崩的
excelReader. finish();

```


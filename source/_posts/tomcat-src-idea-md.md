---
title: tomcat_src_idea.md
date: 2021-03-26 11:38:08
tags:
---

### 在IDEA查看Tomcat代码,并编译成功

这里我只是想看代码,看他编译成功就好

```bash
 git clone git@github.com:apache/tomcat.git
 #这里就用8.5了
 git checkout 8.5.x
 cd tomcat
 mkdir catalina-home
 cp -rf conf webapps catalina-home
 touch pom.xml
```

在IDEA中导入tomcat目录, pom添加如下文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>Tomcat8.0</artifactId>
    <name>Tomcat7.0</name>
    <version>8.0</version>

    <build>
        <finalName>Tomcat8.0</finalName>
        <sourceDirectory>java</sourceDirectory>
        <testSourceDirectory>test</testSourceDirectory>
        <resources>
            <resource>
                <directory>java</directory>
            </resource>
        </resources>
        <testResources>
            <testResource>
                <directory>test</directory>
            </testResource>
        </testResources>
        <plugins>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <encoding>UTF-8</encoding>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.easymock</groupId>
            <artifactId>easymock</artifactId>
            <version>3.4</version>
        </dependency>
        <dependency>
            <groupId>org.apache.ant</groupId>
            <artifactId>ant</artifactId>
            <version>1.10.9</version>
        </dependency>
        <dependency>
            <groupId>wsdl4j</groupId>
            <artifactId>wsdl4j</artifactId>
            <version>1.6.2</version>
        </dependency>
        <dependency>
            <groupId>javax.xml</groupId>
            <artifactId>jaxrpc</artifactId>
            <version>1.1</version>
        </dependency>
        <dependency>
            <groupId>org.eclipse.jdt.core.compiler</groupId>
            <artifactId>ecj</artifactId>
            <version>4.5.1</version>
        </dependency>

    </dependencies>
</project>
```

然后就可以以maven项目导入idea了, idea也会主动识别, 不主动的话就右键pom.xml文件会有选项, 然后可以执行 `mvn compile` 成功变异了, 就可以愉快地在IDEA看tomcat源码了

Main class设置为org.apache.catalina.startup.Bootstrap

添加VM options 

-Dcatalina.home=catalina-home 

-Dcatalina.base=catalina-home 

-Djava.endorsed.dirs=catalina-home/endorsed 

-Djava.io.tmpdir=catalina-home/temp 

-Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager 

-Djava.util.logging.config.file=catalina-home/conf/logging.properties

说明：如果编译build的时候出现Test测试代码报错，注释该代码即可。本文中的Tomcat源码util.TestCookieFilter类会报错，将其注释即可。

四、启动项目
上面第三步已经构建了项目的运行环境，点击运行或者调试按钮后，正常运行。

项目启动完毕我们可以测试了，在浏览器访问http://localhost:8080/  发现打不开我们看到的经典欢迎页面了，页面报了一个错



原因是我们直接启动org.apache.catalina.startup.Bootstrap的时候没有加载org.apache.jasper.servlet.JasperInitializer，从而无法编译JSP。解决办法是在tomcat的源码org.apache.catalina.startup.ContextConfig中手动将JSP解析器初始化：

line 776: context.addServletContainerInitializer(new JasperInitializer(), null);
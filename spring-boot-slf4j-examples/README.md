# 本案例记录springboot使用slf4f和logback记录日志

## 1、slf4j的使用

slf4j为门面日志，logback为日志实现，不直接调用实现类，而是调用抽象层里的方法。

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class HelloWorld {
  public static void main(String[] args) {
    Logger logger = LoggerFactory.getLogger(HelloWorld.class);
    logger.info("Hello World");
  }
}
```

![img](https://www.slf4j.org/images/concrete-bindings.png)

## 2、日志配置文件

每个日志的实现框架都有自己的配置文件，使用slf4j以后，配置文件还是做成日志实现框架自己本身的配置文件。

## 3、springboot使用logback的配置

此处使用springboot来集成slf4j和logback，只需要在application.properties或application.yml文件中做如下配置即可

```properties
#logging.level.* : 作为package（包）的前缀来设置日志级别。
logging.level.com.sullylei = trace


#logging.path :配置日志的路径。如果没有配置logging.file,Spring Boot 将默认使用spring.log作为文件名。
#在当前磁盘的根路径下创建spring文件夹和log文件夹；使用spring.log作为默认文件
#logging.path=/spring/log
#logging.file :配置日志输出的文件名，也可以配置文件名的绝对路径。
logging.file=/spring/log/slf4j-test.log

#在控制台输出的日志格式
logging.pattern.console = %d{yyyy-MMM-dd HH:mm:ss.SSS} ==== [%thread] === %-5level === %logger{15} === %msg%n

#在文件中输出的日志格式
logging.pattern.file== %d{yyyy-MMM-dd HH:mm:ss.SSS} ### [%thread] ### %-5level ### %logger{15} ### %msg%n
#logging.pattern.level :定义渲染不同级别日志的格式。默认是%5p.

#通过命令行改变日志的输出级别
#java -jar my-app.jar --debug
```

##  4、指定配置

给类路径下放上每个日志框架自己的配置文件即可；SpringBoot就不使用他默认配置的了

| Logging System          | Customization                                                |
| ----------------------- | ------------------------------------------------------------ |
| Logback                 | `logback-spring.xml`, `logback-spring.groovy`, `logback.xml` or `logback.groovy` |
| Log4j2                  | `log4j2-spring.xml` or `log4j2.xml`                          |
| JDK (Java Util Logging) | `logging.properties`                                         |

logback.xml：直接就被日志框架识别了；

**logback-spring.xml**：日志框架就不直接加载日志的配置项，由SpringBoot解析日志配置，可以使用SpringBoot的高级Profile功能

```xml
<springProfile name="staging">
    <!-- configuration to be enabled when the "staging" profile is active -->
  	可以指定某段配置只在某个环境下生效
</springProfile>

```

如：

```xml
<appender name="stdout" class="ch.qos.logback.core.ConsoleAppender">
        <!--
        日志输出格式：
			%d表示日期时间，
			%thread表示线程名，
			%-5level：级别从左显示5个字符宽度
			%logger{50} 表示logger名字最长50个字符，否则按照句点分割。 
			%msg：日志消息，
			%n是换行符
        -->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <springProfile name="dev">
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} ----> [%thread] ---> %-5level %logger{50} - %msg%n</pattern>
            </springProfile>
            <springProfile name="!dev">
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} ==== [%thread] ==== %-5level %logger{50} - %msg%n</pattern>
            </springProfile>
        </layout>
    </appender>
```



如果使用logback.xml作为日志配置文件，还要使用profile功能，会有以下错误

 `no applicable action for [springProfile]`



## 5、原理分析

当在application.properties文件中不做配置时，使用springboot的默认配置，查看spring-boot-2.1.4.release.jar文件中的org.springframework.boot中的logging包，在该包下面分别有java、log4j2、logback文件夹，分别对应三种日志实现。

下面以logback为例对该文件夹的配置进行分析。

### 5.1 base.xml文件说明

该文件中有条注释为：base logback configuration provided for compatibility with Spring boot 1.1内容为：

```xml
<included>
   <include resource="org/springframework/boot/logging/logback/defaults.xml" />
   <property name="LOG_FILE" value="${LOG_FILE:-${LOG_PATH:-${LOG_TEMP:-${java.io.tmpdir:-/tmp}}}/spring.log}"/>
   <include resource="org/springframework/boot/logging/logback/console-appender.xml" />
   <include resource="org/springframework/boot/logging/logback/file-appender.xml" />
   <root level="INFO">
      <appender-ref ref="CONSOLE" />
      <appender-ref ref="FILE" />
   </root>
</included>
```

第一行default.xml中的包含一些默认配置转换规则，日志输出格式，日志登记

第二行日志文件的默认定义，包括路径和日志文件名

第三行是springboot初始执行时，控制台输出logback的默认配置

第四行是springboot初始执行时，file输出logback的默认配置

root几行表示root的日志输出级别

### 5.2 default.xml文件说明

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!--
Default logback configuration provided for import, equivalent to the programmatic
initialization performed by Boot
-->

<included>
   <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />
   <conversionRule conversionWord="wex" converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter" />
   <conversionRule conversionWord="wEx" converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter" />
   <property name="CONSOLE_LOG_PATTERN" value="${CONSOLE_LOG_PATTERN:-%clr(%d{${LOG_DATEFORMAT_PATTERN:-yyyy-MM-dd HH:mm:ss.SSS}}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>
   <property name="FILE_LOG_PATTERN" value="${FILE_LOG_PATTERN:-%d{${LOG_DATEFORMAT_PATTERN:-yyyy-MM-dd HH:mm:ss.SSS}} ${LOG_LEVEL_PATTERN:-%5p} ${PID:- } --- [%t] %-40.40logger{39} : %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>

   <logger name="org.apache.catalina.startup.DigesterFactory" level="ERROR"/>
   <logger name="org.apache.catalina.util.LifecycleBase" level="ERROR"/>
   <logger name="org.apache.coyote.http11.Http11NioProtocol" level="WARN"/>
   <logger name="org.apache.sshd.common.util.SecurityUtils" level="WARN"/>
   <logger name="org.apache.tomcat.util.net.NioSelectorPool" level="WARN"/>
   <logger name="org.eclipse.jetty.util.component.AbstractLifeCycle" level="ERROR"/>
   <logger name="org.hibernate.validator.internal.util.Version" level="WARN"/>
</included>
```

里面记录了转换规则，控制台日志输出的默认格式、文件日志输出的默认格式、相关模块的日志输出级别


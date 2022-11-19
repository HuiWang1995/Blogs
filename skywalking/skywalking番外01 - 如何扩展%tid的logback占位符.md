# skywalking番外01 - 如何扩展%tid的logback占位符

## 配置

```xml
    <appender name="FILE"  class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <FileNamePattern>./logs/ruoyi-vue-pro-%d{yyyy-MM-dd_HH}.log</FileNamePattern>
            <MaxHistory>3</MaxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout">
                <pattern>%d{ISO8601} | %tid | %thread | %-5level | %msg%n</pattern>
            </layout>
        </encoder>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <MaxFileSize>10MB</MaxFileSize>
        </triggeringPolicy>
    </appender>
```

这是skywalking使用%tid做占位符来展示TraceId的方式.那么究竟怎么实现的呢?

## Layout

观察配置很容易发现在/appender/encoder/layout中class属性指向的那个类是skywalking提供的工具包中的类.

```java
package org.apache.skywalking.apm.toolkit.log.logback.v1.x;

import ch.qos.logback.classic.PatternLayout;

public class TraceIdPatternLogbackLayout extends PatternLayout {
    public TraceIdPatternLogbackLayout() {
    }

    static {
        defaultConverterMap.put("tid", LogbackPatternConverter.class.getName());
    }
}
```

这个类很简单,看似什么都没做,就是继承了Logback的PatternLayout类,并在该类的static 变量defaultConverterMap放入了一个key为tid,一个value为Sting类型的className.

## Converter

```java
package org.apache.skywalking.apm.toolkit.log.logback.v1.x;

import ch.qos.logback.classic.pattern.ClassicConverter;
import ch.qos.logback.classic.spi.ILoggingEvent;

public class LogbackPatternConverter extends ClassicConverter {
    public LogbackPatternConverter() {
    }

    public String convert(ILoggingEvent iLoggingEvent) {
        return "TID: N/A";
    }
}
```

看起来这个Converter也很简单,就是继承了logback的ClassicConverter,并实现了converter方法,返回一个固定值!根本就不是动态的啊!

所以,运行过工程,且没有打探针包的朋友们会发现就是输出了TID: N/A.而有探针的情况下,会输出动态的TraceId.那么问题肯定在探针中了.

## 探针增强

skywalking中探针在做增强的时候,在源码中会有被增强类的全类名来匹配,所以找到了这样的代码:

```java
package org.apache.skywalking.apm.toolkit.activation.log.logback.v1.x;


public class LogbackPatternConverterActivation extends ClassInstanceMethodsEnhancePluginDefine {

    public static final String INTERCEPT_CLASS = "org.apache.skywalking.apm.toolkit.activation.log.logback.v1.x.PrintTraceIdInterceptor";
    public static final String ENHANCE_CLASS = "org.apache.skywalking.apm.toolkit.log.logback.v1.x.LogbackPatternConverter";
    public static final String ENHANCE_METHOD = "convert";

    /**
     * @return the target class, which needs active.
     */
    @Override
    protected ClassMatch enhanceClass() {
        return byName(ENHANCE_CLASS);
    }

    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return new InstanceMethodsInterceptPoint[] {
            new InstanceMethodsInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getMethodsMatcher() {
                    return named(ENHANCE_METHOD).and(takesArgumentWithType(0, "ch.qos.logback.classic.spi.ILoggingEvent"));
                }

                @Override
                public String getMethodsInterceptor() {
                    return INTERCEPT_CLASS;
                }
            }
        };
    }
}

```



删除掉多余的代码,可以看到,通过匹配ENHANCE_CLASS,以及ENHANCE_METHOD,然后返回一个增强过的类INTERCEPT_CLASS,即`org.apache.skywalking.apm.toolkit.activation.log.logback.v1.x.PrintTraceIdInterceptor`

```java
package org.apache.skywalking.apm.toolkit.activation.log.logback.v1.x;

public class PrintTraceIdInterceptor implements InstanceMethodsAroundInterceptor {

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
        Object ret) throws Throwable {
        if (!ContextManager.isActive()) {
            if (allArguments[0] instanceof EnhancedInstance) {
                String tid = (String) ((EnhancedInstance) allArguments[0]).getSkyWalkingDynamicField();
                if (tid != null) {
                    return "TID:" + tid;
                }
            }
        }
        return "TID:" + ContextManager.getGlobalTraceId();
    }
}
```

就是这个类去上下文管理器中去取来了全局流水号.

## 学以致用

那么对于Logback可能陌生的同学,看完这篇应该就大致知道怎么扩展一个logback的占位符了吧.

1. 继承ClassicConverter, 在convert方法中写入实际需要给占位符替换的内容.
2. 继承PatternLayout,在static的map中放入(占位符,转换类类名)映射对.
3. 配置文件中的layout指定class为该Layout的全类名.另外,Encoder节点的class也要使用`ch.qos.logback.core.encoder.LayoutWrappingEncoder`.

再说一种其他的方式, 如果用过spring cloud sleuth做全链路流水号的朋友会发现他们使用的占位符是这样的:`%X{X-B3-TraceId:-},%X{X-B3-SpanId:-}`.这是一种%X{key}的形式,使用了logback的MDC进行的扩展.可以参考这篇文章:[<logback扩展字段>](https://blog.csdn.net/weixin_38815505/article/details/102569619).

### 举例

那么究竟实际上有什么使用场景呢,举个例子: 我们不知道这个API是哪个用户登陆后进行的操作.

第一种解决方案: 拦截servlet的filter,从session中获取用户ID,打一条日志. 实际查询时根据全链路流水号来查询用户号.或者拦截spring mvc也是同理,不过是filter级别更高.

第二种解决方案:拦截servlet后,将其用户ID放入ThreadLocal中,利用本篇学到的扩展logback占位符的方式,输出到每一行日志中.

- 两种方式都行,各取所需.

## 其他日志框架的扩展

日志框架有很多,需要去官方看文档学习. 而我发现skywalking中基本有了我们常用的日志框架的扩展占位符的方式,是一个很好的例子.

在[skywalking/apm-snifer/apm-tookit-activation](https://gitee.com/OpenSkywalking/sky-walking/tree/master/apm-sniffer/apm-toolkit-activation)包下涵盖了对log4j-1.x/ log4j-2.x/ logback-1.x/ logging-common的扩展.(github墙高筑,这里的链接改成gitee中的链接了).
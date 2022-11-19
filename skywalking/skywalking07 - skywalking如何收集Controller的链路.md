# skywalking07 - skywalking如何收集Controller的链路

> 对于我们在java中常用的注解@Controller、@RestController，在运行时，将会产生链路，在“链路追踪”中可以进行查看，那么来看看怎么收集的

## Instrumentation 指明拦截的类

- 在`org.apache.skywalking.apm.plugin.spring.mvc.v5.define.AbstractControllerInstrumentation`抽象类的实现类`ControllerInstrumentation`、`RestControllerInstrumentation`中指定了`ENHANCE_ANNOTATION`分别为`org.springframework.stereotype.Controller`以及`org.springframework.web.bind.annotation.RestController`进行指定了增强的类。

```java
public class RestControllerInstrumentation extends AbstractControllerInstrumentation {

    public static final String ENHANCE_ANNOTATION = "org.springframework.web.bind.annotation.RestController";

    @Override
    protected String[] getEnhanceAnnotations() {
        return new String[] {ENHANCE_ANNOTATION};
    }
}
```

- 然后通过如下方法指定拦截增强的方法

```java
    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return new InstanceMethodsInterceptPoint[] {
            new DeclaredInstanceMethodsInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getMethodsMatcher() {
                    return byMethodInheritanceAnnotationMatcher(named("org.springframework.web.bind.annotation.RequestMapping"));
                }
            },
            new DeclaredInstanceMethodsInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getMethodsMatcher() {
                    return byMethodInheritanceAnnotationMatcher(named("org.springframework.web.bind.annotation.GetMapping"))
                        .or(byMethodInheritanceAnnotationMatcher(named("org.springframework.web.bind.annotation.PostMapping")))
                        .or(byMethodInheritanceAnnotationMatcher(named("org.springframework.web.bind.annotation.PutMapping")))
                        .or(byMethodInheritanceAnnotationMatcher(named("org.springframework.web.bind.annotation.DeleteMapping")))
                        .or(byMethodInheritanceAnnotationMatcher(named("org.springframework.web.bind.annotation.PatchMapping")));
                }
            }
        };
    }
```

- 同时定义了用来增强该类的增强类`org.apache.skywalking.apm.plugin.spring.mvc.v5.ControllerConstructorInterceptor`

## InstanceConstructorInterceptor 进行类的增强

拦截构造实例的增强，直接对该Controller进行增强。该段代码中的注释比较重要！

```java
/**
 * The <code>ControllerConstructorInterceptor</code> intercepts the Controller's constructor, in order to acquire the
 * mapping annotation, if exist.
 * <p>
 * But, you can see we only use the first mapping value, <B>Why?</B>
 * <p>
 * Right now, we intercept the controller by annotation as you known, so we CAN'T know which uri patten is actually
 * matched. Even we know, that costs a lot.
 * <p>
 * If we want to resolve that, we must intercept the Spring MVC core codes, that is not a good choice for now.
 * <p>
 * Comment by @wu-sheng
 */
public class ControllerConstructorInterceptor implements InstanceConstructorInterceptor {

    @Override
    public void onConstruct(EnhancedInstance objInst, Object[] allArguments) {
        String basePath = "";
        RequestMapping basePathRequestMapping = objInst.getClass().getAnnotation(RequestMapping.class);
        if (basePathRequestMapping != null) {
            if (basePathRequestMapping.value().length > 0) {
                basePath = basePathRequestMapping.value()[0];
            } else if (basePathRequestMapping.path().length > 0) {
                basePath = basePathRequestMapping.path()[0];
            }
        }
        EnhanceRequireObjectCache enhanceRequireObjectCache = new EnhanceRequireObjectCache();
        // PathMappingCache对象会存储方法中方法与URL映射的映射关系
        enhanceRequireObjectCache.setPathMappingCache(new PathMappingCache(basePath));
        objInst.setSkyWalkingDynamicField(enhanceRequireObjectCache);
    }
}
```

- Right now, we intercept the controller by annotation as you known, so we CAN'T know which uri patten is actually matched. Even we know, that costs a lot.

  截止目前，我们用注解的方式拦截controller，所以我们不知道匹配的具体的URI。即使我们能够知道，但是那个消耗太大。

- If we want to resolve that, we must intercept the Spring MVC core codes, that is not a good choice for now.

  如果我们想要去解决这个问题，我们必须拦截Spring MVC 核心代码，但是目前那不是个好选择。

这段话，说明了一个问题，如果有如下代码：

```java
    @GetMapping("/test/{pageId}")
    public ApiResult<Boolean> test(@PathVariable("pageId") String pageId) {
        return ApiResult.ok(true);
    }
```

那么在链路追踪中，看到的链路就是：`{GET} XXX/test/{pageId}`，那么这个pageId并不会使用实际的值，例如“00001”等等，这么做主要考虑了性能的消耗。

不过这么做，对于进行性能分析会带来一定的困扰。尤其是我们工程中，每一个页面都是特殊编辑生成的，每个页面的性能相差巨大，如果仅仅是这样的展示，我们运用链路追踪中的Duration进行排序，则不会按照pageId进行归类了，很是头疼。**所以你有解决方案吗？**

## AbstractMethodInterceptor 进行方法的增强

`org.apache.skywalking.apm.plugin.spring.mvc.commons.interceptor.AbstractMethodInterceptor`是`RequestMappingMethodInterceptor`以及`RestMappingMethodInterceptor`的父类。对方法进行增强，将上一步的映射关系在`EnhanceRequireObjectCache`中真正形成。并且生成对应的EntrySpan或者LocalSpan。

```java
    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                             MethodInterceptResult result) throws Throwable {

        Boolean forwardRequestFlag = (Boolean) ContextManager.getRuntimeContext().get(FORWARD_REQUEST_FLAG);
        /**
         * Spring MVC plugin do nothing if current request is forward request.
         * Ref: https://github.com/apache/skywalking/pull/1325
         */
        if (forwardRequestFlag != null && forwardRequestFlag) {
            return;
        }

        String operationName;
        if (SpringMVCPluginConfig.Plugin.SpringMVC.USE_QUALIFIED_NAME_AS_ENDPOINT_NAME) {
            operationName = MethodUtil.generateOperationName(method);
        } else {
            EnhanceRequireObjectCache pathMappingCache = (EnhanceRequireObjectCache) objInst.getSkyWalkingDynamicField();
            String requestURL = pathMappingCache.findPathMapping(method);
            if (requestURL == null) {
                requestURL = getRequestURL(method);
                // 添加方法与URL的映射关系
                pathMappingCache.addPathMapping(method, requestURL);
                requestURL = pathMappingCache.findPathMapping(method);
            }
            operationName = getAcceptedMethodTypes(method) + requestURL;
        }

        RequestHolder request = (RequestHolder) ContextManager.getRuntimeContext()
                                                              .get(REQUEST_KEY_IN_RUNTIME_CONTEXT);
        if (request != null) {
            StackDepth stackDepth = (StackDepth) ContextManager.getRuntimeContext().get(CONTROLLER_METHOD_STACK_DEPTH);

            if (stackDepth == null) {
                // 栈为空，为一个新的链路
                ContextCarrier contextCarrier = new ContextCarrier();
                CarrierItem next = contextCarrier.items();
                while (next.hasNext()) {
                    next = next.next();
                    next.setHeadValue(request.getHeader(next.getHeadKey()));
                }
				// 创建一个EntrySpan
                AbstractSpan span = ContextManager.createEntrySpan(operationName, contextCarrier);
                Tags.URL.set(span, request.requestURL());
                Tags.HTTP.METHOD.set(span, request.requestMethod());
                span.setComponent(ComponentsDefine.SPRING_MVC_ANNOTATION);
                SpanLayer.asHttp(span);

                if (SpringMVCPluginConfig.Plugin.SpringMVC.COLLECT_HTTP_PARAMS) {
                    collectHttpParam(request, span);
                }

                if (!CollectionUtil.isEmpty(SpringMVCPluginConfig.Plugin.Http.INCLUDE_HTTP_HEADERS)) {
                    collectHttpHeaders(request, span);
                }

                stackDepth = new StackDepth();
                // 加入一个栈深度
                ContextManager.getRuntimeContext().put(CONTROLLER_METHOD_STACK_DEPTH, stackDepth);
            } else {
                // 本地调用，创建一个LocalSpan
                AbstractSpan span = ContextManager.createLocalSpan(buildOperationName(objInst, method));
                span.setComponent(ComponentsDefine.SPRING_MVC_ANNOTATION);
            }

            stackDepth.increment();
        }
    }

```

## 总结

Controller的链路收集，主要分两步，一步是对类的增强，一步是对方法的增强。代码均位于apm-sniffer/apm-sdk-plugin/spring-plugins子工程中。

skywalking收集链路时，使用的URL都是通配符，在链路中，无法针对某个pageId，或者其他通配符的具体的值进行查找。或许skywalking出于性能考虑，但是对于这种不定的通用大接口，的确无法用于针对性的性能分析了。这个问题还需要深入思考寻找，能否在自己项目中，找到一个Balance，只针对自己需要分析性能的接口，按照具体值进行收集，而其他走默认呢？
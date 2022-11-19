# skywalking09 - 异步线程链路续接（下）--源码分析

> 在上篇，我们提到了，多线程可能会导致链路断开，而可以通过三种方式将其接上。那你有没有好奇，为什么它会断开，它又是怎么接上的呢？

## 链路为何断开

要知道链路为何断开，我们就需要知道，正常情况下的链路是如何工作的，几个Span之间是如何接在一起的。我们可以通过第四篇提到的`@Trace`注解进行入手，这个注解会增加一个Span。

### 正常情况下@Trace添加Span

对skywalking源码有一定了解的你一定知道，其对类做修改增强的时候，会定义一个该类全类名的字符串，以及会用来增强该类的增强类的全类名，所以我们找到了`TraceAnnotationActivation`：

```java
/**
 * {@link TraceAnnotationActivation} enhance all method that annotated with <code>org.apache.skywalking.apm.toolkit.trace.annotation.Trace</code>
 * by <code>TraceAnnotationMethodInterceptor</code>.
 */
public class TraceAnnotationActivation extends ClassInstanceMethodsEnhancePluginDefine {
   // 用来增强的类
    public static final String TRACE_ANNOTATION_METHOD_INTERCEPTOR = "org.apache.skywalking.apm.toolkit.activation.trace.TraceAnnotationMethodInterceptor";
    // 被增强处理的注解
    public static final String TRACE_ANNOTATION = "org.apache.skywalking.apm.toolkit.trace.Trace";

    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return new ConstructorInterceptPoint[0];
    }

    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return new InstanceMethodsInterceptPoint[] {
            new DeclaredInstanceMethodsInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getMethodsMatcher() {
                    return isAnnotatedWith(named(TRACE_ANNOTATION));
                }

                @Override
                public String getMethodsInterceptor() {
                    return TRACE_ANNOTATION_METHOD_INTERCEPTOR;
                }

                @Override
                public boolean isOverrideArgs() {
                    return false;
                }
            }
        };
    }

    @Override
    protected ClassMatch enhanceClass() {
        return MethodAnnotationMatch.byMethodAnnotationMatch(TRACE_ANNOTATION);
    }
}
```

然后我们去翻查TraceAnnotationMethodInterceptor:

```java
/**
 * {@link TraceAnnotationMethodInterceptor} create a local span and set the operation name which fetch from
 * <code>org.apache.skywalking.apm.toolkit.trace.annotation.Trace.operationName</code>. if the fetch value is blank
 * string, and the operation name will be the method name.
 */
public class TraceAnnotationMethodInterceptor implements InstanceMethodsAroundInterceptor {
    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                             MethodInterceptResult result) throws Throwable {
        Trace trace = method.getAnnotation(Trace.class);
        final AbstractSpan localSpan = ContextManager.createLocalSpan(operationName);

    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                              Object ret) throws Throwable {
   
        ContextManager.stopSpan();
        
        return ret;
    }

}
```

代码做了一定精简，我们可以看到，在beforeMethod()方法中，其使用ContextManager创建了一个LocalSpan，在afterMethod()方法中，使用了ContextManager停止了Span。而`ContextManager`是skywalking链路管理的一个核心的类，那么也一定是它在创建Span的时候没有续接上导致的。

### `ContextManager`创建Span的核心

```java
/**
 * {@link ContextManager} controls the whole context of {@link TraceSegment}. Any {@link TraceSegment} relates to
 * single-thread, so this context use {@link ThreadLocal} to maintain the context, and make sure, since a {@link
 * TraceSegment} starts, all ChildOf spans are in the same context. <p> What is 'ChildOf'?
 * https://github.com/opentracing/specification/blob/master/specification.md#references-between-spans
 *
 * <p> Also, {@link ContextManager} delegates to all {@link AbstractTracerContext}'s major methods.
 */
public class ContextManager implements BootService {
    private static final String EMPTY_TRACE_CONTEXT_ID = "N/A";
    private static final ILog LOGGER = LogManager.getLogger(ContextManager.class);
    private static ThreadLocal<AbstractTracerContext> CONTEXT = new ThreadLocal<AbstractTracerContext>();
    private static ThreadLocal<RuntimeContext> RUNTIME_CONTEXT = new ThreadLocal<RuntimeContext>();
    private static ContextManagerExtendService EXTEND_SERVICE;

    private static AbstractTracerContext getOrCreate(String operationName, boolean forceSampling) {
        AbstractTracerContext context = CONTEXT.get();
        if (context == null) {
            if (StringUtil.isEmpty(operationName)) {
                if (LOGGER.isDebugEnable()) {
                    LOGGER.debug("No operation name, ignore this trace.");
                }
                context = new IgnoredTracerContext();
            } else {
                if (EXTEND_SERVICE == null) {
                    EXTEND_SERVICE = ServiceManager.INSTANCE.findService(ContextManagerExtendService.class);
                }
                context = EXTEND_SERVICE.createTraceContext(operationName, forceSampling);

            }
            CONTEXT.set(context);
        }
        return context;
    }

    private static AbstractTracerContext get() {
        return CONTEXT.get();
    }
    
        public static AbstractSpan createLocalSpan(String operationName) {
        operationName = StringUtil.cut(operationName, OPERATION_NAME_THRESHOLD);
        AbstractTracerContext context = getOrCreate(operationName, false);
        return context.createLocalSpan(operationName);
    }
}
```

- 看到人家写的注释没，“Any TraceSegment relates to single-thread, so this context use ThreadLocal to maintain the context, and make sure, since a TraceSegment starts, all ChildOf spans are in the same context.” 一条链路就是一个单线程的，所以用了ThreadLocal来保存，让我们自己来保证，子Span是同一个上下文中的。
- `ThreadLocal<AbstractTracerContext> CONTEXT` 这一个变量，用来存Span，那难怪了，**新的线程中，它就是断开的**。

## 链路如何续接

​		我们搞清楚了，断开是因为CONTEXT是存在ThreadLocal中的，导致新的线程中没有上下文，那么我们只要将父线程的上下文传入进去，就可以完成续接。那让我们来看看skywalking是怎么做的。我们以`@TraceCrossThread`为例，其他方式大体思路是一致的。

```java
/**
 * {@link CallableOrRunnableActivation} presents that skywalking intercepts all Class with annotation
 * "org.skywalking.apm.toolkit.trace.TraceCrossThread" and method named "call" or "run".
 */
public class CallableOrRunnableActivation extends ClassInstanceMethodsEnhancePluginDefine {

    public static final String ANNOTATION_NAME = "org.apache.skywalking.apm.toolkit.trace.TraceCrossThread";
    private static final String INIT_METHOD_INTERCEPTOR = "org.apache.skywalking.apm.toolkit.activation.trace.CallableOrRunnableConstructInterceptor";
    private static final String CALL_METHOD_INTERCEPTOR = "org.apache.skywalking.apm.toolkit.activation.trace.CallableOrRunnableInvokeInterceptor";
    private static final String CALL_METHOD_NAME = "call";
    private static final String RUN_METHOD_NAME = "run";
    private static final String GET_METHOD_NAME = "get";

    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return new ConstructorInterceptPoint[] {
            new ConstructorInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getConstructorMatcher() {
                    return any();
                }

                @Override
                public String getConstructorInterceptor() {
                    return INIT_METHOD_INTERCEPTOR;
                }
            }
        };
    }

    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return new InstanceMethodsInterceptPoint[] {
            new InstanceMethodsInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getMethodsMatcher() {
                    return named(CALL_METHOD_NAME)
                        .and(takesArguments(0))
                        .or(named(RUN_METHOD_NAME).and(takesArguments(0)))
                        .or(named(GET_METHOD_NAME).and(takesArguments(0)));
                }

                @Override
                public String getMethodsInterceptor() {
                    return CALL_METHOD_INTERCEPTOR;
                }

                @Override
                public boolean isOverrideArgs() {
                    return false;
                }
            }
        };
    }

    @Override
    protected ClassMatch enhanceClass() {
        return byClassAnnotationMatch(new String[] {ANNOTATION_NAME});
    }

}
```

通过全局搜索，我们找到`CallableOrRunnableActivation`，它完成了对"org.skywalking.apm.toolkit.trace.TraceCrossThread" and method named "call" or "run"的增强。增强方式分为构造时增强、以及对方法的增强。

### CallableOrRunnableConstructInterceptor

```java
public class CallableOrRunnableConstructInterceptor implements InstanceConstructorInterceptor {
    @Override
    public void onConstruct(EnhancedInstance objInst, Object[] allArguments) {
        if (ContextManager.isActive()) {
            objInst.setSkyWalkingDynamicField(ContextManager.capture());
        }
    }

}
```

在构造的时候，ContextManager对当前的上下文做了一次快照，并存到`skyWalkingDynamicField`这个动态属性中，共子线程来取。

### CallableOrRunnableInvokeInterceptor

```java
public class CallableOrRunnableInvokeInterceptor implements InstanceMethodsAroundInterceptor {
    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
        MethodInterceptResult result) throws Throwable {
        ContextManager.createLocalSpan("Thread/" + objInst.getClass().getName() + "/" + method.getName());
        ContextSnapshot cachedObjects = (ContextSnapshot) objInst.getSkyWalkingDynamicField();
        if (cachedObjects != null) {
            ContextManager.continued(cachedObjects);
        }
    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
        Object ret) throws Throwable {
        ContextManager.stopSpan();
        // clear ContextSnapshot
        objInst.setSkyWalkingDynamicField(null);
        return ret;
    }

}

```

这个类，将`skyWalkingDynamicField`这个动态属性中的内容取出，并通过“ContextManager.continued(cachedObjects);”完成了续接。最后，在afterMethod()中，也完成了对Span的关闭。

### TracingContext#continued

```java
    /**
     * Continue the context from the given snapshot of parent thread.
     *
     * @param snapshot from {@link #capture()} in the parent thread. Ref to {@link AbstractTracerContext#continued(ContextSnapshot)}
     */
    @Override
    public void continued(ContextSnapshot snapshot) {
        if (snapshot.isValid()) {
            TraceSegmentRef segmentRef = new TraceSegmentRef(snapshot);
            this.segment.ref(segmentRef);
            this.activeSpan().ref(segmentRef);
            this.segment.relatedGlobalTraces(snapshot.getTraceId());
            this.correlationContext.continued(snapshot);
            this.extensionContext.continued(snapshot);
            this.extensionContext.handle(this.activeSpan());
        }
    }
```

这个注释也很明白的说明了，这个上下文会将父线程的快照进行续接。



## 总结

第一步，通过对对象的构造方法进行增强，将链路上下文快照作为动态属性赋值给子线程；第二步，子线程的异步方法在开始前，将快照续接上并创建新的Span，方法结束后将Span关闭。

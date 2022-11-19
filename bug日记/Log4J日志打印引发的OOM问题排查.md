# Log4J日志打印引发的OOM问题排查

> 上周，充当回消防队员去救火，一个老的CRM系统，生产上一天出现了CPU占用高，两次OOM问题。从时间上看，CPU占用高的报警也是因为JVM为了自救的疯狂GC导致的。



## 查看Dump文件

OOM提供了堆Dump以及线程栈Dump。由于是内网，无法截图，也不方便拍照。在此就引用一篇来自老东家的，极度相似的博客：https://rdc.hundsun.com/portal/article/884.html

### 栈Dump

从栈Dump线程中，可以看到大部分线程都在com.lmax.disruptor.RingBuffer#next()的UNSAFE.park()方法中。其堆栈的入口全是log4j的append方法中。

### 堆Dump

用mat打开was的Dump phd文件（工具不一定，但是was的我还是第一次遇到这个格式）

堆Dump中可以看到4G的内存，3.8G均为RingBuffer占用，毫无疑问，这个类所占肯定过多，而且还没法回收。



## 排查问题

JVM 内存溢出无非两种大情况，内存泄露，或者内存不足。情况一不管分配多大内存都不会足够，情况二，需要考虑加大内存，或者减少、限制内存的使用。

对于log4j这种成熟的组件，我一般不会怀疑其是内存泄露。当然，我还是找到那个工程使用的版本，下载源码进行查看：

```xml
        <!-- https://mvnrepository.com/artifact/com.lmax/disruptor -->
        <dependency>
            <groupId>com.lmax</groupId>
            <artifactId>disruptor</artifactId>
            <version>3.3.6</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-core -->
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.3</version>
        </dependency>
```

其版本为上述版本。对于RingBuffer，其使用的处理不加赘述，直接上代码证明其没有内存泄露，以及会释放内存的代码：

```java
// org.apache.logging.log4j.core.async.RingBufferLogEventHandler#onEvent
    @Override
    public void onEvent(final RingBufferLogEvent event, final long sequence,
            final boolean endOfBatch) throws Exception {
        event.execute(endOfBatch);
        event.clear();

        // notify the BatchEventProcessor that the sequence has progressed.
        // Without this callback the sequence would not be progressed
        // until the batch has completely finished.
        if (++counter > NOTIFY_PROGRESS_THRESHOLD) {
            sequenceCallback.set(sequence);
            counter = 0;
        }
    }
```

```java
// org.apache.logging.log4j.core.async.RingBufferLogEvent#clear

    /**
     * Release references held by ring buffer to allow objects to be garbage-collected.
     */
    public void clear() {
        setValues(null, // asyncLogger
                null, // loggerName
                null, // marker
                null, // fqcn
                null, // level
                null, // data
                null, // t
                null, // map
                null, // contextStack
                null, // threadName
                null, // location
                0 // currentTimeMillis
        );
    }
```

其通过clear方法，对所有属性都设置为null，引用释放，可以交给JVM进行内存释放的。

所以，应该和篇头博客遇到的问题一样，就是buffer声明的过大：

```java
private static final int RINGBUFFER_DEFAULT_SIZE = 256 * 1024;
private static int calculateRingBufferSize() {
        int ringBufferSize = RINGBUFFER_DEFAULT_SIZE;
        final String userPreferredRBSize = PropertiesUtil.getProperties().getStringProperty(
                "AsyncLoggerConfig.RingBufferSize",
                String.valueOf(ringBufferSize));
}
```

默认256k的条数缓存，在每条日志都很大的时候，肯定会撑爆内存。当然，我没因为监控没有历史记录，我们并不知道发生OOM时，是并发太高导致日质量堆积，还是IO（磁盘、网络）导致的写文件或者发送kafka太慢导致的堆积。

> 至于怎么找到这些源码的位置，暂时没时间写。



## 解决方案

设置`AsyncLoggerConfig.RingBufferSize`值为4096或者其他稍小的值，宁愿丢日志，也不愿意节点挂掉，自己选择取舍吧。




# Java通过File获取Class字节码并构造Class对象

## 步骤

1. 已经在运行的jvm中的一个class的文件，或者外部某个位置的.class文件,并读取字节流。

```java
public class ClassUtil {

    /**
     * 将当前的类转为byte[]
     *
     * @param name 全类名
     * @return 字节码byte数组
     */
    public static byte[] classToBytes(String name) {
        // 获取classLoader是为了从根路径开始读取文件
        ClassLoader classLoader = ClassUtil.class.getClassLoader();
        String path = name.replaceAll("\\.", "\\" + File.separator);
        path += ".class";
        URL url = classLoader.getResource(path);
        if (null == url) {
            return null;
        }
        // 检测类是否存在
        File classFile = new File(url.getFile());
        if (!classFile.exists()) {
            return null;
        }
        byte[] result = null;
        try (InputStream is = url.openStream();) {
            result = new byte[(int) new File(url.getFile()).length()];
            is.read(result);
        } catch (IOException e) {
            System.out.println(e.getMessage());
        }
        return result;
    }
}
```

2. 通过ClassLoader的defineClass方法进行Class对象的定义，不过由于这是一个protected修饰方法，需要自定义一个classLoader将其暴露出来。

```java
// -- 自定义一个classloader,用公有方法暴露保护方法
public class MyClassLoader extends ClassLoader {
    public Class<?> defineClass(String name, byte[] clazzFile) {
        return defineClass(name, clazzFile, 0, clazzFile.length);
    }
}

// -- 调用该方法返回Class<?>
public class ClassUtil {
    /**
     * 定义类
     *
     * @param name      全类名
     * @param clazzFile 字节码文件
     * @return 类
     */
    public static Class<?> defineClass(String name, byte[] clazzFile) {
        MyClassLoader myClassLoader = new MyClassLoader();
        return myClassLoader.defineClass(name, clazzFile);
    }
}
```

3. 接下来通过反射进行该类的各种调用即可。

   

## 有啥用呢

你可能问我这样搞到底有啥意义？

因为我刚发现了有个有意思的东西，我们通过javaagent进行代理的包，它里面的类其实都是在程序中可以使用的。

```bash
-javaagent:/home/dark/IdeaProjects/DeCoverAgent/target/decover-agent.jar
```

但是它里面的类，在编码时我们都是不知道的，所以可以尝试自己反射去调用它。（其实不去搞那么麻烦也可以直接反射调用的哈哈哈）。

其实我搞这么麻烦，本来初心是读取agent里面的class，转成byte[]， 然后在`ClassFileTransformer#transform`中进行替换原有的类的～

详细可以了解一下java agent机制（java5引入）。



硬要说有什么用，就是把内部的class文件转byte[]，用于网络传输给别的虚拟机还是干啥就看你的了～
# 使用场景描述

在业务方使用我封装的mybatis的时候，需要继承我提供的AbstractDao，然后传入自己的映射器接口就可以使用mybatis，具体如下：

```java
public class FrameDynamicDBDao extends AbstractDao
{
    public FrameDynamicDBDao(String dbName)
    {
        super(dbName);
    }

    public List<NeSyncStatePojo> getAllNeSyncState(){
        FrameDynamicDBMapper mapper = getMapper(FrameDynamicDBMapper.class);
        return mapper.getAllNeSyncState();
    }
}
```

目前在osgi环境下，提供mybatis封装的jar包在A bundle中，业务使用方在B bundle中。

那么业务方为了能继承到基类，必须在A bundle中 export `AbstractDao`所在的包路径：

```
Export-Package:
com.xxx.xxx.dao
```

B bundle 必须import这个包路径:

```
Import-Package:
com.xxx.xxx.dao
```

这样，业务方就可以正确的引用到这个AbstractDao基类。

# 问题描述

但是像如上配置之后，业务方虽然能引用框架提供的AbstractDao。但是框架这边却无法识别业务方提供的映射器。原因很简单，因为osgi的类加载模式是不同bundle对应 有自己的类加载器。A bundle无法加载在B bundle中的类。

那么有如下解决方案：

### 方案一

bundle B export自己的映射器包，然后框架import相应的路径。



这么做虽然能让框架识别业务方的映射器，但是这样设计并不优雅。而且只能应付只业务方只有一个B bundle使用框架的mybatis的情况。如果在一个osgi环境中有两个业务方bundle B和C。那么这种方案就不可行。

而且这样做会让A Bbundle 耦合在一起，如果在另一个osgi环境中没有B bundle ，那么A bundle就会因为import 找不到相应的export而报错。



### 方案二

获取并使用业务方的bundle的类加载器。



因为业务方在调用框架的代码 `getMapper(FrameDynamicDBMapper.class);`的时候会把映射器接口类传进来。那么我们可以通过`clazz.getClassLoader()`来获取业务方B bundle的类加载器。

另一方面，mybatis在解析映射器的xml和接口的时候，是通过反射来获取相应的对象的。通过阅读mybatis的源码，可以发现其中有一个`ClassLoaderWrapper`类，其中的classForName，getResourceAsStream两个方法是用来反射类和解析资源的，其中一个方法如下：

```java
Class<?> classForName(String name, ClassLoader[] classLoader) throws ClassNotFoundException {
        ClassLoader[] var3 = classLoader;
        int var4 = classLoader.length;

        for(int var5 = 0; var5 < var4; ++var5) {
            ClassLoader cl = var3[var5];
            if (null != cl) {
                try {
                    Class<?> c = Class.forName(name, true, cl);
                    if (null != c) {
                        return c;
                    }
                } catch (ClassNotFoundException var8) {
                    ;
                }
            }
        }

        throw new ClassNotFoundException("Cannot find class: " + name);
    }
```

可以看到`Class<?> c = Class.forName(name, true, cl);`这句是通过类加载器来反射的，而传入的类加载器是一个数组，调用这个方法的地方是通过如下方法获取类加载器数组的：

```java
ClassLoader[] getClassLoaders(ClassLoader classLoader) {
        return new ClassLoader[]{
          classLoader, 
          this.defaultClassLoader, 
          Thread.currentThread().getContextClassLoader(), 
          this.getClass().getClassLoader(), 
          this.systemClassLoader
        };
    }
```

可以看到这里的类加载器数组包括了线程上下文类加载器，系统类加载器和自己所在类的类加载器。然后classLoader是传入的，defaultClassLoader默认为空。

那么我们可以在哪将类加载器传入这个数组呢，通过看代码发现mybatis的常用类`Resources`。这个类代码如下：

```
public class Resources {
    private static ClassLoaderWrapper classLoaderWrapper = new ClassLoaderWrapper();
    private static Charset charset;

    Resources() {
    }

    public static ClassLoader getDefaultClassLoader() {
        return classLoaderWrapper.defaultClassLoader;
    }

    public static void setDefaultClassLoader(ClassLoader defaultClassLoader) {
        classLoaderWrapper.defaultClassLoader = defaultClassLoader;
    }
    ...
}
```

可以看到是这个Resources类管理了一个classLoaderWrapper对象，然后可以设置他的defaultClassLoader。

那么现在方案就很明显了，通过这里将业务bundle的类加载器设置进来，这样mybatis就可以正确的反射和识别业务方的映射器了。



最终的生成sqlsessionFactory的代码如下：

```java
...
Resources.setDefaultClassLoader(clazz.getClassLoader());

Properties properties = getDatasourceProp(dbName);
sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream,properties);
sqlSessionFactory.getConfiguration().addMapper(clazz);
...
```




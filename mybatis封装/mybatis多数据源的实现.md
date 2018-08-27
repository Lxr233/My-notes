## 概述

最近有个需求，需要支持不同数据库类型的动态切换。而老代码的sql都是在代码中拼装的，如果要支持不同数据库类型（sybase、mysql）需要在代码中加if else。鉴于实现太low，需要将其与业务层解耦，所以需要切换至mybatis。由于是老代码，存在一个进程连接多个数据库的情况，所以切换mybatis的时候也要提供连接不同数据库的功能。

现在需要提供一个mybatis的封装，具体要求如下：

* 不能在代码中拼装sql，需要写到配置文件中（mybatis）
* Sybase、mysql等数据库切换的时候不能简单的使用if else来判断
* 数据库的连接信息在运行时才能获取
* 提供各进程连接多数据库的支持
* 使用者可以自己配置数据库连接池的各种参数，不配置的提供一套默认的配置
* 因为提供的是jar包，要轻量化，所以只能引入mybatis，不能引入Spring


目前封装的目录结构如下：

![目录结构](C:\Users\l00427576\Desktop\学习\笔记\java\mybatis封装\目录结构.jpg)

其中被选中的部分是使用方在自己进程中编写的，不会收入到提供的jar包当中，剩下的部分就是将会封装到jar包中的。

##  使用

映射器文件和数据库配置文件需要使用方和框架约定一个包路径，比如：

`xxx.xxx.mybatis.resource.mapper`



MCDBMapper.java

```java
public interface MCDBMapper
{
    List<Pojo> getAllBrdType();
}
```

MCDBMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
  PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="xxx.xxx.mybatis.resource.mapper.MCDBMapper">
  <select id="getAllBrdType" resultType="Pojo" databaseId="mysql">
    select * from tableName
  </select>
  <select id="getAllBrdType" resultType="Pojo" databaseId="sybase">
    select * from tableName
  </select>
</mapper>
```

这个映射器是对应MCDB这个数据库的。

这里命名规则需要是  `数据库名+Mapper`

根据xml文件中的databaseId标识可以动态识别当是sybase或者mysql数据库时使用对应设置好的sql语句。如果sybase和mysql语句一样，没有特性，则可以去掉databaseId字段，那么无论什么数据库类型都是默认使用那条sql语句。



之后使用Dao层将数据层和业务层解耦：

MCDBDao.java

```java
public class MCDBDao extends AbstractDao
{
    public MCDBDao(){
        super("MCDB");
    }

    public List<Pojo> getAllBrdType(){
        MCDBMapper mapper = getMapper(MCDBMapper.class);
        return mapper.getAllBrdType();
    }
}
```

命名规则也是  `数据库名+Mapper`

需要继承`AbstractDao`基类。然后在构造函数的时候需要调用父构造函数，并提供需要连接到的数据库的名字，最好和文件命名的一样。

之后这里就可以调用框架提供的`getMapper`方法获取到之前设置的映射器接口的实现。



最后调用这个Dao的实例，就可以在业务中使用：

```Java
MCDBDao mcdbDao = new MCDBDao();
List<Pojo> insertableAlmList = mcdbDao.getAllBrdType();
```



## 封装介绍

AbstractDao.java基类：

```Java
public abstract class AbstractDao
{

    protected String dbName;


    public AbstractDao(String dbName)
    {
        this.dbName = dbName;
    }

    /**
     * 获取Mybatis Mapper
     *
     * @param <T> 类型
     * @return Mapper
     */
    public <T> T getMapper(Class<T> clazz)
    {
        return MapperMgr.createMapper(clazz, dbName);
    }
}
```

使用方通过构造函数指定了dbName，然后调用MapperMgr来动态生成mapper的实现。这里需要提供dbName给基类，这样子类再调用父类的getMapper方法就不需要再指定dbName了

`这里dbName是用来标识连接哪个数据库`



MapperMgr.java

```Java
public class MapperMgr
{

    public static <T> T createMapper(Class<T> clazz, String dbName) {
        SqlSessionFactory sqlSessionFactory = getSqlSessionFactory(dbName);
        SqlSession sqlSession = sqlSessionFactory.openSession();
        System.out.println(sqlSessionFactory.getConfiguration().getDatabaseId());
        T mapper = sqlSession.getMapper(clazz);
        return MapperProxy.bind(mapper, sqlSession);
    }

    private static class MapperProxy <T> implements InvocationHandler
    {
        private T mapper;
        private SqlSession sqlSession;

        private MapperProxy(T mapper, SqlSession sqlSession) {
            this.mapper = mapper;
            this.sqlSession = sqlSession;
        }

        private static<T> T bind(T mapper, SqlSession sqlSession) {
            return (T) Proxy.newProxyInstance(mapper.getClass().getClassLoader(),
                    mapper.getClass().getInterfaces(), new MapperProxy(mapper, sqlSession));
        }

        /**
         * 动态代理，执行mapper的方法，然后关闭sqlsession，同时捕获异常
         */
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            Object object = null;
            try {
                object = method.invoke(mapper, args);
                sqlSession.commit();
            } catch(Exception e) {
                sqlSession.rollback();
            } finally {
                sqlSession.close();
            }
            return object;
        }
    }

    private static SqlSessionFactory getSqlSessionFactory(String dbName) {
        return SqlSessionFactoryUtil.getSqlSessionFactory(dbName);
    }
}
```

其中的createMapper用于返回映射器接口的实现。其中主要是根据dbName获取缓存好的SqlSessionFactory，然后生成sqlSession。

这里使用了一个动态代理来将sqlSession的操作拦截并封装。

可以看到我将mapper接口进行拦截，并且执行了`sqlSession.rollback()`和`sqlSession.close()`

这样就可以只将mapper实现类暴露给使用方，不需要使用方去管理sqlSession的关闭和回退等，只需要拿到mapper来执行即可。





配置文件mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
  <properties>
    <property name="maxActive" value="15"/>
    <property name="maxIdle" value="10"/>
    <property name="minIdle" value="0"/>
    <property name="maxWait" value="10000"/>
  </properties>
  <environments default="product">
    <environment id="product">
      <transactionManager type="JDBC"/>
      <dataSource type="xxx.xxx.mybatis.src.DbcpDataSourceFactory">
        <property name="driver" value="${driver}" />
        <property name="url" value="${url}" />
        <property name="username" value="${username}" />
        <property name="password" value="${password}" />
        <property name="maxActive" value="${maxActive}"/>
        <property name="maxIdle" value="${maxIdle}"/>
        <property name="minIdle" value="${minIdle}"/>
        <property name="maxWait" value="${maxWait}"/>
      </dataSource>
    </environment>
  </environments>
  <databaseIdProvider type="DB_VENDOR">
    <property name="MySQL" value="mysql"/>
    <property name="Sybase" value="sybase"/>
  </databaseIdProvider>
  <mappers>
    <package name="xxx.xxx.mybatis.resource.mapper"/>
  </mappers>
</configuration>
```

可以看到我提供了dbcp作为数据源，并且提供了一套数据库的默认参数。

这里关于mapper的文件位置需要与使用方约定，和前面提到的一样。



提供数据源的代码如下：

DbcpDataSourceFactory.java

```java
public class DbcpDataSourceFactory extends BasicDataSource implements DataSourceFactory
{

    private static Logger log = LoggerFactory.getLogger(DbcpDataSourceFactory.class);

    private Properties props = null;

    @Override
    public void setProperties(Properties props)
    {
        this.props = props;
    }

    @Override
    public DataSource getDataSource()
    {
        DataSource dataSource = null;
        try
        {
            dataSource = BasicDataSourceFactory.createDataSource(props);
        }
        catch (Exception e)
        {
            log.error("datasource init failed , {}", e.getMessage());
        }
        return dataSource;
    }
}
```





SqlSessionFactoryUtil.java

```java
class SqlSessionFactoryUtil
{

    private static Logger log = LoggerFactory.getLogger(SqlSessionFactoryUtil.class);

    private static final String CONFIGURATION_PATH = "xxx/mybatis/resource/mybatis-config.xml";

    private static final String DATASOURCECONF_PATH = "xxx/resource/datasource.properties";

    //dbname为key
    private static final Map<String, SqlSessionFactory> SQLSESSION_FACTORY_MAP = new HashMap<>();

    //读写锁
    private static final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();


    public static SqlSessionFactory getSqlSessionFactory(String dbName)
    {
        SqlSessionFactory sqlSessionFactory = null;
        /**
         * 读写锁保证线程同步
         */
        lock.readLock().lock();
        try
        {
            sqlSessionFactory = SQLSESSION_FACTORY_MAP.get(dbName);
            if (sqlSessionFactory != null)
            {
                return sqlSessionFactory;
            }
        }
        finally
        {
            lock.readLock().unlock();
        }


        lock.writeLock().lock();
        try (InputStream inputStream = Resources.getResourceAsStream(CONFIGURATION_PATH))
        {
            Properties properties = getDatasourceProp(dbName);
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream, properties);
            SQLSESSION_FACTORY_MAP.put(dbName, sqlSessionFactory);
        }
        catch (IOException e)
        {
            log.error("Get {} SqlSessionFactory error:",e.getMessage());
        }
        finally
        {
            lock.writeLock().unlock();
        }
        return sqlSessionFactory;

    }

    private static Properties getDatasourceProp(String dbName)
    {
        Properties properties = new Properties();
        /**
         * 运行时动态获取数据库连接信息
         */
        DbConf dbConf = MultipleDbConf.getDbConf(dbName);
        properties.put("username", dbConf.getUserName());
        properties.put("password", dbConf.getPassWord());
        properties.put("driverClassName", dbConf.getDriver());
        properties.put("url", dbConf.getUrl());

        /**
         * 读取各个进程配置的数据库信息，如果不存在则使用默认配置
         */
        try (InputStream inputStream = Resources.getResourceAsStream(DATASOURCECONF_PATH)){
            Properties dbconfigProperties = new Properties();
            dbconfigProperties.load(inputStream);
            properties.put("maxActive", dbconfigProperties.getProperty(dbName+".maxActive"));
            properties.put("maxIdle", dbconfigProperties.getProperty(dbName+".maxIdle"));
            properties.put("minIdle", dbconfigProperties.getProperty(dbName+".minIdle"));
            properties.put("maxWait", dbconfigProperties.getProperty(dbName+".maxWait"));
        }
        catch (IOException e)
        {
            log.error("datasource.properties is not exist , use default properties");
        }

        return properties;
    }
}
```

这个类主要用来缓存不同数据库对应的sqlsessionFactory。当MapperMgr调用getSqlSessionFactory时会从缓存中取数据，没有的话就生成一个放进缓存。这里使用读写锁保证线程安全。



# Java SPI机制

## 前言

虽然我自己在前段时间再总结一些Java知识，但是经过最近的面试发现，很多自己掌握的并不牢靠，所以决定把原来很多内容拆分出来一部分一部分自己写，这篇主要在梳理一遍Java的SPI 机制吧。温故而知新，可以为师矣。

---

## 介绍

Java SPI 全程为 Service Provider Interface，直译过来就是 服务提供商接口。我理解的概念的话就是，由JDK语言开发组制定一系列功能接口，但功能的具体实现是由各个服务商自行提供。这也满足的**依赖倒置原则**。依赖倒置原则的核心就是要我们面向接口编程,理解了面向接口编程。

## 具体事例

最熟悉的SPI服务应该就是JDBC了

在java.util.sql中定义了对于数据库功能的各个接口，以及数据库操作生命周期中各对象的接口

如Driver表示数据库的驱动器,Connection表示一次数据库的连接

我们在使用第三方实现的时候我们一般是直接通过java.util.sql中的DriverManager来获取具体的第三方Driver或者Connection，那么DriverManager是如何加载到第三方数据库的呢？

接下来我们就好好梳理一下从接口Class文件加载到第三方实现的加载，以及第三方实现的调用的完整流程,看一下DriverManager的源码

### 环境

```jdk1.8.0_144```

```xml
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.46</version>
        </dependency>
```

### 加载SPI第三方实现

```java
    static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }
```



```java
    private static void loadInitialDrivers() {
        String drivers;
        try {
            drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
                public String run() {
                    return System.getProperty("jdbc.drivers");
                }
            });
        } catch (Exception ex) {
            drivers = null;
        }

        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {

                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();
		//.... 此处省略一些不重要的内容

        for (String aDriver : driversList) {
            try {
                println("DriverManager.Initialize: loading " + aDriver);
         //关键是这里的Class.forName(aDriver,true,ClassLoader.getSystemClassLoader())
         //此处使用ClassLoader.getSystemClassLoader()直接使用了AppClassLoader来加载具体实现类,打破了双亲委派机制
                Class.forName(aDriver, true,
                        ClassLoader.getSystemClassLoader());
            } catch (Exception ex) {
                println("DriverManager.Initialize: load failed: " + ex);
            }
        }
    }
```



### 再看一下第三方的Driver实现类中都干了些什么，此处以com.mysql.jdbc的Driver为例

```java
    static {
        try {
        //将自己的Driver实例注册进java.util.sql.DriverManager的CopyOnWriteArrayList列表中，供后期使用
            DriverManager.registerDriver(new Driver());
        } catch (SQLException var1) {
            throw new RuntimeException("Can't register driver!");
        }
    }
```



### 当SPI接口调用第三方的功能实现，此处以JDBC的getConnection为例

```java
       try {
            DriverManager.getConnection("jdbc:mysql://127.0.0.1:3306/test","test","123456");
        } catch (SQLException e) {
            e.printStackTrace();
        }
```



```java
    @CallerSensitive
    public static Connection getConnection(String url)
        throws SQLException {

        java.util.Properties info = new java.util.Properties();
        //此处返回的是直接调用DriverManager的类，一般为我们自己的程序类
        return (getConnection(url, info, Reflection.getCallerClass()));
    }
```



```java
    private static Connection getConnection(
        String url, java.util.Properties info, Class<?> caller) throws SQLException {
		
        //此处为我们自己程序类的类加载器 一般为AppClassLoader
        ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
        synchronized(DriverManager.class) {
            // synchronize loading of the correct classloader.
            if (callerCL == null) {
                callerCL = Thread.currentThread().getContextClassLoader();
            }
        }

		//此处省略一些不重要的内容

        for(DriverInfo aDriver : registeredDrivers) {
            
            //由AppClassLoader检查是否已经装载了第三方驱动类
            if(isDriverAllowed(aDriver.driver, callerCL)) {
                try {
                    println("    trying " + aDriver.driver.getClass().getName());
                    Connection con = aDriver.driver.connect(url, info);
                   
                } catch (SQLException ex) {
                    if (reason == null) {
                        reason = ex;
                    }
                }

            } else {
                println("    skipping: " + aDriver.getClass().getName());
            }

        }
        //此处省略一些不重要的内容
       
    }
```

## 总结

到目前为止我们已经弄清楚，由BootstrapClassloader加载的协议接口类，如何打破双亲委派机制 来加载AppClassLoader才可以加载到的classpath中的第三方实现类

主要方式为

1. 使用ClassLoader.getSystemClassLoader来获取AppClassLoader

```java
      Class.forName(aDriver, true,ClassLoader.getSystemClassLoader());
```

2. 虽然实际使用中没有用到，使用Thread.currentThread().getContextClassLoader()获取AppClassLoader

```java
    synchronized(DriverManager.class) {
        // synchronize loading of the correct classloader.
        if (callerCL == null) {
            callerCL = Thread.currentThread().getContextClassLoader();
        }
    }
```
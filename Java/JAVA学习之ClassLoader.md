##                            JAVA学习之ClassLoader

### 前言

> 最近被 一句话所触动——**种一棵树最好的时间是十年前，其次是现在。**所以决定要开始记录自己的学习之路。

### 什么是类加载？

我们都知道，每个.java文件可以经过javac指令编译成.class文件，里面包含着java虚拟机的机器指令。当我们需要使用一个java类时，虚拟机会加载它的.class文件，创建对应的java对象。将.class调入虚拟机的过程，称之为加载。

![](https://img2020.cnblogs.com/blog/1479430/202003/1479430-20200326195721536-1009072198.png)

loading ：加载。通过类的完全限定名找到.class字节码文件，同时创建一个对象。

verification：验证。确保class字节码文件符合当前虚拟机的要求。

preparation：准备。这时候将static修饰的变量进行内存分配，同时设置初始值。

resolution：解析。虚拟机将常量池中的符号引用变为直接引用。

- 符号引用：在编译之后完成的，一个常量并没有进行内存分配，也就只能用符号引用。
- 直接引用：常量会在preparation阶段将常量进行内存分配，于是就可以建立直接的虚拟机内存联系，就可以直接引用。

initialization：初始化。类加载的最后阶段。如果这个类有超类，进行超类的初始化，执行类的静态代码块，同时给类的静态变量赋予初值。前面的preparation阶段是分配内存，都只是默认的值，并没有被赋予初值。

## 类加载在java中有两种方式

- 显示加载。通过class.fornNme(classname)或者this.getClass().getClassLoader.loadClass(classname)加载。即通过调用类加载器classLoader来完成类的加载
- 隐式加载。类在需要被使用时，不直接调用ClassLoader，而是虚拟机自动调用ClassLoader加载到内存中。

## 什么是ClassLoader？

类加载器的任务是将类的二进制字节流读入到JVM中，然后变成一个JVM能识别的class对象，同时实例化。于是我们就可以分解ClassLoader的任务。

1. 找到类的二进制字节流，并加载进来
2. 规则化为JVM能识别的class对象

我们查看源码，找到对应解决方案：

在ClassLoader中，定义了两个接口:

1. findClass(String name).
2. defineClass(byte[] b , int off , int len)

findClass用于找到二进制文件并读入，调用defineClass用字节流变成JVM能识别的class对象，同时实例化class对象。

我们来看个例子：

```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
	  // 获取类的字节数组,通过name找到类，如果你对字节码加密，需要自己解密出来
      byte[] classData = getClassData(name);  
      if (classData == null) {
          throw new ClassNotFoundException();
      } else {
	      //使用defineClass生成class对象
          return defineClass(name, classData, 0, classData.length);
      }
  }

```

## 代理模式

提到类加载器，一定得涉及的是委派模式。在JAVA中，ClassLoader存在一下几类，他们的关系如下：

![](https://img2020.cnblogs.com/blog/1479430/202003/1479430-20200326233208909-1834889441.png)



- Bootstrap ClassLoader：引导类加载器。采用原生c++实现，用于加载java的核心类（`%JAVA_HOME%/lib`路径下的核心类库或者 `-Xbootclasspath`指定下的jar包）到内存中。**没有父类**。

- Extension ClassLoader：扩展类加载器。java实现，加载`/lib/ext`目录下或者由系统变量-Djava.ext.dir指定位路径中的类库。**父类加载器为null。**

- System ClassLoader：它会根据java应用的类路径（CLASSPATH）来加载类，即`java -classpath`或`-D java.class.path` 指定路径下的类库，也就是我们经常用到的classpath路径。一般来说，java应用的类都是由它完成。可以由ClassLoader.getSystemLoader()方法获得。**父类加载器是Extension ClassLoader。**

- Customize ClassLoader：用户自定义加载器。用于完成用户自身的特有需求。**父类加载器为System ClassLoader。**

  1. 加载不在CLASSPATH路径的class文件
  2. 加密后的class文件需要用户自定义ClassLoader来解密后才能被JVM识别。
  3. 热部署，同一个class文件通过不同的类加载器产生不同的class对象。**两个类完全相同不仅仅要是同一个class文件，还需要同一个类加载器、同一个JVM。**

  而**代理模式** ，就是像上面图片所展示那样，当要加载一个类时，加载器会寻求其父类的帮助，让父类尝试去加载这个类。只有当父类失败后，才会由自己加载。ClassLoader的loadClass()方法中体现。

  示例代码：

  ```java
  protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        //如果找不到，则委托给父类加载器去加载
                        c = parent.loadClass(name, false);
                    } else {
                    //如果没有父类，则委托给启动加载器去加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }
  
                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // 如果都没有找到，则通过自定义实现的findClass去查找并加载
                    c = findClass(name);
  
                }
            }
            if (resolve) {//是否需要在加载时进行解析
                resolveClass(c);
            }
            return c;
        }
    }
  
  ```

  findClass(String name)的类似实现上面有示例，resolveClass()就是完成**解析**功能。

  ## URLClassLoader

  ![](https://img2020.cnblogs.com/blog/1479430/202003/1479430-20200327013957750-1913147479.png)

  URLClassLoader是常用的ClassLoader类，其实现了findclass()接口，所以如果自定义时继承URLClassLoader可以不用重写findclass()。ExtClassLoader在代理模式中属于Extension ClassLoader，而AppClassLoader属于System ClassLoader。

  

  ## 线程上下文加载器

  前面我们提到过**BootStrap ClassLoader加载的是%JAVA_HOME%/lib下的核心库文件，而CLASSPATH路径下的库由System ClassLoader加载。**但在java语言中，存在这种现象，Java 提供了很多服务提供者接口（Service Provider Interface，SPI），允许第三方为这些接口提供实现，如JDBC、JCE、JNDI等。而SPI是Java的核心库文件，由 BootStrap ClassLoader加载，第三方实现是放在CLASSPATH路径下，由System ClassLoader加载。当BootStrap ClassLoader启用时，需要加载其实现，但自己找不到，又因为**代理模式**的存在，无法委托System ClassLoader来加载，所以无法实现。

  Contex ClassLoader（线程上下文加载器）刚好可以解决这个问题。

  Contex ClassLoader（线程上下文加载器）是从 JDK 1.2 开始引入的。类 `java.lang.Thread`中的方法 `getContextClassLoader()`和 `setContextClassLoader(ClassLoader cl)`用来获取和设置线程的上下文类加载器。如果没有通过 `setContextClassLoader(ClassLoader cl)`方法进行设置的话，线程将继承其父线程的上下文类加载器。Java 应用运行的初始线程的上下文类加载器是系统类加载器。在线程中运行的代码可以通过此类加载器来加载类和资源。

  

  例子JDBC：

  ```java
  public class DriverManager {
  	//省略......
      static {
          loadInitialDrivers();
          println("JDBC DriverManager initialized");
      }
  
   private static void loadInitialDrivers() {
       sun.misc.Providers()
       AccessController.doPrivileged(new PrivilegedAction<Void>() {
              public Void run() {
                  ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                //省略不必要的代码......
              }
          });
      }
  public class Driver extends com.mysql.cj.jdbc.Driver {
      public Driver() throws SQLException {
          super();
      }
  
      static {
        //省略
      }
  }
  public static <S> ServiceLoader<S> load(Class<S> service) {
  	 //通过线程上下文类加载器加载
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }
  //调用
  String url = "jdbc:mysql://localhost:3342/cm-storylocker?characterEncoding=UTF-8";
  // 通过java库获取数据库连接
  Connection conn = java.sql.DriverManager.getConnection(url, "root", "password");
  
  
  ```

  我们可以看到，当我们在使用JDBC的时候，会使用有DriverManager类，它的static代码区引用的ServiceLoader类会完成JDBC实现类的加载。

  ## 如何设计自己的ClassLoader

  给出例子,重写文件系统加载器

  ```java
  public class FileSystemClassLoader extends ClassLoader { 
   
     private String rootDir; 
   
     public FileSystemClassLoader(String rootDir) { 
         this.rootDir = rootDir; 
     } 
   
     protected Class<?> findClass(String name) throws ClassNotFoundException { 
         //根据规则获取字节流数组
         byte[] classData = getClassData(name); 
         if (classData == null) { 
             throw new ClassNotFoundException(); 
         } 
         else { 
             return defineClass(name, classData, 0, classData.length); 
         } 
     } 
   //设定自己的读取规则
     private byte[] getClassData(String className) { 
         String path = classNameToPath(className); 
         try { 
             InputStream ins = new FileInputStream(path); 
             ByteArrayOutputStream baos = new ByteArrayOutputStream(); 
             int bufferSize = 4096; 
             byte[] buffer = new byte[bufferSize]; 
             int bytesNumRead = 0; 
             while ((bytesNumRead = ins.read(buffer)) != -1) { 
                 baos.write(buffer, 0, bytesNumRead); 
             } 
             return baos.toByteArray(); 
         } catch (IOException e) { 
             e.printStackTrace(); 
         } 
         return null; 
     } 
   
     private String classNameToPath(String className) { 
         return rootDir + File.separatorChar 
                 + className.replace('.', File.separatorChar) + ".class"; 
     } 
  }
  ```

  > 本文学习参考自
  >
  > [1] https://www.ibm.com/developerworks/cn/java/j-lo-classloader/index.html#minor1.1
  >
  > [2] https://blog.csdn.net/javazejian/article/details/73413292
  >
  > [3] https://juejin.im/post/5e1aaf626fb9a0301d11ac8e#heading-8


JVM总共包括3部分内容，类加载子系统、运行时数据区和字节码引擎，如下图所示：

![JVM运行时数据区](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/JVM运行时数据区.png)

接下来几篇文章主要会对类加载子系统、运行时数据区两部分内容介绍，而字节码执行引擎是比较底层的实现，会大概了解下它的相关指令。今天主要介绍下类装载子系统是如何执行类加载的。

### 类加载过程

多个java文件经过编译打包生成可运行jar包，最终由java命令运行某个主类的main函数启动程序，这里首先需要通过类加载器把主类加载到JVM。主类在运行过程中如果使用到其它类，会逐步加载这些类。注意，jar包里的类不是一次性全部加载的，是使用到时才加载。

类加载到使用整个过程有如下几步：

![JVM类加载过程](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/JVM类加载过程.png)

1. 加载：从jar、ear、war包中通过一个类的全限类名来获取定义此类的二进制字节流，将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构，在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口；

2. 验证：校验字节码文件的正确性，包括文件格式验证、元数据验证、字节码验证、符号引用验证等；
3. 准备：给类的静态变量分配内存，并赋予默认值；
4. 解析：将符号引用替换为直接引用，该阶段会把一些静态方法(符号引用，比如main()方法)替换为指向数据所存内存的指针或句柄等(直接引用)，这是所谓的静态链接过程(类加载期间完成)，动态链接是在程序运行期间完成的将符号引用替换为直接引用；
5. 初始化：对类的静态变量初始化为指定的值，执行静态代码块。

### 类加载器

上面的类加载过程主要是通过类加载器来实现的，Java里有如下几种类加载器：

- 启动类加载器：负责加载支撑JVM运行的位于JRE的lib目录下的核心类库，比如rt.jar、charsets.jar等；
- 扩展类加载器：负责加载支撑JVM运行的位于JRE的lib目录下的ext扩展目录中的JAR类包；
- 应用程序加载器：负责加载ClassPath路径下的类包，主要就是加载你自己写的那些类；
- 自定义类加载器：负责加载用户自定义路径下的类包。

![类加载器](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/类加载器.png)

查看类加载器：

```java
public class TestJdkClassLoader {

    public static void main(String[] args) {
        System.out.println(String.class.getClassLoader());
        System.out.println(DESKeyFactory.class.getClassLoader().getClass().getName());
        System.out.println(TestJdkClassLoader.class.getClassLoader().getClass().getName());
        System.out.println(ClassLoader.getSystemClassLoader().getClass().getName());
    }
}
执行结果：
null // 启动类加载器是C++语言实现的
sun.misc.Launcher$ExtClassLoader
sun.misc.Launcher$AppClassLoader
sun.misc.Launcher$AppClassLoader
```

#### 自定义一个类加载器

自定义类加载器只需要继承java.lang.ClassLoader类，该类有两个核心方法，一个是loadClass(String,boolean)，实现了双亲委派机制，大体逻辑如下：

1. 首先，检查一下指定名称的类是否已经加载过，如果加载过了，就不需要再加载直接返回。
2. 如果此类没有加载过，那么，再判断一下是否有父加载器；如果有父加载器，则由父加载器加载（即调用parent.loadClass(name,false);）.或者是调用bootstrap类加载器来加载。
3. 如果父加载器及bootstrap类加载器都没有找到指定的类，那么调用当前类加载器的findClass方法来完成类加载。

还有一个方法是findClass，默认实现是抛出异常，所以我们自定义类加载器主要是重写findClass方法。

```java
public class MyClassLoader extends ClassLoader {
		/**
		 * 指定加载路径
		 */
    private String classPath;

    public String getClassPath() {
        return classPath;
    }

    public void setClassPath(String classPath) {
        this.classPath = classPath;
    }

    private byte[] loadByte(String name) throws IOException {
        name = name.replaceAll("\\.", "/");
        FileInputStream fileInputStream = new FileInputStream(classPath + "/" + name + ".class");
        int len = fileInputStream.available();
        byte[] data = new byte[len];
        fileInputStream.read(data);
        fileInputStream.close();
        return data;
    }
  	
     /**
     * 重写findClass 方法，在指定Class Path 下加载文件名为name的Class
     * @param name
     * @return
     * @throws ClassNotFoundException
     */
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            byte[] bytes = loadByte(name);
          	// 将字节数组转换为Class对象
            return defineClass(name, bytes, 0, bytes.length);
        } catch (Exception e) {
            e.printStackTrace();
            throw new ClassNotFoundException();
        }
    }
}
```

#### 双亲委派机制

一个类加载器收到类加载请求，先不会自己去尝试，而是将这个请求委派给父类去执行，只有当父类无法完成加载时，才由自己完成。

**为什么要设计双亲委派机制？**

- 沙箱安全机制：自己写的java.lang.String.class类不会被加载，这样便可以防止核心API库被随意篡改；
- 避免类的重复加载：当父亲已经加载了该类时，就没有必要子ClassLoader再加载一次，保证被加载类的唯一性。

一个简单示例：

```java
package java.lang;

public class String {

    public static void main(String[] args) {
        System.out.println("---my string class----");
    }
}

运行结果：
错误: 在类 java.lang.String 中找不到 main 方法, 请将 main 方法定义为:
   public static void main(String[] args)
否则 JavaFX 应用程序类必须扩展javafx.application.Application
```

#### 打破双亲委派机制

##### 如何打破双亲委派

使用自定义类加载器重写loadClass方法，打破双亲委派机制。

```java
    /**
     * 重写loadClass方法，实现自己的加载逻辑，不再使用委派机制
     * @param name
     * @param resolve
     * @return
     * @throws ClassNotFoundException
     */
    @Override
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        synchronized (getClassLoadingLock(name)) {
            Class<?> loadedClass = findLoadedClass(name);
            if (loadedClass == null) {
                long nanoTime = System.nanoTime();
                loadedClass = findClass(name);
                PerfCounter.getFindClassTime().addElapsedTimeFrom(nanoTime);
                PerfCounter.getFindClasses().increment();
            }
            if (resolve) {
                resolveClass(loadedClass);
            }
        }
        return super.loadClass(name, resolve);
    }
```

##### Tomcat打破双亲委派

Tomcat作为一个web容器，那么它要解决的问题如下：

1. 一个web容器可能需要部署两个应用程序，不同的应用程序可能会依赖同一个第三方类库的不同版本，不能要求同一个类库在同一个服务器只有一份，因此要保证每个应用程序的类库是独立的，保证相互隔离。
2. 部署在同一个web容器中相同的类库相同的版本可以共享。否则，如果服务器有10个应用程序，那么要有10份相同的类加载进虚拟机。
3. web容器也有自己依赖的类库，不能与应用程序的类库混淆。基于安全考虑，应该让容器的类库和程序的类库隔离开来。
4. web容器要支持jsp的修改，jsp文件最终也要编译成class文件才能在虚拟机中运行，但程序运行后修改jsp文件是很常见的，web容器需要支持jsp修改后不需要重启。

那么**Tomcat为什么需要打破类加载机制**？

第一个问题，如果使用默认的类加载器机制，那么是无法加载两个相同类库的不同版本的，默认的类加器是不管你是什么版本的，只在乎你的全限定类名，并且只有一份。

第二个问题，默认的类加载器是能够实现的，因为他的职责就是保证唯一性。

第三个问题和第一个问题一样。

第四个问题，我们想我们要怎么实现jsp文件的热加载，jsp文件其实也就是class文件，那么如果修改了，但类名还是一样，类加载器会直接取方法区中已经存在的，修改后的jsp是不会重新加载的。那么怎么办呢？我们可以直接卸载掉这jsp文件的类加载器，所以你应该想到了，每个jsp文件对应一个唯一的类加载器，当一个jsp文件修改了，就直接卸载这个jsp类加载器。重新创建类加载器，重新加载jsp文件。

![Tomcat类加载图](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/Tomcat类加载图.png)

CommonClassLoader能加载的类都可以被CatalinaClassLoader和SharedClassLoader使用，从而实现了公有类库的共用，而CatalinaClassLoader和SharedClassLoader自己能加载的类则与对方相互隔离。WebAppClassLoader可以使用SharedClassLoader加载到的类，但**各个WebAppClassLoader实例之间相互隔离**。而JasperLoader的加载范围仅仅是这个JSP文件所编译出来的那一个.Class文件，它出现的目的就是为了被丢弃：当Web容器检测到JSP文件被修改时，会替换掉目前的JasperLoader的实例，并通过再建立一个新的Jsp类加载器来实现JSP文件的热加载功能。


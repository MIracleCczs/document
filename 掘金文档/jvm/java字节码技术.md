### java程序是如何执行的
刚开始学习java的时候我们都写过HelloWorld.java，我们会执行javac HelloWorld.java生成字节码文件，再使用java HelloWorld就可以执行了。这个执行过程其实是虚拟机从文件系统中加载字节码文件到内存中，生成HelloWorld对象实例。在这个过程中主要涉及到字节码、类加载器、虚拟机这三个方面的技术。

![java字节码](/Users/miracle/miracle/chrome/java字节码.png)


### 从java字节码技术开始

#### 什么是字节码

Java bytecode由单字节的指令组成，理论上最多支持256个操作码。

根据指令的性质，主要分为四大类：

1. 栈操作指令，包括与局部变量交互的指令
2. 程序流程控制指令
3. 对象操作指令，包括方法调用指令
4. 算术运算以及类型转换指令

#### 生成字节码文件-javap命令

你可以使用javap命令直接生成字节码文件，如果你使用idea作为开发工具，也可以集成javap命令快捷操作。配置方式可以参考[IDEA javap命令工具配置](https://blog.csdn.net/kongfanyu/article/details/107734950)。


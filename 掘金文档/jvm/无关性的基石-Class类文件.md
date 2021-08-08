Java诞生之初就宣称的“一次编译，处处运行”的性质一直是Java的一大特点，而Java实现这个特点的方式是将`.java`文件编译成`.class`文件，通过JVM屏蔽系统差异实现的。事实上，不仅是Java，其他的比如Kotlin、Groovy等语言也都可以通过编译成字节码文件运行在Java虚拟机上，而Java虚拟机也并不关心被编译成字节码文件之前是什么语言。

![Java虚拟机提供的语言无关性](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/Java虚拟机提供的语言无关性.png)

### Class类文件的结构-枯燥的基础知识

Java技术能够一直保持着非常良好的向后兼容性，Class文件结构的稳定功不可没，任何一门程序 语言能够获得商业上的成功，都不可能去做升级版本后，旧版本编译的产品就不再能够运行这种事 情。本章所讲述的关于Class文件结构的内容，绝大部分都是在第一版中就已经定义好的，内容虽然古老，但时至今日，Java发展 经历了十余个大版本、无数小更新，那时定义的Class文件格式的各项细节几乎没有出现任何改变。尽管不同版本的《Java虚拟机规范》对Class文件格式进行了几次更新，但基本上只是在原有结构基础上 新增内容、扩充功能，并未对已定义的内容做出修改。我们先来看下class文件格式：

| 类型 | 名称 | 含义 |
| ---- | ---- | ---- |
| u4   |   magic   |    文件的标志：魔数  |
| u2     |   minor_version   | Class的次版本号 |
| u2     |    major_version  |   Class的主版本号   |
| u2     |    constant_pool_count  |   常量池的数量   |
| cp_info     |   constant_pool[constant_pool_count-1]   |   常量池   |
| u2     |   access_flags   |   Class 的访问标志   |
| u2     |   this_class   |   当前类   |
|  u2    |    super_class  |  父类    |
|   u2   |   interfaces_count   |   接口   |
|   u2   |   interfaces[interfaces_count]   |  一个类可以实现多个接口    |
|   u2   |   fields_count   |  Class 文件的字段属性    |
|    field_info  |   fields[fields_count]   |   一个类会可以有个字段   |
|  u2    |   methods_count   |   Class 文件的方法数量   |
|   method_info   |   methods[methods_count]   |  一个类可以有个多个方法   |
|   u2   |   attributes_count   |   此类的属性表中的属性数   |
|  attribute_info    |    attributes[attributes_count]  |   属性表集合   |

以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个 字节的无符号数，无符号数可以用来描述数字、索引引用、数量值或者按照UTF-8编码构成字符串。表是由多个无符号数或者其他表作为数据项构成的复合数据类型，为了便于区分，所有表的命名 都习惯性地以“_info”结尾

![Class类文件结构](/Users/miracle/miracle/githubdoc/document/掘金文档/jvm/Class类文件结构.png)

#### 魔数

每个Class文件的头4个字节被称为魔数（Magic Number），它的唯一作用是确定这个文件是否为 一个能被虚拟机接受的Class文件。不仅是Class文件，很多文件格式标准中都有使用魔数来进行身份识 别的习惯，Class文件的魔数取得很有“浪漫气息”， 值为0xCAFEBABE（咖啡宝贝？）。

#### 文件版本号

紧接着魔数的4个字节存储的是Class文件的版本号：第5和第6个字节是次版本号（Minor Version），第7和第8个字节是主版本号（Major Version）。

#### 常量池

常量池中主要存放两大类常量：字面量（Literal）和符号引用（Symbolic References）。字面量比 较接近于Java语言层面的常量概念，如文本字符串、被声明为final的常量值等。而符号引用则属于编译 原理方面的概念，主要包括下面几类常量：

- 被模块导出或者开放的包（Package） 
- 类和接口的全限定名（Fully Qualified Name） 
- 字段的名称和描述符（Descriptor） 
- 方法的名称和描述符 
- 方法句柄和方法类型（Method Handle、Method Type、Invoke Dynamic） 
- 动态调用点和动态常量（Dynamically-Computed Call Site、Dynamically-Computed Constant）

常量池中每一项常量都是一个表，最初常量表中共有11种结构各不相同的表结构数据，后来为了 更好地支持动态语言调用，额外增加了4种动态语言相关的常量[1]，为了支持Java模块化系统 （Jigsaw），又加入了CONSTANT_Module_info和CONSTANT_Package_info两个常量，所以截至JDK 13，常量表中分别有17种不同类型的常量。

#### 访问标志

这个标志用于识别一些类或 者接口层次的访问信息，包括：这个Class是类还是接口；是否定义为public类型；是否定义为abstract 类型；如果是类的话，是否被声明为final；等等。

#### 类索引、父类索引与接口索引集合

类索引（this_class）和父类索引（super_class）都是一个u2类型的数据，而接口索引集合 （interfaces）是一组u2类型的数据的集合，Class文件中由这三项数据来确定该类型的继承关系。类索引用于确定这个类的全限定名，父类索引用于确定这个类的父类的全限定名。

#### 字段表集合

字段表用来描述类/接口中的成员变量，不是描述方法中的局部变量。下面是field_info的结构：

```c++
field_info {
  u2 access_flag;
  u2 name_index;
  u2 descriptor_index;
  attribute_info attributes[attributes_count];
}
```

字段修饰符放在access_flags项目中，它与类中的access_flags项目是非常类似的，都是一个u2的数据类型。name_index是对常量池的引用，用于描述字段的名称。descriptor_index也是对常量池的引用，是字段和方法的描述符。

#### 方法表集合

Class文件存储 格式中对方法的描述与对字段的描述采用了几乎完全一致的方式，方法表的结构如同字段表一样，依次包括访问标志（access_flags）、名称索引（name_index）、描述符索引（descriptor_index）、属性表 集合（attributes）几项。

#### 属性表集合

属性表（attribute_info）在前面的讲解之中已经出现过数次，Class文件、字段表、方法表都可以携带自己的属性表集合，以描述某些场景专有的信息。与Class文件中其他的数据项目要求严格的顺序、长度和内容不同，属性表集合的限制稍微宽松一些，不再要求各个属性表具有严格顺序，并且《Java虚拟机规范》允许只要不与已有属性名重复，任何人实现的编译器都可以向属性表中写入自己定义的属性信息，Java虚拟机运行时会忽略掉它不认识的属性。


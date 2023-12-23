---
title: JVM系列—虚拟机类加载机制
date: 2017-11-17 16:59:07
tags:
    - JVM
    - Java
categories: 基础原理
---

## 概述

### 什么是类加载

虚拟机把描述类的数据从 Class 文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的 Java 类型，这就是虚拟机的类加载机制。

<!-- more -->

### 为什么有类加载

Class 文件中所描述的信息，是使用字节码进行描述的，而这种描述必然是机器无法读取的，需要 JVM 在中间作用来将其转换成机器可以运行的信息。而且对于 Java 语言来说，类型的加载，连接和初始化过程都在程序运行期间完成，这虽然会令类加载时稍微增加一些性能上的开销，但是所带来的是 Java 高于 C 这类语言的灵活度，Java 里天生可以动态扩展的语言特性就是依赖运行期动态加载和动态连接这个特点实现的。

### 类加载发生在什么时候

#### 类加载的生命周期

类从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期包括：加载（Loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、初始化（Initialization）、使用（Using）和卸载（Unloading）7个阶段。其中验证、准备、解析3个部分统称为连接（Linking），这7个阶段的发生顺序如下图所示：

![类的生命周期](https://github.com/BingLau7/blog/blob/master/images/blog_33/1.png?raw=true)

加载、验证、准备、初始化和卸载这5个阶段的顺序是确定的，**类的加载过程必须按照这种顺序按部就班地开始，而解析阶段则不一定：它在某些情况下可以在初始化阶段之后再开始，这是为了支持Java语言的运行时绑定（也称为动态绑定或晚期绑定）**。注意，这里**笔者写的是按部就班地『开始』**，而不是按部就班地“进行”或“完成”，强调这点是因为这些阶段通常都是互相交叉地混合式进行的，通常会在一个阶段执行的过程中调用、激活另外一个阶段。

#### 类加载的时机

什么情况下需要开始类加载过程的第一个阶段加载？Java虚拟机规范中并没有进行强制约束，也就是说不同虚拟机上面实现可能是不一样的，这个由虚拟机的具体实现来自由把握。

#### 类初始化的时机

对于初始化阶段，虚拟机规范则是严格规定了**有且只有**5种情况必须立即对类进行“初始化”（而加载、验证、准备自然需要在此之前开始）。

##### 特定字节码指令

遇到 new、getstatic、putstatic 或 invokestatic 这 4 条字节码指令时，如果类没有进行过初始化，则需要先触发其初始化。生成这 4 条指令的最常见的 Java 代码场景是：使用 new 关键字实例化对象的时候、读取或设置一个类的静态字段（被 final 修饰、已在编译器把结果放入常量池的静态字段除外）的时候，以及调用一个类的静态方法的时候。

-  new: 创建一个对象，并将其引入值压入栈顶
-  getstatic: 获取指定类的静态域，并将其值压入栈顶
-  putstatic: 为指定的类的静态域赋值
-  invokestatic: 调用静态方法

##### 反射

使用 java.lang.reflect 包的方法对类进行反射调用的时候，如果类没有进行初始化，则需要先触发其初始化。

##### 初始化子类其父类未被初始化

当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。

##### 用户指定的主类

当虚拟机启动时，用户需要指定一个要执行的主类（包含 `main()` 方法的那个类），虚拟机会先初始化这个主类。

##### JDK1.7 动态语言支持

当使用 JDK 1.7 的动态语言支持时，如果一个 `java.lang.invoke.MethodHandle` 实例最后的解析结果是 `REF_getStatic`、`REF_putStatic`、`REF_invokeStatic` 的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。

#### 被动引用

上述 5 种场景中的行为称为对一个类进行主动引用。除此之外，所有引用类的方式都不会触发初始化，称为被动引用，接下来举三个被动引用的例子。

```Java
package io.github.binglau.jvm.passive_reference;

class SuperClass {
    static {
        System.out.println("SuperClass init!");
    }
}

class SubClass extends SuperClass {
    static {
        System.out.println("SubClass init!");
    }
}

class ConstClass {
    static {
        System.out.println("ConstClass init!");
    }

    public static final String HELLOWORLD = "hello world";
}

public class NotInitialization {

    public static void main(String[] args) {
        // 通过子类引用父类的静态字段，不会导致子类初始化
      	// 只会输出 SuperClass init! 而不会输出 SubClass init!
        System.out.println(SubClass.value); // 1
        // 通过数组定义来引用类，不会触发此类的初始化
      	// 没有输出
        SuperClass[] sca = new SuperClass[10]; // 2
        // 常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，
        // 因此不会触发定义常量的类的初始化
      	// 因为在编译阶段通过常量传播优化，已经将此常量值 "hello world" 存储到 
      	// NotInitialization 类的常量池中，以后 NotInitialization 对常量
        // ConstClass.HELLOWORLD 的引用都被转化为 NotInitialization 类对自身常量池的引用
        System.out.println(ConstClass.HELLOWORLD); // 3
    }
}

/**
结果：
# 1 输出结果
SuperClass init!
123
# 3 输出结果
hello world

**/
```

>  扩展：至于是否要触发子类的加载和验证，在虚拟机规范中并未明确规定，这点取决于虚拟机的具体实现。使用Sun HotSpot虚拟机通过-XX:+TraceClassLoading参数可观察到此操作会导致子类的加载（但未初始化）。
>
>  ```
>  ....
>  [Loaded NotInitialization from file:/Users/binglau/code/cradle/Java-demo/basic/src/main/java/io/github/binglau/jvm/passive_reference/]
>  [Loaded sun.launcher.LauncherHelper$FXHelper from /Library/Java/JavaVirtualMachines/jdk8/Contents/Home/jre/lib/rt.jar]
>  [Loaded java.lang.Class$MethodArray from /Library/Java/JavaVirtualMachines/jdk8/Contents/Home/jre/lib/rt.jar]
>  [Loaded java.lang.Void from /Library/Java/JavaVirtualMachines/jdk8/Contents/Home/jre/lib/rt.jar]
>  [Loaded SuperClass from file:/Users/binglau/code/cradle/Java-demo/basic/src/main/java/io/github/binglau/jvm/passive_reference/]
>  [Loaded SubClass from file:/Users/binglau/code/cradle/Java-demo/basic/src/main/java/io/github/binglau/jvm/passive_reference/]
>  SuperClass init
>  123
>  hello world
>  [Loaded java.lang.Shutdown from /Library/Java/JavaVirtualMachines/jdk8/Contents/Home/jre/lib/rt.jar]
>  [Loaded java.lang.Shutdown$Lock from /Library/Java/JavaVirtualMachines/jdk8/Contents/Home/jre/lib/rt.jar]
>  ```
>
>  上述输入皆没有加载相应的类



## 类加载过程

### 加载

需要做的事情：

1. 通过一个类的全限定名来获取定义此类的二进制字节流。
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。

其规定并非具体，而是根据各个虚拟机实现来定义的，比如『通过一个类的全限定名来获取定义此类的二进制字节流』这条，并没有指定非要从一个 class 文件中获取二进制字节流，有很多别样的玩法，比如：

-  从ZIP包中读取，这很常见，最终成为日后JAR、EAR、WAR格式的基础。
-  从网络中获取，这种场景最典型的应用就是Applet。
-  运行时计算生成，这种场景使用得最多的就是动态代理技术，在java.lang.reflect.Proxy中，就是用了ProxyGenerator.generateProxyClass来为特定接口生成形式为“*$Proxy”的代理类的二进制字节流。
-  由其他文件生成，典型场景是JSP应用，即由JSP文件生成对应的Class类。
-  从数据库中读取，这种场景相对少见些，例如有些中间件服务器（如SAP Netweaver）可以选择把程序安装到数据库中来完成程序代码在集群间的分发。
-  ….

#### 加载类与数组

相对于类加载过程的其他阶段，一个非数组类的加载阶段（准确地说，是加载阶段中获取类的二进制字节流的动作）是开发人员可控性最强的，因为加载阶段既可以使用系统提供的引导类加载器来完成，也可以由用户自定义的类加载器去完成，开发人员可以通过定义自己的类加载器去控制字节流的获取方式（即重写一个类加载器的`loadClass()`方法）。

对于数组类而言，情况就有所不同，**数组类本身不通过类加载器创建，它是由Java虚拟机直接创建的**。但数组类与类加载器仍然有很密切的关系，因为数组类的元素类型（Element Type，指的是数组去掉所有维度的类型）最终是要靠类加载器去创建，一个数组类（下面简称为C）创建过程就遵循以下规则：

-  如果数组的组件类型（Component Type，指的是数组去掉一个维度的类型）是引用类型，那就递归采用本节中定义的加载过程去加载这个组件类型，数组 C 将在加载该组件类型的类加载器的类名称空间上被标识（这点很重要，后面会介，一个类必须与类加载器一起确定唯一性）。
-  如果数组的组件类型不是引用类型（例如int［］数组），Java虚拟机将会把数组 C 标记为与引导类加载器关联。
-  数组类的可见性与它的组件类型的可见性一致，如果组件类型不是引用类型，那数组类的可见性将默认为public。

### 验证

目的是为了确保 Class 文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。

验证阶段大致上会完成下面 4 个阶段的验证工作：

#### 文件格式验证

本阶段要验证字节流是否符合 Class 文件格式的规范，是否能被当前版本的虚拟机处理。主要目的是保证输入的字节流能正确地解析并存储于方法区之内，格式上符合描述一个 Java 类型信息的要求。本阶段的验证是基于二进制字节流进行的，只有通过了这个阶段的验证后，字节流才会进入内存的方法区中进行存储，所以后面的 3 个验证阶段全部是基于方法区的存储结构进行的，不会再直接操作字节流。

包括但不限于：

-  是否以魔数0xCAFEBABE开头。
-  主、次版本号是否在当前虚拟机处理范围之内。
-  常量池的常量中是否有不被支持的常量类型（检查常量tag标志）。
-  指向常量的各种索引值中是否有指向不存在的常量或不符合类型的常量。
-  `CONSTANT_Utf8_info` 型的常量中是否有不符合UTF8编码的数据。
-  Class 文件中各个部分及文件本身是否有被删除的或附加的其他信息。

#### 元数据验证

 本阶段是对字节码描述的信息进行语义分析，以保证其描述的信息符合Java语言规范的要求。主要目的是对类的元数据信息进行语义校验，保证不存在不符合Java语言规范的元数据信息。

可能包括但不限于：

-  这个类是否有父类（除了java.lang.Object之外，所有的类都应当有父类）。
-  这个类的父类是否继承了不允许被继承的类（被final修饰的类）。
-  如果这个类不是抽象类，是否实现了其父类或接口之中要求实现的所有方法。
-  类中的字段、方法是否与父类产生矛盾（例如覆盖了父类的final字段，或者出现不符合规则的方法重载，例如方法参数都一致，但返回值类型却不同等）。

#### 字节码验证

本阶段是整个验证过程中最复杂的一个阶段，**主要目的是通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的**。在元数据验证阶段对元数据信息中的数据类型做完校验后，本阶段将对类的方法体进行校验分析，保证被校验类的方法在运行时不会做出危害虚拟机安全的事件。例如：

-  保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作。不会出现类似这样的情况：在操作栈放置了一个int类型的数据，使用时却按long类型来加载入本地变量表中。
-  保证跳转指令不会跳转到方法体以外的字节码指令上。
-  保证方法体中的类型转换是有效的。可以把一个子类对象赋值给父类数据类型，这是安全的，但是把父类对象赋值给子类数据类型，甚至把对象赋值给与它毫无继承关系、完全不相干的一个数据类型，则是危险和不合法的。

别相信机器给你纠错，不然他就先给自己纠错了。

#### 符合引用验证

本阶段发生在虚拟机将符号引用转化为直接引用的时候，这个转化动作将在连接的第三阶段——解析阶段中发生。符号引用验证可以看做是对类自身以外（常量池中的各种符号引用）的信息进行匹配性校验，通常需要校验下列内容：

-  符号引用中通过字符串描述的全限定名是否能找到对应的类。
-  在指定类中是否存在符合方法的字段描述符以及简单名称所描述的方法和字段。
-  符号引用中的类、字段、方法的访问性（private、protected、public、default）是否可被当前类访问。

符号引用验证的目的是确保解析动作能正常执行，如果无法通过符号引用验证，那么将会抛出一个`java.lang.IncompatibleClassChangeError`异常的子类，如`java.lang.IllegalAccessError`、`java.lang.NoSuchFieldError`、`java.lang.NoSuchMethodError`等。

对于虚拟机的类加载机制来说，验证阶段是一个非常重要的、但不是一定必要（因为对程序运行期没有影响）的阶段。如果所运行的全部代码（包括自己编写的及第三方包中的代码）都已经被反复使用和验证过，那么在实施阶段就可以考虑使用`-Xverify:none`参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。

### 准备

准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些变量所使用的内存都将在方法区中进行分配。

 本阶段中有两个容易产生混淆的概念需要强调一下：

-  此时进行内存分配的仅包括类变量（被static修饰的变量），而不包括实例变量，实例变量将会在对象实例化时随着对象一起分配在Java堆中。
-  这里所说的初始值“通常情况”下是数据类型的零值。

#### 通常情况

`public static int value = 123;` 这里的 value 在准备之后的值是 0 而不是 123

#### 特殊情况

`public static final int value = 123;` 

编译时Javac将会为value生成ConstantValue属性，在准备阶段虚拟机就会根据ConstantValue的设置将value赋值为123。

### 解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程，符号引用在前一章讲解Class文件格式的时候已经出现过多次，在 Class 文件中它以 `CONSTANT_Class_info`、`CONSTANT_Fieldref_info`、`CONSTANT_Methodref_info` 等类型的常量出现。那解析阶段中所说的直接引用与符号引用又有什么关联呢？

-  符号引用（Symbolic References）：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。**符号引用与虚拟机实现的内存布局无关，引用的目标并不一定已经加载到内存中**。各种虚拟机实现的内存布局可以各不相同，但是它们能接受的符号引用必须都是一致的，因为符号引用的字面量形式明确定义在Java虚拟机规范的Class文件格式中。
-  直接引用（Direct References）：直接引用可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。**直接引用是和虚拟机实现的内存布局相关的**，同一个符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同。如果有了直接引用，那引用的目标必定已经在内存中存在。

虚拟机规范之中并未规定解析阶段发生的具体时间，只要求了在执行`anewarray`、`checkcast`、`getfield`、`getstatic`、`instanceof`、`invokedynamic`、`invokeinterface`、`invokespecial`、`invokestatic`、`invokevirtual`、`ldc`、`ldc_w`、`multianewarray`、`new`、`putfield`和 `putstatic` 这16个用于操作符号引用的字节码指令之前，先对它们所使用的符号引用进行解析。

#### 符号解析中的差异

对同一个符号引用进行多次解析请求是很常见的事情，**除`invokedynamic`指令以外，虚拟机实现可以对第一次解析的结果进行缓存（在运行时常量池中记录直接引用，并把常量标识为已解析状态）从而避免解析动作重复进行。**无论是否真正执行了多次解析动作，虚拟机需要保证的是在同一个实体中，如果一个符号引用之前已经被成功解析过，那么后续的引用解析请求就应当一直成功；同样的，如果第一次解析失败了，那么其他指令对这个符号的解析请求也应该收到相同的异常。

对于`invokedynamic`指令，上面规则则不成立。当碰到某个前面已经由`invokedynamic`指令触发过解析的符号引用时，并不意味着这个解析结果对于其他`invokedynamic`指令也同样生效。因为`invokedynamic`指令的目的本来就是用于动态语言支持（目前仅使用Java语言不会生成这条字节码指令），它所对应的引用称为『动态调用点限定符』（Dynamic Call Site Specifier），这里『动态』的含义就是必须等到程序实际运行到这条指令的时候，解析动作才能进行。相对的，其余可触发解析的指令都是“静态”的，可以在刚刚完成加载阶段，还没有开始执行代码时就进行解析。

#### 解析对象

解析动作主要针对**类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符**7类符号引用进行，分别对应于常量池的 `CONSTANT_Class_info`、`CONSTANT_Field-ref_info`、`CONSTANT_Methodref_info`、`CONSTANT_InterfaceMethodref_info`、`CONSTANT_MethodType_info`、`CONSTANT_MethodHandle_info`和 `CONSTANT_InvokeDynamic_info` 7种常量类型

#### 类或接口的解析

假设当前代码所处的类为D，如果要把一个从未解析过的符号引用N解析为一个类或接口C的直接引用，那虚拟机完成整个解析的过程需要以下3个步骤：

1. **如果C不是一个数组类型，那虚拟机将会把代表N的全限定名传递给D的类加载器去加载这个类C。**在加载过程中，由于元数据验证、字节码验证的需要，又可能触发其他相关类的加载动作，例如加载这个类的父类或实现的接口。一旦这个加载过程出现了任何异常，解析过程就宣告失败。
2. **如果C是一个数组类型，并且数组的元素类型为对象，也就是N的描述符会是类似“[Ljava/lang/Integer”的形式，那将会按照第1点的规则加载数组元素类型**。如果N的描述符如前面所假设的形式，需要加载的元素类型就是“java.lang.Integer”，接着由虚拟机生成一个代表此数组维度和元素的数组对象。
3. **如果上面的步骤没有出现任何异常，那么C在虚拟机中实际上已经成为一个有效的类或接口了，但在解析完成之前还要进行符号引用验证，确认D是否具备对C的访问权限。**如果发现不具备访问权限，将抛出java.lang.IllegalAccessError异常。

#### 字段解析

要解析一个未被解析过的字段符号引用，首先将会对字段表内 `class_index` 项中索引的`CONSTANT_Class_info`符号引用进行解析，也就是字段所属的类或接口的符号引用。如果在解析这个类或接口符号引用的过程中出现了任何异常，都会导致字段符号引用解析的失败。**如果解析成功完成，那将这个字段所属的类或接口用C表示，虚拟机规范要求按照如下步骤对C进行后续字段的搜索**。

1. 如果C本身就包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束（**类本身字段描述**）
2. 否则，如果在C中实现了接口，将会按照继承关系从下往上递归搜索各个接口和它的父接口，如果接口中包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。（**接口及接口继承**）
3. 否则，如果C不是java.lang.Object的话，将会按照继承关系从下往上递归搜索其父类，如果在父类中包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。（**类继承**）
4. 否则，查找失败，抛出java.lang.NoSuchFieldError异常。
5. 如果查找过程成功返回了引用，将会对这个字段进行权限验证，如果发现不具备对字段的访问权限，将抛出java.lang.IllegalAccessError异常。

#### 类方法解析

类方法解析的第一个步骤与字段解析一样，也需要先解析出类方法表的`class_index`项中索引的方法所属的类或接口的符号引用，如果解析成功，我们依然用C表示这个类，接下来虚拟机将会按照如下步骤进行后续的类方法搜索。

1. 类方法和接口方法符号引用的常量类型定义是分开的，如果在类方法表中发现class_index中索引的C是个接口，那就直接抛出java.lang.IncompatibleClassChangeError异常。（**类本身方法**）
2. 如果通过了第1步，在类C中查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。
3. 否则，在类C的父类中递归查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。（**类继承父类的方法**）
4. 否则，在类C实现的接口列表及它们的父接口之中递归查找是否有简单名称和描述符都与目标相匹配的方法，如果存在匹配的方法，说明类C是一个抽象类，这时查找结束，抛出`java.lang.AbstractMethodError`异常。(**抽象类查找抽象方法**)
5. 否则，宣告方法查找失败，抛出`java.lang.NoSuchMethodError`。
6. 如果查找过程成功返回了直接引用，将会对这个方法进行权限验证，如果发现不具备对此方法的访问权限，将抛出`java.lang.IllegalAccessErro`r异常。

#### 接口方法解析

接口方法也需要先解析出接口方法表的`class_index`项中索引的方法所属的类或接口的符号引用，如果解析成功，依然用C表示这个接口，接下来虚拟机将会按照如下步骤进行后续的接口方法搜索。

1. 与类方法解析不同，如果在接口方法表中发现`class_index`中的索引C是个类而不是接口，那就直接抛出`java.lang.IncompatibleClassChangeError`异常。(**接口确认**)
2. 否则，在接口C中查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。（**查找接口本身方法**）
3. 否则，在接口C的父接口中递归查找，直到`java.lang.Object`类（查找范围会包括Object类）为止，看是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。（**查找继承接口方法**）
4. 否则，宣告方法查找失败，抛出`java.lang.NoSuchMethodError`异常。

由于接口中的所有方法默认都是public的，所以不存在访问权限的问题，因此接口方法的符号解析应当不会抛出`java.lang.IllegalAccessError`异常。

### 初始化

初始化之前除了用户可以自定义类加载器参与之前都是由 JVM 主导和控制，到了初始化阶段，才真正开始执行类中定义的 Java 程序代码（或者说是字节码）

在准备阶段，类变量已经赋过一次系统要求的初始值。在初始化阶段，则根据程序员通过程序制定的主观计划去初始化类变量和其他资源，或者可以从另外一个角度来表达：**初始化阶段是执行类构造器＜clinit＞()方法的过程。**

#### `<client>()` 方法

`＜clinit＞()`方法是由编译器自动收集**类中的所有类变量的赋值动作和静态语句块**（`static{}`块）中的语句合并产生。**编译器收集的顺序由语句在源文件中出现的顺序所决定**。**静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问**，示例代码如下所示。

```Java
package io.github.binglau.jvm;


public class TestClassinitialization {
    static {
        i = 0;//给变量赋值可以正常编译通过
        System.out.print(i);//这句编译器会提示"非法向前引用"
    }

    static int i = 1;
}
```

#### `<clinit>()`执行特定和细节

这里只限于 Java 语言编译产生的 Class 文件，并不包括其他 JVM 语言。

-  `<clinit>()`方法与类的构造函数（或者说实例构造器`<init>()`方法）不同，它不需要显式地调用父类构造器，**虚拟机会保证在子类的`<clinit>()`方法执行之前，父类的`<clinit>()`方法已经执行完毕**。因此在虚拟机中第一个被执行的`<clinit>()`方法的类肯定是java.lang.Object。

-  `<clinit>()`方法对于类或接口来说并**不是必需的**，如果一个类中没有静态语句块，也没有对变量的赋值操作，那么编译器可以不为这个类生成`<clinit>()`方法。

-  接口中不能使用静态语句块，但仍然有变量初始化的赋值操作，因此**接口与类一样都会生成`<clinit>()`方法。但接口与类不同的是，执行接口的`<clinit>()`方法不需要先执行父接口的`<clinit>()`方法。只有当父接口中定义的变量使用时，父接口才会初始化。**另外，接口的实现类在初始化时也一样不会执行接口的`<clinit>()`方法。

-  **虚拟机会保证一个类的`<clinit>()`方法在多线程环境中被正确地加锁、同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的`<clinit>()`方法，其他线程都需要阻塞等待，直到活动线程执行`<clinit>()`方法完毕。如果在一个类的`<clinit>()`方法中有耗时很长的操作，就可能造成多个进程阻塞（被唤醒之后不会再次进入`<clinit>()`方法，同一个类加载器下一个类型只会初始化一次），在实际应用中这种阻塞往往是很隐蔽的。**

   ```Java
   package io.github.binglau.jvm;

   /**
    * 文件描述:
    */

   public class TestClassinitialization {
       static class DeadLoopClass {
           static {
               /*如果不加上这个if语句，编译器将提示"Initializer does not complete normally"并拒绝编译*/
               if (true) {
                   System.out.println(Thread.currentThread() + "init DeadLoopClass");
                   while (true) {
                   }
               }
           }
       }

       public static void main(String[] args) {
           Runnable script = new Runnable() {
               public void run() {
                   System.out.println(Thread.currentThread() + "start");
                   DeadLoopClass dlc = new DeadLoopClass();
                   System.out.println(Thread.currentThread() + "run over");
               }
           };
           Thread thread1 = new Thread(script);
           Thread thread2 = new Thread(script);
           thread1.start();
           thread2.start();
       }
   }

   /**
   运行结果如下，即一条线程在死循环以模拟长时间操作，另外一条线程在阻塞等待
   Thread[Thread-1,5,main]start
   Thread[Thread-0,5,main]start
   Thread[Thread-1,5,main]init DeadLoopClass
   **/
   ```

   ​

## 参考书籍

《深入理解 Java 虚拟机》

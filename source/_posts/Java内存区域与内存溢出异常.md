---
title: Java内存区域与内存溢出异常
date: 2017-10-21 16:55:36
tags:
    - JVM
    - Java
categories: 基础原理
---

### 图示各区用途

![内存区域图](https://github.com/BingLau7/blog/blob/master/images/blog_27/1.png?raw=true)

<!-- more -->

#### 程序计数器

可看作**当前线程**所执行的字节码的行号指示器。在 JVM 概念模型中（仅概念模型，各实现可能不一致），字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程回复等基础功能都需要依赖这个计数器完成。

Java 方法中：正在执行的虚拟机字节码指令的地址

Native 方法中：空（Undefined）



#### Java 虚拟机栈

线程私有，生命周期与线程相同。

虚拟机栈描述的是 Java 方法执行的内存模型：每个方法在执行的同时都会创建一个栈帧（Stack Frame） 用于存储局部变量表（粗浅流行划分中的栈）、操作数栈、动态链接、方法出口等信息。每一个方法从调用直到执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。

局部变量表：存放编译期可知的各种基本类型、对象引用（不等同于对象本身，可能指向一个对象起始地址的引用指针， 也可能是指向一个代表对象的句柄或其他与此对象相关的位置） 和 returnAddress 类型（指向了一条字节码指令的地址）。



#### 本地方法栈

为虚拟机使用到的 Native 方法服务。



#### Java 堆

被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例（The heap is the run-time data area from which memory for all class instances and arrays is allocated.）都在这里分配内存。

但随着 JIT 编译器的发展与逃逸分析技术逐渐成熟，栈上分配、标量替换优化技术将会导致一些微妙的变化，使之不『绝对』了。

Java 堆是 GC 管理的主要区域。从内存回收角度来看，可以将其分为『新生代』（Eden 空间、From Survivor、To Survivor）和『老年代』。从内存分配的角度来看，线程共享的 Java 堆中可能划分出多个线程私有的分配缓冲区（Thread Local Allocation Buffer，TLAB）。

Java 堆只需要保证逻辑上连续即可。

通过 `-Xmx` (最大堆大小)和 `-Xms` (初始堆大小)控制大小。



#### 方法区

各个线程共享的内存区域，它用于存储**已被虚拟机加载的类信息、常量、静态变量、JIT 编译后的代码等**数据。

被很多人作为『永久代』对待（其实并不是），因为仅仅是之前的 HotSpot 设计团队将其 GC 分代收集扩展至方法区，或者说是使用永久代实现方法区而已。

JVM 规范对方法区的限制非常宽松，除了和 Java 堆一样不需要连续的内存和可以选择固定大小和可扩展外，还可以选择不实现 GC（与此对应，如果实现了则内存回收主要目标是**针对常量池的回收和对类型的卸载**）。



#### 运行时常量池

方法区的一部分。Class 文件除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池（Constant Pool Table），用于**存放编译器生成的各种字面量和符号引用**，这部分内容将在类加载后进行方法区的运行时常量池中存放。**（这个不是运行时常量池）**

运行时长期是区别于上面的 Class 常量池：

1. 保存 Class 文件中描述的符号引用外，还会把翻译出来的直接引用也存储在运行时常量池中。

2. 相较 Class 文件中的常量池具备动态性，并非预置入 Class 文件中常量池的内容才能进入方法区，运行期间也可能将新的常量放入池中，这种特性被开发人员利用较多的就是 `String#intern()` 方法（[intern详细介绍](https://tech.meituan.com/in_depth_understanding_string_intern.html)）

   ​



#### 直接内存

不是运行时数据区的一部分，也不是 Java 虚拟机规范中定义的内存区域，但是这部番内存也被频繁使用，且可能会导致 OOM 的出现。

出现原因：1.4 加入 NIO，引入基于 Channel 和 Buffer 的 I/O 方式，可以使用 Native 函数库直接分配堆外内存，然后通过存在 Java 堆中的 `DirectByteBuffer` 对象作为这块内存的引用进行操作。（避免 Java 堆与 Native 堆复制数据）。



### 对象探秘

#### 对象创建

1. 检查指令是否在常量池中对应到一个类的符号引用，该符号是否被加载、解析和初始化，如果没有则需要执行相应类加载过程。

2. 为新生对象分配内存，等同于将一块确定大小的内存从 Java 堆中划分出来。使用方法：

   1. 指针碰撞：用过的内存放一边，空闲的内存放一边，中间加一个指针作为分界点的指示器，分配仅需将指针移动。
   2. 空闲列表：维护一个可用的内存块，从中分配。

   使用二者是看 Java 堆是否规整决定的。

   有一个问题需要考虑，创建对象过于频繁，即使只是修改指针位置也有可能出现线程不安全的情况。实际中 JVM 采用 CAS 配上失败重试的方法保证更新操作的原子性；另一种是把内存分配的动作按照线程划分在不同的空间进行，即每一个线程在 Java 堆中预先分配一小块内存，称为本地线程分配缓存（Thread Local Allocation Buffer，TLAB）。线程分配各自的 TLAB，只有 TLAB 用完并分配新的 TLAB 时，才需要同步锁定。使用 `-XX:+/-UseTLAB ` 参数来设定是否使用。

3. JVM 需要将分配到的内存空间都初始化为零值（不包括对象头），如果使用 TLAB，这一工作过程也可以提前至 TLAB 分配时进行。

4. 虚拟机要对对象进行必要的设置（对象是哪个类的实例，如何才能找到类的元信息，对象 hash code，对象 GC 分代年龄等信息）。存放于对象头(Object Header)中。

5. 执行 new 指令之后会接着执行 `<init>` 方法，将对象初始化。

#### 对象的内存布局

![内存布局](https://github.com/BingLau7/blog/blob/master/images/blog_27/2.png?raw=true)

##### 对象头：

-  存储对象自身的运行时数据，如哈希码、GC分代年龄等。长度在32位和64位的虚拟机中，分别为32bit、 64bit,官方称它为“Mark Word”


-  类型指针，对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。

注：如果对象是一个java数组，对象头中还必须有一块记录数据长度的数据。

##### 实例数据

对象真正存储的有用信息，也是程序中定义的各种类型的字段内容。

##### 对齐填充

由于HotSpot虚拟机要求对象的起始地址必须是8字节的整数倍，通俗的说，就是对象大小必须是8字节的整数倍。对象头正好是8字节的倍数。当实例数据部分没有对齐时，需要通过对齐填充来补全。

#### 对象的定位访问

建立对象是为了适用对象，Java通过栈上的引用来引用堆中的对象，引用通过何种方式去定位、访问堆中的对象的具体位置取决于虚拟机的实现。目前主流的访问方式有两种：使用句柄和直接指针。 

由于 HotSpot 是使用直接指针访问，所以这里指介绍直接指针：

如果是指针访问方式，java堆对象的布局中就必须考虑如何放置访问类型数据的相关信息，而reference中存储的直接就是对象地址。

![直接指针](https://github.com/BingLau7/blog/blob/master/images/blog_27/3.png?raw=true)

优点：速度相对更快。

#### 关于一个程序的创建过程

```Java
package io.github.binglau.jvm;

/**
 * 文件描述:
 */

public class ClassCreateProcessTest {
    public static B b = new B();
    private C c = new C();

    static {
        System.out.println("A created");
    }

    public ClassCreateProcessTest() {}

    public static void main(String[] args) {
        ClassCreateProcessTest a = new ClassCreateProcessTest();
        System.out.println(a);
    }
}

class B{
    static {
        System.out.println("B created");
    }
    //some fields.
}

class C {
    static {
        System.out.println("C created");
    }
    //some fields.
}

/**
B created
A created
C created
io.github.binglau.jvm.ClassCreateProcessTest@d716361
**/
```

```Java
binglau@BingLaudeMBP ~/c/c/J/basic> javap -verbose target/classes/io/github/binglau/jvm/ClassCreateProcessTest.class 
Classfile /Users/binglau/code/cradle/Java-demo/basic/target/classes/io/github/binglau/jvm/ClassCreateProcessTest.class
  Last modified 2017-10-17; size 938 bytes
  MD5 checksum 3197a5d18757ab40c96b935f44cba193
  Compiled from "ClassCreateProcessTest.java"
public class io.github.binglau.jvm.ClassCreateProcessTest
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #14.#34        // java/lang/Object."<init>":()V
   #2 = Class              #35            // io/github/binglau/jvm/C
   #3 = Methodref          #2.#34         // io/github/binglau/jvm/C."<init>":()V
   #4 = Fieldref           #5.#36         // io/github/binglau/jvm/ClassCreateProcessTest.c:Lio/github/binglau/jvm/C;
   #5 = Class              #37            // io/github/binglau/jvm/ClassCreateProcessTest
   #6 = Methodref          #5.#34         // io/github/binglau/jvm/ClassCreateProcessTest."<init>":()V
   #7 = Fieldref           #38.#39        // java/lang/System.out:Ljava/io/PrintStream;
   #8 = Methodref          #40.#41        // java/io/PrintStream.println:(Ljava/lang/Object;)V
   #9 = Class              #42            // io/github/binglau/jvm/B
  #10 = Methodref          #9.#34         // io/github/binglau/jvm/B."<init>":()V
  #11 = Fieldref           #5.#43         // io/github/binglau/jvm/ClassCreateProcessTest.b:Lio/github/binglau/jvm/B;
  #12 = String             #44            // A created
  #13 = Methodref          #40.#45        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #14 = Class              #46            // java/lang/Object
  #15 = Utf8               b
  #16 = Utf8               Lio/github/binglau/jvm/B;
  #17 = Utf8               c
  #18 = Utf8               Lio/github/binglau/jvm/C;
  #19 = Utf8               <init>
  #20 = Utf8               ()V
  #21 = Utf8               Code
  #22 = Utf8               LineNumberTable
  #23 = Utf8               LocalVariableTable
  #24 = Utf8               this
  #25 = Utf8               Lio/github/binglau/jvm/ClassCreateProcessTest;
  #26 = Utf8               main
  #27 = Utf8               ([Ljava/lang/String;)V
  #28 = Utf8               args
  #29 = Utf8               [Ljava/lang/String;
  #30 = Utf8               a
  #31 = Utf8               <clinit>
  #32 = Utf8               SourceFile
  #33 = Utf8               ClassCreateProcessTest.java
  #34 = NameAndType        #19:#20        // "<init>":()V
  #35 = Utf8               io/github/binglau/jvm/C
  #36 = NameAndType        #17:#18        // c:Lio/github/binglau/jvm/C;
  #37 = Utf8               io/github/binglau/jvm/ClassCreateProcessTest
  #38 = Class              #47            // java/lang/System
  #39 = NameAndType        #48:#49        // out:Ljava/io/PrintStream;
  #40 = Class              #50            // java/io/PrintStream
  #41 = NameAndType        #51:#52        // println:(Ljava/lang/Object;)V
  #42 = Utf8               io/github/binglau/jvm/B
  #43 = NameAndType        #15:#16        // b:Lio/github/binglau/jvm/B;
  #44 = Utf8               A created
  #45 = NameAndType        #51:#53        // println:(Ljava/lang/String;)V
  #46 = Utf8               java/lang/Object
  #47 = Utf8               java/lang/System
  #48 = Utf8               out
  #49 = Utf8               Ljava/io/PrintStream;
  #50 = Utf8               java/io/PrintStream
  #51 = Utf8               println
  #52 = Utf8               (Ljava/lang/Object;)V
  #53 = Utf8               (Ljava/lang/String;)V
{
  public io.github.binglau.jvm.ClassCreateProcessTest(); // 在实例创建出来的时候调用，包括调用new操作符；调用Class或Java.lang.reflect.Constructor对象的newInstance()方法；调用任何现有对象的clone()方法；通过java.io.ObjectInputStream类的getObject()方法反序列化。
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: new           #2                  // class io/github/binglau/jvm/C
         8: dup
         9: invokespecial #3                  // Method io/github/binglau/jvm/C."<init>":()V
        12: putfield      #4                  // Field c:Lio/github/binglau/jvm/C;
        15: return
      LineNumberTable:
        line 15: 0
        line 9: 4
        line 15: 15
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      16     0  this   Lio/github/binglau/jvm/ClassCreateProcessTest;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: new           #5                  // class io/github/binglau/jvm/ClassCreateProcessTest
         3: dup
         4: invokespecial #6                  // Method "<init>":()V
         7: astore_1
         8: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
        11: aload_1
        12: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
        15: return
      LineNumberTable:
        line 18: 0
        line 19: 8
        line 20: 15
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      16     0  args   [Ljava/lang/String;
            8       8     1     a   Lio/github/binglau/jvm/ClassCreateProcessTest;

  static {};      // 在jvm第一次加载class文件时调用，包括静态变量初始化语句和静态块的执行
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=2, locals=0, args_size=0
         0: new           #9                  // class io/github/binglau/jvm/B
         3: dup
         4: invokespecial #10                 // Method io/github/binglau/jvm/B."<init>":()V
         7: putstatic     #11                 // Field b:Lio/github/binglau/jvm/B;
        10: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
        13: ldc           #12                 // String A created
        15: invokevirtual #13                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        18: return
      LineNumberTable:
        line 8: 0
        line 12: 10
        line 13: 18
}
SourceFile: "ClassCreateProcessTest.java"
binglau@BingLaudeMBP ~/c/c/J/basic> 
```

static的方法其实是`<clinit>`方法，在 `<init>` 也就是实例化之前执行，所有 先 new B 然后这边调用  #7 并 ldc 也就是其中的 static {} 方法中的 sout 执行 打印出  String A created，此时还没有对 A 进行初始化，A 初始化时候创建了 C。

所以 B 是在 3 时候创建的，C 是在 5 创建的，A 是在 5之后创建完成的。



### 可能出现 StackOverflowError 处

#### 虚拟机（本地方法）栈中出现：

-  线程请求的栈深度大于虚拟机所允许的深度

```Java
/** 
 * 虚拟机栈和本地方法栈溢出
 * VM Args: -Xss128k 
 * @author Administrator 
 * 
 */  
public class JavaVMStackSOF {  
    private int stackLength = 1;  
    public void stackLeak() {  
        stackLength++;  
        stackLeak();  
    }  
    public static void main(String[] args) throws Throwable{  
        JavaVMStackSOF oom = new JavaVMStackSOF();  
        try {  
            oom.stackLeak();  
        } catch (Throwable e) {  
            System.out.println("stack length: " + oom.stackLength);  
            throw e;  
        }  
    }  
}  
```

### 可能出现 OutOfMemoryError 处

#### 虚拟机（本地方法）栈中出现：

-  如果虚拟机栈可以动态扩展，扩展时无法申请到足够的内存（实验不出来，如果是不断创建线程可以，但是这与栈空间十分足够大不存在关系，而是与操作系统对进程分配内存限制有关。）

#### 堆：

-  在堆中没有内存完成实例分配，并且堆也无法再扩展时

```Java
import java.util.ArrayList;  
import java.util.List;  
  
/** 
 * Java堆用于存储对象实例，我们只要不断创建对象，并且保证GC Roots到对象之间有可达路径来避免GC清除这些对象，就会在对象数量到达最大堆的容量限制后产生内存溢出异常。
 * VM Args: -Xms10m -Xmx10m -XX:+HeapDumpOnOutOfMemoryError 
 * @author Administrator 
 * 
 */  
public class HeapOOM {  
    static class OOMObject{  
        private String name;  
        public OOMObject(String name) {  
            this.name = name;  
        }  
    }  
    public static void main(String[] args) {  
        List<OOMObject> list = new ArrayList<OOMObject>();  
        long i = 1;  
        while(true) {  
            list.add(new OOMObject("IpConfig..." + i++));  
        }  
    }  
}  
```

#### 方法区 & 运行时常量池

-  无法满足内存分配需求

```Java
import java.util.ArrayList;  
import java.util.List;  
  
/** 
 * 方法区、运行时常量池溢出
 * 方法区用于存放Class的相关信息，如类名、访问修饰符、常量池、字段描述符、方法描述等。对于这个区域的测试，
 * 基本的思路是运行时产生大量的类去填满方法区，直到溢出。比如动态代理会生成动态类。
 * 运行时常量池分配在方法区内，可以通过 -XX:PermSize和 -XX:MaxPermSize限制方法区大小，从而间接限制其中常量池的容量。
 * VM Args: -XX:PermSize=10M -XX:MaxPermSize=10M 
 * @author Administrator 
 * 
 */  
public class RuntimeConstantPoolOOM {  
    public static void main(String[] args) {  
        // 使用List保持着常量池引用，避免Full GC回收常量池行为  
        List<String> list = new ArrayList<String>();  
        // 10MB的PermSize在integer范围内足够产生OOM了  
        int i = 0;  
        while (true) {  
            list.add(String.valueOf(i++).intern());  
        }  
    }  
}  
```

#### 直接内存：

-  本机内存限制，这个调整  `-Xmx` 也没用，直接内存动态扩展时出现

```Java
/** 
 * DirectMemory容量可以通过-XX:MaxDirectMemorySize指定，如果不指定，则默认与Java堆的最大值-Xmx指定一样。
 * VM Args: -Xmx20M -XX:MaxDirectMemorySize=10M 
 * @author Administrator 
 */  
public class DirectMemoryOOM {  
    private static final int _1MB = 1024 * 1024;  
    public static void main(String[] args) {  
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];  
        unsafeField.setAccessible(true);  
        Unsafe unsafe = (Unsafe) unsafeField.get(null);  
        while(true) {  
            unsafe.allocateMemory(_1MB);  
        }  
    }  
}  
```


#### OOM 时 dump 日志

`-XX:+HeapDumpOnOutOfMemoryError`

### 参考书目
《深入理解 Java 虚拟机》

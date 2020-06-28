# 浅谈JVM中的符号引用和直接引用

## 前言

在 JVM 类加载过程的解析阶段，Java虚拟机将常量池中的符号引用替换为直接引用。符号引用存在于 Class 常量池，在Class文件格式中多以 CONSTANT_Class_info 、 CONSTANT_Fieldref_info 、 CONSTANT_Methodref_info 等类型的常量出现，符号引用和直接引用的区别在于：

- 符号引用是以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只用使用时能无歧义地定位到目标即可，与当前虚拟机实现的内存布局无关。
- 直接引用是可以直接指向目标的指针、相对偏移量或者是一个能间接定位到目标的句柄。直接引用和虚拟机实现的内存布局直接相关的，同一个符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同。如果有了直接引用，那么引用的目标必定已经在虚拟机的内存中存在。

此外，在JVM 中方法调用(分派、执行)过程中，有五条相关指令用于方法调用：

- invokevirtual 指令：用于调用对象的实例方法，根据对象的实际类型进行分派(虚方法分派)，这 也是 Java 语言中最常见的方法分配方式。
- invokeinterface 指令：用于调用接口方法，它会在运行时搜索一个实现了这个接口方法的对象，找出适合的方法进行调用。
- invokespecial 指令：用于调用一些需要特殊处理的实例方法，包括实例初始化方法、私有方法和父类方法。
- invokestatic 指令：用于调用类静态方法( static 方法)。
- invokedynamic 指令：用于在运行时动态解析出调用点限定符所引用的方法，并执行该方法。前面四条调用指令的分派逻辑都固化在 Java 虚拟机内部，用户无法改变，而 invokedynamic 指令的分派逻辑是由用户所设定的引导方法决定的。

方法调用指令与数据类型无关，而方法返回指令是根据返回值的类型区分的，包括 ireturn (当返回值是 boolean、 byte、 char、 short和 int类型时使用)、 ireturn、 ireturn、ireturn 和 ireturn，另外还有一个 return 指令供声明为void的方法、实例初始化方法、类和接口的类初始化方法使用。

以上的概念介绍，结合一个实例来分析举证。看下面的Java代码：

```java
public class Main {
    public static void main(String[] args) {
        Sub sub = new Sub();
        sub.a();
    }
}

class Sub {
    public void a() {
    }
}
```

编译后使用javap工具，得到Class字节码，如下：

```java
Constant pool:
   #1 = Methodref          #2.#3          // java/lang/Object."<init>":()V
   #2 = Class              #4             // java/lang/Object
   #3 = NameAndType        #5:#6          // "<init>":()V
   #4 = Utf8               java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Class              #8             // Sub
   #8 = Utf8               Sub
   #9 = Methodref          #7.#3          // Sub."<init>":()V
  #10 = Methodref          #7.#11         // Sub.a:()V
  #11 = NameAndType        #12:#6         // a:()V
  #12 = Utf8               a
  #13 = Class              #14            // Main
  #14 = Utf8               Main
  #15 = Utf8               Code
  #16 = Utf8               LineNumberTable
  #17 = Utf8               LocalVariableTable
  #18 = Utf8               this
  #19 = Utf8               LMain;
  #20 = Utf8               main
  #21 = Utf8               ([Ljava/lang/String;)V
  #22 = Utf8               args
  #23 = Utf8               [Ljava/lang/String;
  #24 = Utf8               sub
  #25 = Utf8               LSub;
  #26 = Utf8               SourceFile
  #27 = Utf8               Main.java
{
  public Main();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 8: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   LMain;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: new           #7                  // class Sub
         3: dup
         4: invokespecial #9                  // Method Sub."<init>":()V
         7: astore_1
         8: aload_1
         9: invokevirtual #10                 // Method Sub.a:()V
        12: return
      LineNumberTable:
        line 10: 0
        line 11: 8
        line 12: 12
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      13     0  args   [Ljava/lang/String;
            8       5     1   sub   LSub;
}
```

因为篇幅有限，上面的内容只保留了静态常量池和 Code 部分。

- Sub 实例初始化的指令如下，印证invokespecial指令。

```java
4: invokespecial #9                  // Method Sub."<init>":()V
```

- void 方法的返回指令如下，印证return指令。

```java
12: return
```

下面我们主要对 Sub.a() 方法的调用来进行说明。

## 符号引用

在 main 方法的字节码中，调用 Sub.a() 方法的指令如下：

```java
9: invokevirtual #10                 // Method Sub.a:()V
```

invokevirtual 指令就是调用实例方法的指令，后面的操作数 10 是 Class 文件中常量池的下标，表示用来指定要调用的目标方法。我们再来看常量池在这个位置上的内容：

```java
#10 = Methodref          #7.#11         // Sub.a:()V
```

这是一个 Methodref 类型的数据，我们再来看看虚拟机规范中对该类型的说明：

```java
CONSTANT_Methodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
```

这实际上就是一种引用类型，tag 表示了常量池数据类型，这里固定是 10 (参照 Class 常量池项目类型表)。class_index 表示了类的索引，name_and_type_index 表示了名称与类型的索引，这两个也都是常量池的下标。在 javap 的输出中，已经将对应的关系打印了出来，我们可以直接的观察到它都引用了哪些类型：

```java
#10 = Methodref          #7.#11         // Sub.a:()V 类中方法的符号引用
|--#7 = Class              #8             // Sub   类或接口的符号引用
|  |--#8 = Utf8               Sub
|--#11 = NameAndType        #12:#6         // a:()V  字段或者方法的部分符号引用
|  |--#12 = Utf8               a
|  |--#6 = Utf8               ()V
```

这里我们将其表现为树的形式。可以看到，我们可以得到该方法所在的类，以及方法的名称和描述符。于是我们根据 invokevirtual 的操作数，找到了常量池中方法对应的 Methodref，进而找到了方法所在的类以及方法的名称和描述符，当然这些内容最终都是 UTF8 字符串形式。

实际上这就是一个符号引用的例子，符号引用可以理解为利用文字形式来描述引用关系，简单来说是一个包含足够信息的字符串，以供实际使用时可以找到相应的位置。比如说“java/io/PrintStream.println:(Ljava/lang/String;)V”，这里面有类、方法名和方法参数等信息。

## 直接引用

在第一次运行时，发现指令还没有被解析，根据指令去把常量池中有关系的所有项找出来，得到以“UTF-8”编码描述的此方法所属的“类，方法名，描述符”的常量池项，这就是上文中提到的**符号引用**的字符串信息，类的加载解析过程会根据字符串的内容，到该类的方法表中搜索这个方法，得到放发的偏移量(指针)，这个偏移量(指针)就是**直接引用** 。再将偏移量赋给常量池的#10 (根据指令，在常量池中找到的第一个项)，最后再将 Code 中方法调用的 invokevirtual 指令修改为invokevirtual_quick，并把操作数修改成指向方法表的偏移量（指针）， 并加上参数个数，类似于以下格式：

```java
9: invokevirtual_quick 偏移量(指针)                 // Method Sub.a:()V
```

运行一次之后，符号引用会被替换为直接引用，下次就不用重新搜索了。直接引用就是偏移量，通过偏移量虚拟机可以直接在该类的内存区域中找到方法字节码的起始位置。

上面提到的“invokevirtual_quick”是R大借用Sun JDK 1.0.2 虚拟机为例，提出解释直接引用过程的，详细说明请去看 RednaxelaFX 的回答，参考链接在后文。

谈一下我自己的理解，Java代码中所有方法调用的目标方法在 Class 文件里面都是一个常量池的符号引用，在类加载的解析阶段，会将其中符号引用转换为直接引用，这个根据字符串搜素具体方法的过程有些类似于类加载器的运行机制。

此外，解析能够顺利进行的前提是：方法在程序真正运行之前就有一个可确定的调用版本，并且这个方法的调用版本在运行期是不可改变的。换句话说，调用目标在程序代码写好、编译器进行编译的那一刻就已经确定下来了，这类方法的调用就叫解析。

解析调用一定是一个静态的过程，在编译期间就已经完成，将涉及的符号引用全部转变为明确的直接引用，不必延迟到运行期再去完成。而另一种主要的方法调用形式，分派调用则复杂很多，它可能是静态的也可能是动态的，与继承、多态相关，有机会去学习理解一波，这里就先挖个坑，不再做多介绍了。

参考资料：

[《深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）》](https://book.douban.com/subject/34907497/)

[《自己动手写Java虚拟机》](https://book.douban.com/subject/26802084/)

[浅析 JVM 中的符号引用与直接引用](https://www.dazhuanlan.com/2019/10/18/5da94ef24e667/)  

[JVM里的符号引用如何存储？ - RednaxelaFX的回答 - 知乎](https://www.zhihu.com/question/30300585/answer/51335493)  

[JVM方法调用（invokevirtual）](https://www.cnblogs.com/Jc-zhu/p/4482294.html)

ZingBug，2020/6/28
[EOF]

# Java常量池理解及总结

## 1、相关概念

Java中的常量池，实际上分为两种形态：**静态常量池** 和 **运行时常量池** 。

### 1.1 静态常量池

在Class文件结构中，最头的4个字节用于存储魔数Magic Number(OxCAFEBABE)，用于确定一个文件是否能被JVM接受；再接着4个字节用于存储版本号，前2个字节存储次版本号，后2个存储主版本号；再接着是用于存放常量的常量池，由于常量的数量是不固定的，所以常量池的入口放置一个U2类型的数据(constant_pool_count)存储常量池容量计数值。这部分常量池称为静态常量池。与Java中语言习惯不同，常量池容量计数是从1而不是0开始的。

Class常量池中主要存放两大类常量：字面量(Literal)和符号引用(Symbolic References)，字面量比较接近于Java语言层面的常量概念，如文本字符串、被声明为final的常量值等，而符号引用则属于编译原理方面的概念。Class常量将在类加载后进入方法区的运行时常量池中存放（稍后讲解）。其中，符号引用主要包括下面几类常量：

- 被模块导出或者开放的包(Package)
- 类和接口的全限定名(Fully Qualified Name)
- 字段的名称和描述符(Descriptor)
- 方法的名称和描述符
- 方法的句柄和方法类型(Method Handle、Method Type、Invoke Dynamic)
- 动态调用点和动态常量(Dynamically-Computed Call Site、Dynamically-Computed Constant)

常量池中每一项常量都是一个表，共有17种不同类型的表。它们共有一个特定，表结构起始的第一位是个u1类型的标志位(tag，用于区分常量类型)，代表着当前常量属于哪种常量类型。17种常量类型所代表的具体含义如下表所示。
| 类  型 | 标  志 | 描  述 |
| :-----| :----: | :----: |
| CONSTANT_Utf8_info | 1 | UTF-8编码的字符串 |
| CONSTANT_Integer_info | 3 | 整型字面量 |
| CONSTANT_Float_info | 4 | 浮点型字面量 |
| CONSTANT_Long_info | 5 | 长整型字面量 |
| CONSTANT_Double_info | 6 | 双精度浮点型字面量 |
| CONSTANT_Class_info | 7 | 类或接口的符号引用 |
| CONSTANT_String_info | 8 | 字符串类型字面量 |
| CONSTANT_Fieldref_info | 9 | 字段的符号引用 |
| CONSTANT_Methodref_info | 10 | 类中方法的符号引用 |
| CONSTANT_InterfaceMethodref_info | 11 | 接口中方法的符号引用 |
| CONSTANT_NameAndType_info | 12 | 字段或方法的部分符号引用 |
| CONSTANT_MethodHandle_info | 15 | 表示方法句柄 |
| CONSTANT_MethodType_info | 16 | 表示方法类型 |
| CONSTANT_Dynamic_info | 17 | 表示一个动态计算常量 |
| CONSTANT_InvokeDynamic_info | 18 | 表示一个动态方法调用点 |
| CONSTANT_Module_info | 19 | 表示一个模块 |
| CONSTANT_Package_info | 20 | 表示一个模块中开放或者导出的包 |

以简单的Java代码为例：

```java
public static void main(String[] args) {
    String str="zingbug";//
}
```

利用javap命令，生成字节码后，静态常量池如下，具体含义见上表。

```java
Constant pool:
   #1 = Methodref          #2.#3          // java/lang/Object."<init>":()V
   #2 = Class              #4             // java/lang/Object
   #3 = NameAndType        #5:#6          // "<init>":()V
   #4 = Utf8               java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = String             #8             // zingbug
   #8 = Utf8               zingbug
   #9 = Class              #10            // Main
  #10 = Utf8               Main
  ...
```

顺便提一下，最常见的CONSTANT_Utf8_info型常量结构体如下表所示。
| 类  型 | 名  称 | 数  量 |
| :----: :----: | :----- |
| u1 | tag | 1 |
| u2 | length | 1 |
| u1 | bytes | length |

其中，length值说明了这个UTF-8编码的字符串长度是多少字节，它后面就紧跟着长度为length字节的连续数据，用于表示UTF-8编码的字符串。
由于Class文件中方法、字段等都需要引用CONSTANT_Utf8_info型常量来描述名称，所以CONSTANT_Utf8_info型常量的最大长度也就是Java中方法、字段名的最大长度。而这里的最大长度就是length的最大值，即u2类型能表达的最大值65535。所以Java程序中如果定义了超过64KB英文字符的变量或方法名，即使规则和全部字符都是合法的，也会无法编译。

### 1.2 运行时常量池

一个类加载到JVM中后对应一个运行时常量池。运行时常量池，是指jvm虚拟机在类加载的解析阶段，将class常量池内的符号引用替换为直接引用(具体请参考另一篇笔记)，，并保存在方法区中，我们常说的常量池，就是指方法区中的运行时常量池。这其中涉及到类加载的解析阶段。

运行时常量池相对于CLass文件常量池的另外一个重要特征是**具备动态性**，Class文件常量只是一个静态存储结构，里面的引用都是符号引用。而运行时常量池可以在运行期间将符号引用解析为直接引用。可以说运行时常量池就是用来索引和查找字段和方法名称和描述符的。给定任意一个方法或字段的索引，通过这个索引最终可得到该方法或字段所属的类型信息和名称及描述符信息，这涉及到方法的调用和字段获取。

Java语言并不要求常量一定只有编译期才能产生，也就是并非预置入CLass文件中常量池的内容才能进入方法区运行时常量池，运行期间也可能将新的常量放入池中，这种特性被开发人员利用比较多的就是String类的intern()方法。

> String的intern()方法会查找在常量池中是否存在一份equal相等的字符串,如果有则返回该字符串的引用,如果没有则添加自己的字符串进入常量池。

我自己的理解是，**运行时常量池是Class文件常量池在运行时的表示。**

我们都知道运行时常量池是放在方法区的，但需要注意的是，方法区本身就是一个逻辑性的区域。在JDK7及之前，HotSpot使用永久代来实现方法区时，实现是完全符合这种逻辑概念的；但在JDK8及以后，取消了永久代，新增了元空间(Metaspace)，方法区就“四分五裂了”，不再是在单一的一个去区域内进行存储，其中，元空间只存储类和类加载器的元数据信息，符号引用存储在native heap中，字符串常量和静态类型变量存储在普通的堆区中，这时候方法区只是对逻辑概念的表述罢了。

### 1.3 常量池的优势

常量池是为了避免频繁的创建和销毁对象而影响系统性能，其实现了对象的共享。例如字符串常量池，在编译阶段就把所有的字符串文字放到一个常量池中。

- **节省内存空间**：常量池中所有相同的字符串常量被合并，只占用一个空间。
- **节省运行时间**：比较字符串时，==比equals()快。对于两个引用变量，只用==判断地址引用是否相等，也就可以判断实际值是否相等。

> 双等号==的含义：  
> 基本数据类型之间应用双等号，比较的是他们的数值。  
> 复合数据类型(字符串、类)之间应用双等号，比较的是他们在内存中的存放地址。当然，内存地址相同，值也必定相同。

## 2、Java基本数据类型的包装类和缓存池

Java基本数量类型有8种，分别是byte、short、int、long、float、double、char和boolean。

### 2.1 6种基本数据类型的包装类实现了缓存池技术，即Byte、Short、Integer、Long、Character和Boolean

这五种基本数据类型默认创建了数值[-128，127]的相应类型的缓存数据，但是超出此范围仍然会去创建新的对象。以Integer为例：

```java
Integer i1=100;
Integer i2=100;
System.out.println(i1==i2);//输出true
Integer i3=300;
Integer i4=300;
System.out.println(i3==i4);//输出false
```

对于"Integer i1=100;”，Java在编译的时候会直接将代码封装成“Integer i1=Integer.valueOf(100);”，从而使用常量池的缓存。进一步去看Integer.valueOf方法的实现源码。

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

其中，IntegerCache.low为-128，IntegerCache.high默认为127，但也可以通过JVM的启动参数 -XX:AutoBoxCacheMax=size 修改。

更多比较场景：

```java
Integer i1 = 100;
Integer i2 = 0;
Integer i3 = new Integer(100);
Integer i4 = new Integer(100);
Integer i5 = new Integer(0);

System.out.println("i1=i3   " + (i1 == i3));
System.out.println("i1=i2+i3   " + (i1 == i2 + i3));
System.out.println("i3=i4   " + (i3 == i4));
System.out.println("i3=i4+i5   " + (i3 == i4 + i5));
System.out.println("100=i4+i5   " + (100 == i4 + i5));

//输出
i1=i3   false
i1=i2+i3   true
i3=i4   false
i3=i4+i5   true
100=i4+i5   true
```

对此，有如下解释：

- Integer i3 = new Integer(100);这种情况下会创建新的对象，所以i1不等于i3；

- 语句i1 == i2 + i3，因为+这个操作符不适用于Integer对象，首先i2和i3会自动拆箱操作，进行数值相加，即i1 == 100，然后Integer对象无法与数值进行直接比较，所以i1自动拆箱转为int值100，最终这条语句转为100 == 100进行数值比较，故输出true；

- i3和i4分别是不同的新对象，地址不同，故而不相等。

- 语句i3 == i4 + i5，和100 == i4 + i5，同样是自动拆箱操作，最后为数值比较而已。

### 2.2 2种浮点数类型的包装类Float,Double并没有实现缓存池技术

```java
Double d1=1.8;
Double d2=1.8;
System.out.println(d1==d2);//输出false
```

## 3、String类和常量池

### 3.1 不同场景下的String比较

String与常量池的关系比较复杂多样，我们直接以实际代码为例。

```java
String s1 = "Hello";
String s2 = "Hello";
String s3 = "Hel" + "lo";
String s4 = "Hel" + new String("lo");
String s5 = new String("Hello");
String s6 = s5.intern();
String s7 = "H";
String s8 = "ello";
String s9 = s7 + s8;

System.out.println(s1 == s2);  // true
System.out.println(s1 == s3);  // true
System.out.println(s1 == s4);  // false
System.out.println(s1 == s9);  // false
System.out.println(s4 == s5);  // false
System.out.println(s1 == s6);  // true
```

首先说明一点，在String中，直接使用==操作符，比较的是两个字符串的引用地址，并不是比较内容，比较内容请用String.equals()。对上述场景一一分析：

- s1 == s2 这个非常好理解，s1、s2在赋值时，均使用的字符串字面量，说白话点，就是直接把字符串写死，在编译期间，这种字面量会直接放入class文件的常量池中，从而实现复用，载入运行时常量池后，s1、s2指向的是同一个内存地址，所以相等。
- s1 == s3 这个地方有个坑，s3虽然是动态拼接出来的字符串，但是所有参与拼接的部分都是已知的字面量，在编译期间，这种拼接会被优化，编译器直接帮你拼好，因此String s3 = "Hel" + "lo";在class文件中被优化成String s3 = "Hello"，所以s1 == s3成立。**只有使用引号包含文本的方式创建的String对象之间使用“+”连接产生的新对象才会被加入字符串池中。**
- s1 == s4 当然不相等，s4虽然也是拼接出来的，但new String("lo")这部分不是已知字面量，是一个不可预料的部分，编译器不会优化，必须等到运行时才可以确定结果，结合字符串不变定理，不清楚s4被分配到哪去了，所以地址肯定不同。**对于所有包含new方式新建对象（包括null）的“+”连接表达式，它所产生的新对象都不会被加入字符串池中。**
- s1 == s9也不相等，道理差不多，虽然s7、s8在赋值的时候使用的字符串字面量，但是拼接成s9的时候，s7、s8作为两个变量，都是不可预料的，编译器毕竟是编译器，不可能当解释器用，不能在编译期被确定，所以不做优化，只能等到运行时，在堆中创建s7、s8拼接成的新字符串，在堆中地址不确定，不可能与方法区常量池中的s1地址相同。jvm常量池，堆，栈内存分布如下：
![N2xxLd.jpg](https://s1.ax1x.com/2020/06/28/N2xxLd.jpg)
- s4 == s5已经不用解释了，绝对不相等，二者都在堆中，但地址不同。
- s1 == s6这两个相等完全归功于intern方法，s5在堆中，内容为Hello ，intern方法会尝试将Hello字符串添加到常量池中，并返回其在常量池中的地址，因为常量池中已经有了Hello字符串，所以intern方法直接返回地址；而s1在编译期就已经指向常量池了，因此s1和s6指向同一地址，相等。

### 3.2 静态变量赋值

```java
public class Main {
    public static final String A = "ab"; // 常量A
    public static final String B = "cd"; // 常量B

    public static void main(String[] args) {
        String s = A + B;  // 将两个常量用+连接对s进行初始化
        String t = "abcd";
        System.out.println(s == t);//true
    }
}
```

此时A和B都是常量，值是固定的，因此s的值也是固定的，它在类被编译时就已经确定了。也就是说：String s=A+B; 等同于：String s="ab"+"cd";

### 3.3 静态方法赋值

```java
public class Main {
    public static final String A; // 常量A
    public static final String B; // 常量B

    static {
        A = "ab";
        B = "cd";
    }

    public static void main(String[] args) {
        // 将两个常量用+连接对s进行初始化
        String s = A + B;
        String t = "abcd";
        System.out.println(s == t);//false
    }
}
```

此时A和B虽然被定义为常量，但是它们都没有马上被赋值。在运算出s的值之前，他们何时被赋值，以及被赋予什么样的值，都是个变数。因此A和B在被赋值之前，性质类似于一个变量。那么s就不能在编译期被确定，而只能在运行时被创建了。

### 3.4 intern方法

运行时常量池相对于CLass文件常量池的另外一个重要特征是具备动态性，Java语言并不要求常量一定只有编译期才能产生，也就是并非预置入CLass文件中常量池的内容才能进入方法区运行时常量池，运行期间也可能将新的常量放入池中，这种特性被开发人员利用比较多的就是String类的intern()方法。

```java
public class Main {
    public static void main(String[] args) {
        String s1 = new String("Hello");
        String s2 = s1.intern();
        String s3 = "Hello";
        System.out.println(s1 == s2);//false
        System.out.println(s3 == s2);//true
    }
}
```

String的intern()方法会查找在常量池中是否存在一份equal相等的字符串,如果有则返回指向该字符串的引用(CONSTAT_String_info),如果没有则添加自己的字符串进入常量池，也同时返回引用。

### 3.5 延伸

```java
String s1 = new String("Hello");
```

上面这行代码中，一共创建了几个对象？

"Hello"会首先在常量池中里创建一个字符常量，然后在new创建对象时，会将常量池中"Hello"的字符串复制到堆中新创建的对象字符数组中，并建立引用s1指向它。所以，这条语句共创建了2个对象，分别位于常量池和堆。

## 4、总结

- 运行时常量池中的常量，基本来源于各个Class文件中的常量池。也就是说，运行时常量池是Class文件常量池在运行时的表示。
- 程序运行时，除非手动向常量池中添加常量(比如调用intern方法)，否则JVM不会自动添加常量到常量池。

## 5、参考资料

[《Java核心技术系列：Java虚拟机规范（Java SE 8版）》](https://book.douban.com/subject/26418340/)

[《深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）》](https://book.douban.com/subject/34907497/)

[浅谈JVM中的符号引用和直接引用](/notes/浅谈JVM中的符号引用和直接引用.md)

[深入浅出java常量池](https://www.cnblogs.com/syp172654682/p/8082625.html)  

[Java常量池理解与总结](https://mp.weixin.qq.com/s?__biz=MzU3NDg0MTY0NQ==&mid=2247485452&idx=1&sn=64178cb2b4e2768b2feedbe0d0971ee1&chksm=fd2d7e4eca5af7587be00fccbec403117cdb7eb681c0542ff542335b1239049bd0216fc1bb1d&mpshare=1&scene=1&srcid=0623b0MIhBaU2sZEhvxEjoME&sharer_sharetime=1592875293438&sharer_shareid=e9e6d524172161e8393308ae6db3aa63&key=242af3e89b1070825a226d604188d71bc5c856baa07c40c96c01389ddc232167ec2a8196658f02f540cce13a26664b46549909f0156708f4d7d26fc7c470faeb92a6c0d4f3ad7328a7cbe59a5c3a0cd6&ascene=1&uin=MjQ3MzkwMTc2Mw%3D%3D&devicetype=Windows+10+x64&version=62090523&lang=zh_CN&exportkey=AVRh0T9RxdKUe0T%2BS0Vu7Jk%3D&pass_ticket=mrrMmVKWvl4QF8i0mBVDO7Xre7kYXlm7qLoXUV%2FJeUsPvRILMjcMWMW1A%2BzBALMH)

ZingBug，2020/6/28
[EOF]

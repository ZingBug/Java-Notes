# 从字节码角度分析try、catch、finally运行细节

## 0、字节码指令

回顾一下字节码的指令含义(参考《Java虚拟机规范》-2.11.2节 加载和存储指令):
加载和存储指令用于将数据从栈帧的本地变量表和操作数栈之间来回传递。

> - 将一个本地变量加载到操作数栈的指令包括：*iload、iload_&lt;n&gt;、lload、lload_&lt;n&gt;、fload、fload_&lt;n&gt;、dload、dload_&lt;n&gt;、aload、aload_&lt;n&gt;*。  
> - 将一个数值从操作数栈存储到局部变量表的指令包括：*istore、istore_&lt;n&gt;、lstore、lstore_&lt;n&gt;、fstore、fstore_&lt;n&gt;、dstore、dstore_&lt;n&gt;、astore、astore_&lt;n&gt;*。  
> - 将一个常量加载到操作数栈的指令包括：*bipush、sipush、ldc、ldc_w、ldc2_w、aconst_null、iconst_null、iconst_&lt;i&gt;、lconst_&lt;i&gt;、fconst_&lt;i&gt;、dconst_&lt;i&gt;*。  
> - 用户扩充局部变量表的访问索引或立即数的指令：*wide*。  

上面所列举的指令助记符中，有一部分是以尖括号结尾的（例如*iload_&lt;n&gt;*），这些指令助记符实际上代表了一组指令（例如*iload_&lt;n&gt;*代表了*iload_0、iload_1、iload_2和iload_3*这几个指令，需要注意的是，n是从0开始计数的）。这几组指令都是某个带有一个操作数的通用指令（例如*iload*）的特殊形式。在尖括号之间的字母指定了指令隐含操作数的数据类型，&lt;n&gt;代表非负的整数，&lt;i&gt;代表是int类型数据，&lt;l&gt;代表了long类型，&lt;f&gt;表示float类型，&lt;d&gt;代表了double类型。操作byte、char和short类型数据时，经常用int类型的指令来表示。
另外，根据int值范围，JVM整型入栈指令分为四类：当int取值-1\~5采用iconst指令，取值-128\~127采用bipush指令，取值-32768\~32767采用sipush指令，取值-2147483648\~2147483647采用 ldc 指令。(参考[JVM字节码之整型入栈指令(iconst、bipush、sipush、ldc)](https://blog.csdn.net/zhaow823/article/details/81199093))

## 1、简单的try、catch、finally例子

写一个最简单的try、catch、finally例子，分析最后返回的结果，并用javap -verbose 命令来显示目标文件(.class文件)字节码信息，如下所示。

> 系统运行环境：Microsoft Windows [版本 10.0.19041.329] 系统 64bit  
> JDK信息：Java(TM) SE Runtime Environment (build 14.0.1+7)
Java HotSpot(TM) 64-Bit Server VM (build 14.0.1+7, mixed mode, sharing)

```java
//Java代码
public int inc() {
    int x;
    try {
        x = 1;
         return x;
    } catch (Exception e) {
        x = 2;
        return x;
    } finally {
           x = 3;
    }
}
//编译后的ByteCode字节码和异常表
public int inc();
    descriptor: ()I
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=5, args_size=1
         0: iconst_1  //try块，将int类型值(1)压入栈顶
         1: istore_1  //将栈顶数据(1)存入到第二个本地变量（从0开始计数）
         2: iload_1   //将第二个本地int变量(1)压入栈顶
         3: istore_2  //将栈顶数据(1)存入第三个本地变量
         4: iconst_3  //将int类型值(3)压入栈顶，第一个finally
         5: istore_1  //将栈顶数据(3)存入到第二个本地变量
         6: iload_2   //将第三个本地int变量(1)压入栈顶
         7: ireturn   //返回栈顶数据值(1)
         8: astore_2  //给catch中定义的Exception e赋值，并存储到第三个变量槽中
         9: iconst_2  //将int类型值(2)压入栈顶
        10: istore_1  //将栈顶数据(2)存入到第二个本地变量
        11: iload_1   //将第二个本地int变量(2)压入栈顶
        12: istore_3  //将栈顶数据(2)存入到第四个本地变量
        13: iconst_3  //将int类型值(3)压入栈顶，第二个finally
        14: istore_1  //将栈顶数据(3)存入到第二个本地变量
        15: iload_3   //将第四个本地int变量(2)压入栈顶
        16: ireturn   //返回栈顶数据值(2)
        17: astore        4  //如果出现了不属于java.lang.Exception及其子类的异常才会走到这
        19: iconst_3  //将int类型值(3)压入栈顶，第三个finally
        20: istore_1  //将栈顶数据(3)存入到第二个本地变量
        21: aload         4  //将异常放入栈顶
        23: athrow    //抛出栈顶的异常
      Exception table:  //异常表
         from    to  target type
             0     4     8   Class java/lang/Exception
             0     4    17   any
             8    13    17   any
            17    19    17   any
```

先解释一下异常表组成：

- From : 从第几行开始检测；
- To ：到第几行结束；
- Target : 假如From和To中的代码执行发生异常跳到哪一行处理；
- Type : Java代码中定义的全部异常类型。
Type 为 any 的情况，就是抛出了捕获不到的类型。所以全部跳到17行去处理。17行就是Java代码中定义的finally语句块，相当于最后一道防线。

编译器为这段Java代码生成了四条异常表记录，对应四条代码可能出现的执行路径，从Java代码的语义上讲，这四条执行路径分别为：

- 如果try语句块中出现属于Exception及其子类的异常时，会转到catch语句块处理；
- 如果try语句块中出现不属于Exception及其子类的异常时，会转到finally语句块处理；
- 如果catch语句块中出现不属于Exception及其子类的异常时，会转到finally语句块处理；
- 如果finally语句块中出现不属于Exception及其子类的异常时，会转到finally语句块处理；

返回到一开始的问题，这段代码的返回值是多少？答案是：

- 如果没有出现异常值，返回值是1；
- 如果出现了Exception异常，返回值是2；
- 如果出现了Exception以外的异常，没有返回值。

具体的运行逻辑根据上文字节码的注释可以知道，字节码中第0-3行所做的操作是将整数1赋值给变量x，并且将此时x的值复制一份副本到第三个本地变量槽中，作为后面方法返回值使用。如果这时候没有出现异常，则会继续走到第4-5行，将变量x赋值为3，然后将之前保存在第三个本地变量槽中的整数1读入到操作栈顶，最后ireturn指令会以int形式返回操作栈顶中的值，方法结束。如果出现了异常，PC寄存器指针转到第8行，第9-12行所做的事情就是将2赋值给变量x，然后将变量x此时的值赋给第四个本地变量槽，最后再将变量x的值改为3。方法返回前同样将第四个本地变量槽中保留的整数2读到了操作栈顶。从第17行开始的代码，作用是将变量x的值赋为3，并将栈顶的异常抛出，方法结束。

## 2、finally中加入return的例子

改一下代码，在finally中加入return，这样这段代码中有两个return语句，但是程序到底返回的是try 还是 finally。接下来我们还是看字节码信息。

```java
//Java代码
public int inc() {
    int x;
    try {
        x = 1;
        return x;
    } catch (Exception e) {
        x = 2;
        return x;
    } finally {
        x = 3;
        return x;//加入了return
    }
}
//编译后的ByteCode字节码和异常表
public int inc();
    descriptor: ()I
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=5, args_size=1
         0: iconst_1  //try块，将int类型值(1)压入栈顶
         1: istore_1  //将栈顶数据(1)存入到第二个本地变量（从0开始计数）
         2: iload_1   //将第二个本地int变量(1)压入栈顶
         3: istore_2  //将栈顶数据(1)存入第三个本地变量
         4: iconst_3  //将int类型值(3)压入栈顶，第一个finally
         5: istore_1  //将栈顶数据(3)存入到第二个本地变量
         6: iload_1   //将第二个本地int变量(3)压入栈顶
         7: ireturn   //返回栈顶数据值(3)
         8: astore_2  //给catch中定义的Exception e赋值，并存储到第三个变量槽中
         9: iconst_2  //将int类型值(2)压入栈顶
        10: istore_1  //将栈顶数据(2)存入到第二个本地变量
        11: iload_1  //将第二个本地int变量(2)压入栈顶
        12: istore_3  //将栈顶数据(2)存入到第四个本地变量
        13: iconst_3  //将int类型值(3)压入栈顶，第二个finally
        14: istore_1  //将栈顶数据(3)存入到第二个本地变量
        15: iload_1  //将第二个本地int变量(3)压入栈顶
        16: ireturn   //返回栈顶数据值(3)
        17: astore        4  //如果出现了不属于java.lang.Exception及其子类的异常才会走到这
        19: iconst_3  //将int类型值(3)压入栈顶，第三个finally
        20: istore_1  //将栈顶数据(3)存入到第二个本地变量
        21: iload_1  //将第2个本地int变量(3)压入栈顶
        22: ireturn  //异常都不用抛出了，直接返回栈顶值(3)
      Exception table:
         from    to  target type
             0     4     8   Class java/lang/Exception
             0     4    17   any
             8    13    17   any
            17    19    17   any
```

与第一个例子相比，大部分代码相似，但区别在于，try语句块中的return被忽略覆盖掉了，它没有进行真正的返回，只是进行一个赋值操作而已，直到finally重新修改了x=3，再return。需要注意的是，异常athrow操作指令消失了。

正常来说我们catch到的Exception e，在这一步处理的时候，如果finally有return，那么会发生吞噬异常的情况，也就是抛出的异常不见了。

## 3、catch多个异常类型的例子

直接去看代码和字节码中的异常表：

```java
//Java代码
public void inc() {
    int x;
    try {
        x = 1;
    } catch (NullPointerException e) {
        x = 2;
    } catch (RuntimeException e) {
         x = 3;
    } catch (Exception e) {
        x = 4;
    } finally {
        x = 5;
    }
}
//字节码中的异常表部分
Exception table:
         from    to  target type
             0     2     7   Class java/lang/NullPointerException
             0     2    15   Class java/lang/RuntimeException
             0     2    23   Class java/lang/Exception
             0     2    31   any
             7    10    31   any
            15    18    31   any
            23    26    31   any
```

当catch多个异常的时候，字节码中异常表会有对应的体现。需要注意的是，小的异常要放在前面，大的异常类放在后面。不然编译不会通过。这样是为了防止大炮打蚊子的浪费，如下所示，我们将大范围的Exception异常放在RuntimeException异常前面的时候，会出现报错现象。

![Nwd5tS.png](https://s1.ax1x.com/2020/06/24/Nwd5tS.png)

## 3、总结

对以上所有的例子进行总结

- try、catch、finally语句中，在如果try语句有return语句，则返回的之后当前try中变量此时对应的值，此后对变量做任何的修改，都不影响try中return的返回值。

- 如果finally块中有return 语句，则返回try或catch中的返回语句忽略。

- 如果finally块中抛出异常，则整个try、catch、finally块中抛出异常

所以使用try、catch、finally语句块中需要注意的是：

- 尽量在try或者catch中使用return语句。通过finally块中达到对try或者catch返回值修改是不可行的。

- finally块中避免使用return语句，因为finally块中如果使用return语句，会显示的消化掉try、catch块中的异常信息，屏蔽了错误的发生。

- finally块中避免再次抛出异常，否则整个包含try语句块的方法回抛出异常，并且会消化掉try、catch块中的异常。

参考资料：  
《Java核心技术系列：Java虚拟机规范（Java SE 8版）》  
《深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）》  
[Java字节码指令收集大全](https://www.cnblogs.com/longjee/p/8675771.html)  
[try catch finally异常机制Java源码+JVM字节码（内含小彩蛋）](https://blog.csdn.net/whiteBearClimb/article/details/104131005)  
[Java中关于try、catch、finally中的细节分析](https://mp.weixin.qq.com/s?__biz=MzU3NDg0MTY0NQ==&mid=2247485213&idx=1&sn=982c5e92d2c15a48b59194258a40c2d2&chksm=fd2d715fca5af84976b8cd034fc52aa94e78c2750ba6bcb586f979438bf38baee283fee7de9b&mpshare=1&scene=1&srcid=0623mdjtJSGrvIRPkaj6HhNx&sharer_sharetime=1592875254832&sharer_shareid=e9e6d524172161e8393308ae6db3aa63&key=c32b5c198f0599707981fa0619311426b3104054b7dd310f3346ed0d2159420a6119aaa288eb9df1d369808227d2bb1eec8f9cbe2510d831061991b4215d7bd786a293f6f9c1033009510d6df3fc2f7f&ascene=1&uin=MjQ3MzkwMTc2Mw%3D%3D&devicetype=Windows+10+x64&version=6209007b&lang=zh_CN&exportkey=AYaxCNjrIahDUSf8A0fYYT4%3D&pass_ticket=4yLw2AbquUCok67oAc4LWcLTUZY6fTDbcUcWHB4Mj69rD%2BHCBbKgabd4I%2BGIDkfk)

ZingBug，2020/6/24
[EOF]

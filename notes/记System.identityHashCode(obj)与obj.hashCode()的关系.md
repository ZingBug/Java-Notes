# System.identityHashCode(obj)与obj.hashCode()的关系

有时候看一些源码的时候，经常出现System.identityHashCode(obj) 的使用，这里仔细去讨论一下这个方法与平常的obj.hashCode()方法的关系。

## 首先去回顾一下hashcode的概念

hashcode是jdk根据对象的地址算出来的一个int数字，即对象的哈希码值，代表了该对象在哈希表中的存储位置。（这里不直接使用物理地址是为了散列存储查找的快捷性）

很多场景下，只要判断两个对象的hashCode是否一致，就知道这两个对象是否相同。hashCode()的性能要比“==”性能高出很多，因为hashCode()的结果计算后会缓存起来，下次直接调用而不需要重新计算。

**回到正文，去依次对比看一下两者的不同。**

## System.identityHashCode(obj)

identityHashCode是System里面提供的本地方法。

```java
    /**
     * Returns the same hash code for the given object as
     * would be returned by the default method hashCode(),
     * whether or not the given object's class overrides
     * hashCode().
     * The hash code for the null reference is zero.
     *
     * @param x object for which the hashCode is to be calculated
     * @return  the hashCode
     * @since   JDK1.1
     */
    public static native int identityHashCode(Object x);
```

看官方注释可知，该方法给定对象的哈希码，与默认的方法hashCode()返回的代码一样，无论给定对象的类是否重写了hashCode()，null引用的哈希码为0。

## obj.hashCode()

```java
public native int hashCode();
```

hashCode()方法是顶级类Object类提供的一个方法，所有的类都可以对hashCode方法就i选哪个重写，此时hash值计算是根据重写后的hashCode方法计算。

从上面概念来说，很清晰的看出来identityHashCode是根据Object类hashCode()方法来计算hash值，无论子类是否重写了hashCode()方法，而obj.hashcode()是根据obj的hashcode()方法计算hash值。

举个例子来验证一下。

```java
public class Main {

    private static class A {
        String a;

        A(String a) {
            this.a = a;
        }
    }

    private static class B {
        String b;

        B(String b) {
            this.b = b;
        }

        //重写了hashCode方法
        @Override
        public int hashCode() {
            return 17 * 34 + b.hashCode();
        }
    }

    public static void main(String[] args) {
        String s1 = "hello world";
        String s2 = new String("hello world");
        System.out.println("s1 identityHashCode: " + System.identityHashCode(s1));
        System.out.println("s1 hashcode: " + s1.hashCode());
        System.out.println("s2 identityHashCode: " + System.identityHashCode(s2));
        System.out.println("s2 hashcode: " + s2.hashCode());

        A a = new A("hello world");
        System.out.println("a identityHashCode: " + System.identityHashCode(a));
        System.out.println("a hashcode: " + a.hashCode());

        B b = new B("hello world");
        System.out.println("b identityHashCode: " + System.identityHashCode(b));
        System.out.println("b hashcode: " + b.hashCode());
    }
}

//输出
s1 identityHashCode: 21685669
s1 hashcode: 1794106052
s2 identityHashCode: 2133927002
s2 hashcode: 1794106052
a identityHashCode: 2133927002
a hashcode: 2133927002
b identityHashCode: 1836019240
b hashcode: 1794106630
```

从上面案例分析：

- A类没有重写hashCode()方法，所以两个都是直接调用Object的HashCode()方法，结果都一样。

- B类重写了hashCode()方法，identityHashCode是调用了Object的HashCode()方法，而hashCode是调用了重写后的hashCode()计算方法，两者结果不一样。

- String类重写了hashCode()方法，它是根据String具体值来计算的哈希值，只要值一样，hashCode也一样。所以会看到s1和s2的hashCode是一致的，但是identityHashCode结果不一样（**不同VM中计算方法不同，Google Dalvik VM就是使用对象地址来实现identityHashCode，而HotSpot VM默认是通过伪随机数生成器来实现**），当然，s1和s2的equal是一样的（equal方法是比较实际值），但“==”不一样（因为地址引用不同）。

到这时就有一个疑惑，**identityHashCode到底有什么用处吗？**

identityHashCode()方法保证了返回的值在对象的生命周期内不会改变，注意的是，identityHashCode不能严格保证唯一！这个标识符在理论上可以用于哈希和哈希表以外的其他用途。

## 总结

hashCode方法可以被重写，并返回重写后的值。

identityHashCode方法是返回对象的hash值，而不管对象是否重写了hashCode方法。

## 参考资料

[System.identityHashCode(obj) 与 obj.hashcode()](https://www.jianshu.com/p/24fa4bdb9b9d)

[关于Object的identity hash code是否存在重复的问题](https://hllvm-group.iteye.com/group/topic/39183)

[GC复制存活的对象，内存地址会变吗？以前的引用怎么办]( https://www.zhihu.com/question/49631727/answer/120113928)

弥有，2020年5月22日
[EOF]

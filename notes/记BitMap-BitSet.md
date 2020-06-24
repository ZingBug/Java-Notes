# 2020届秋招面试题总结——网络篇

提到一个bitmap，肯定想到它是非常经典的海量数据处理工具，其本质是用bit数组的某一位来表示某一数据，从而一个bit数组可以表示海量数据。

在Java中，BitSet类实现了bitmap算法。BitSet是位操作的对象，值只有0或1即false和true，**内部维护了一个long数组，初始只有一个long，所以BitSet最小的size是64，当随着存储的元素越来越多，BitSet内部会动态扩充，最终内部是由N个long来存储**，这些针对操作都是透明的。

**用1位来表示一个数据是否出现过，0为没有出现过，1表示出现过。使用用的时候既可根据某一个是否为0表示，此数是否出现过。**

一个1G的空间，有 `8*1024*1024*1024=8.58*10^9bit`，也就是可以表示85亿个不同的数。

我们来看一下其内部实现，首先看一下**构造函数**。

```java
private final static int ADDRESS_BITS_PER_WORD = 6;
private long[] words;//用一个long数组来存储位
public BitSet(int nbits) {
    // nbits can't be negative; size 0 is OK
    if (nbits < 0)
        throw new NegativeArraySizeException("nbits < 0: " + nbits);
    initWords(nbits);
    sizeIsSticky = true;
}
private void initWords(int nbits) {
    words = new long[wordIndex(nbits-1) + 1];//数组初始化
}
private static int wordIndex(int bitIndex) {
    return bitIndex >> ADDRESS_BITS_PER_WORD;//获取数组长度
}
```

关键在于wordIndex函数，注意这里ADDRESS_BITS_PER_WORD的值是6。为什么是6呢，答案很简单。

在最开始提到的：BitSet里使用一个Long数组里的每一位来存放当前Index是否有数存在。

因为在Java里Long类型是64位，所以一个Long可以存储64个数，而要计算给定的参数bitIndex应该放在数组（在BitSet里存在word的实例变量里）的哪个long里，只需要计算：bitIndex / 64即可，这里正是使用>>来代替除法（因为位运算要比除法效率高）。而64正好是2的6次幂，所以ADDRESS_BITS_PER_WORD的值是6。

通过wordIndex函数就能计算出参数bitIndex应该存放在words数组里的哪一个long里。

接下来看一下**set函数**。

```java
public void set(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);
    int wordIndex = wordIndex(bitIndex);
    expandTo(wordIndex);
    words[wordIndex] |= (1L << bitIndex); // Restores invariants
    checkInvariants();
}
```

这里面比较重要的就是标红的那句。

```java
words[wordIndex] |= (1L << bitIndex);
```

类似于

1<<0 = 1（1）

1<<1 = 2（10）

1<<2 = 4（100）

1<<3 = 8（1000）

**(1L << bitIndex)就意味着将bitIndex所对应的位设置为1，表示这个数字已经存在了。**剩下的就是用|来将当前算出来的和以前的值进行合并了。

最后看一下**get函数**。

```java
public boolean get(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);
    checkInvariants();
    int wordIndex = wordIndex(bitIndex);
    return (wordIndex < wordsInUse)
        && ((words[wordIndex] & (1L << bitIndex)) != 0);
}
```

整体意思就是判断bitIndex所对应的位数是否为1。

怎么去**运用**这个bitMap思想呢。

首先，在原理上（1L << bitIndex)，这个联想到LeetCode上一个题，第268题，**Missing Number**，从0到n之间取出n个不同的数，找出漏掉的那个。

这个题的思路有很多，先举一个，先对0-n的数运用加法进行求和得到SUM1，然后对数组遍历求和得到SUM2，那漏掉的就是SUM1-SUM2了。

这次我们采用bitset的思想来解决。遍历数组，用一个整数的位来记录。

```java
public int missingNumberInByBitSet(int[] array) {
    int bitSet = 0;
    for (int element : array) {
        bitSet |= 1 << element;
    }
    for (int i = 0; i < array.length; i++) {
        if ((bitSet & 1 << i) == 0) {
            return i;
        }
    }
    return 0;
}
```

这里不用数组，只用一个int就可以实现。

回到这个文章开头，bitMap思想最主要的应用是在大数据量上。比如说在20亿个整数中找到一个数，或者找到唯一重复的数字，如下。

```java
public class BitSetTest {
    public static void main(String[] args)
    {
        BitSet bitSet=new BitSet(2000000000);
        int[] nums={1,2,3,4,5,6,7,5};//等等，一共有20亿个数字
        for(int num:nums)
        {
            if(bitSet.get(num))
            {
                System.out.println(num);
                break;
            }
            bitSet.set(num);
        }
    }
}
```

为什么刚才那个问题是20亿个数字呢，是因为BitSet在构造函数中，**传入的长度变量类型为int，int最大值为2147483647**，也就是20亿左右，所以限制了每个BitSet的长度。

如果遇到比20亿更多的数字怎么办，比如说50亿。那么可以采用**分段思想**，将50亿中的最大值进行分段，得到多个BitSet，每段的BitSet都在int下标范围内。代码如下所示。

```java
public class HugeBitset {
    //分段式bitset存储在list中
    private List<BitSet> bitsetList = new ArrayList(){};
    //下标最大值
    private long max;
    //每段值大小
    private int seg;
    //根据下标最大值，段大小初始化bitset
    HugeBitset(long max,int seg){
        this.max = max;
        this.seg = seg;
        int segs = (int)(max/seg) +1;
        for(int i = 0;i<segs;i++){
            BitSet temp = new BitSet(seg);
            bitsetList.add(temp);
        }
        System.out.println("HugeBitset#seg counts is:"+bitsetList.size()+", max is :"+max+", seg is :"+seg);
    }
    //获取段 bitset
    public BitSet getSegBitset(long index){
        int segNo = (int)(index/seg);
        System.out.println("getSegBitset#segNo is:"+segNo+",index is :"+index);
        return bitsetList.get(segNo);
    }
    //获取段 bitset的偏移量
    public int getBsOffset(long index){
        int offset = (int)(index%seg);
        System.out.println("getBsOffset#offset is:"+offset+",index is :"+index);
        return offset;
    }
    public static void main(String[] args){
        long testmax = 6748347838l;//最大值
        int testseg=500000000;//数目
        long testindex1 = 38475643l;//测试案例
        long testindex2 = 838888843l;
        long testindex3 = 4838888843l;
        long testindex4 = 6738888843l;
        HugeBitset testhb = new HugeBitset(testmax,testseg);
        testhb.getSegBitset(testindex1).set(testhb.getBsOffset(testindex1));
        testhb.getSegBitset(testindex2).set(testhb.getBsOffset(testindex2));
        testhb.getSegBitset(testindex3).set(testhb.getBsOffset(testindex3));
        testhb.getSegBitset(testindex4).set(testhb.getBsOffset(testindex4));
        System.out.println("GetResults:testindex1:"+testhb.getSegBitset(testindex1).get(testhb.getBsOffset(testindex1))
        +" , testindex2:"+testhb.getSegBitset(testindex2).get(testhb.getBsOffset(testindex2))
                        +" , testindex3:"+testhb.getSegBitset(testindex3).get(testhb.getBsOffset(testindex3))
                        +" , testindex4:"+testhb.getSegBitset(testindex4).get(testhb.getBsOffset(testindex4))
);
```

参考文章：

[Java的BitSet原理及应用](https://www.jianshu.com/p/4fbad3a6d253)
[无限下标超大型bitset的java实现，超越原生int 20亿下标的限制](https://blog.csdn.net/flyflyflyflyflyfly/article/details/82952529)

弥有，2019年8月5日
[EOF]

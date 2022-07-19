# Java基础

## Java语言特点

- 简单易学；
- 面向对象（封装，继承，多态）；
- 平台无关性（ Java 虚拟机实现平台无关性）；
- 支持多线程（ C++ 语言没有内置的多线程机制，因此必须调用操作系统的多线程功能来进行多线程程序设计，而 Java 语言却提供了多线程支持）；
- 可靠性；
- 安全性；
- 支持网络编程并且很方便（ Java 语言诞生本身就是为简化网络编程设计的，因此 Java 语言不仅支持网络编程而且很方便）；
- 编译与解释并存；
- c++11后提供多线程lib

## 可变长参数

- 实际为数组
- 多个参数，可变长参数必须放最后一位
- 多用于封装，不要使用到api层

## HashMap

- 初始化因子0.75,最小容量16，如果初始化达12个数目，则达到32，若一次性写入，可优化因子

- 最小容量16，并且Java7以后为懒初始化，避免内存消耗

- 线程不安全

- Java 8之前直接使用HashMap中的numberOfLeadingZeros方法计算初始化容量，之后使用Interger中的该方法，初始化容量会被计算为2的N次方

- numberOfLeadingZeros方法,通过依次位运算取或，将进位最高的第一个1后的全部置0，去到大于传入初始化容量的最小的二次幂，该方法全部是位运算，计算机编译后可以高效执行

```java
    @HotSpotIntrinsicCandidate
    public static int numberOfLeadingZeros(int i) {
        // HD, Count leading 0's
        if (i <= 0)
            return i == 0 ? 32 : 0;
        int n = 31;
        if (i >= 1 << 16) { n -= 16; i >>>= 16; }
        if (i >= 1 <<  8) { n -=  8; i >>>=  8; }
        if (i >= 1 <<  4) { n -=  4; i >>>=  4; }
        if (i >= 1 <<  2) { n -=  2; i >>>=  2; }
        return n - (i >>> 1);
    }
```

- Hashmap的hash方法被重写，会进行高8与低8位取或，进行hash扰动，减少hash碰撞

## Java注释注解

- 不存在@date注解

- 符合Java Doc规范的常用注解

```java
/**
 * @author zhanghong
 * @since 
 */
```
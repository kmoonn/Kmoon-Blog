在 Java 后端的世界里，HashMap 是一个绕不开的类。

它既是日常开发中最常用的容器之一，也是面试中被反复追问的“基础题”，甚至还曾因为设计取舍，在并发场景下制造过著名的 CPU 100% 死循环问题。

本文将从 JDK 1.0 到 JDK 24，回顾 HashMap 的设计演进，试图回答三个问题：

- HashMap 一开始是如何设计的？
- 为什么 JDK 1.8 对它进行了“重构级”改造？
- HashMap 背后体现了怎样的工程思想？

# 为什么 HashMap 如此重要？

HashMap 的重要性不在于“会不会用”，而在于：

- 它综合了 数组、链表、红黑树 等经典数据结构

- 它体现了 时间与空间的权衡

- 它明确区分了 单线程容器 与 并发容器 的设计边界

- 它的演进几乎映射了 Java 集合框架的成熟过程

理解 HashMap，往往是 Java 开发者从“API 使用者”走向“底层理解者”的一道分水岭。

# JDK 1.2 ~ 1.7：草莽时代的 HashMap

JDK 1.0/1.1 只有 Hashtable，HashMap 是在 JDK 1.2 的 Collection Framework 中引入的。

## 底层结构

早期 HashMap 的底层结构非常直接：**数组 + 链表**

## 关键参数

## Hash 计算方式

## 元素插入方式：头插法

- JDK 1.2

```java
package java.util;
import java.io.*;

public class HashMap<K, V> 
        extends AbstractMap<K, V> 
        implements Map<K, V>, Cloneable, Serializable 
{
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
    static final int MAXIMUM_CAPACITY = 1 << 30;
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    static final Entry<?,?>[] EMPTY_TABLE = {};
    transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;
    transient int size;
    final float loadFactor;
    transient int modCount;
    static final int ALTERNATIVE_HASHING_THRESHOLD_DEFAULT = Integer.MAX_VALUE;
}
```

- JDK 1.0
```java

```


# JDK 1.8：HashMap 的分水岭

如果说 HashMap 的历史只有一个转折点，那一定是 JDK1.8。

这一版本几乎重写了 HashMap 的核心逻辑。

## 底层结构升级：引入红黑树

## 插入方式改变：尾插法

## 更合理的 Hash 扰动函数

## resize 机制的重大优化

# JDK 9 ~ JDK 24：稳定后的演进期

在 JDK8 之后，HashMap 的核心设计已经趋于稳定。

可以认为：JDK8 定义了现代 HashMap 的最终形态。

# HashMap 背后的设计哲学

回顾 HashMap 的演进，可以总结出几条非常典型的工程取舍。

## 1. 用空间换时间

负载因子 0.75

主动扩容

降低冲突概率

## 2. 为单线程 / 低并发场景优化

HashMap 明确不是线程安全的

并发问题交由 ConcurrentHashMap 解决

职责清晰，避免过度设计

### 3. 常规情况高效，极端情况兜底

链表应对大多数场景

红黑树兜底最坏复杂度 O(log n)
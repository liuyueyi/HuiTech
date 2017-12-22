\# jdk map学习之HashMap \(一\) ： 底层存储结构分析

&gt; by 一灰



\#\# 底层数据结构



首先通过源码，类中的field如下，



\`\`\`java

transient Node&lt;K,V&gt;\[\] table;



transient Set&lt;Map.Entry&lt;K,V&gt;&gt; entrySet;



transient int size;



transient int modCount;



int threshold;



final float loadFactor;

\`\`\`



其中 \`Node\`, \`Map.Entry\` 是两个比较核心的数据结构，先看下Node的定义



\#\#\# 1. \`Map.Entry\`

&gt; Map接口中内部定义的接口, 提供了操作Map中键值对的基本方法



一个\`Entry\`对象，代表了Map中的一个键值对，可以通过它获取key，value也可以重新设置value



\`\`\`java

interface Entry&lt;K,V&gt; {

  K getKey\(\);



  V getValue\(\);



  V setValue\(V value\);



  boolean equals\(Object o\);



  int hashCode\(\);

}

\`\`\`



依次说明下上面的每个方法的作用



\#\#\#\# 获取键 ： \`K getKey\(\)\`



\#\#\#\# 获取值 ： \`V getValue\(\)\`



\#\#\#\# 设置值 : \`V setValue\(V value\)\`



\#\#\#\# haseCode 方法

返回entry 的 \`hash code\`, 定义如下：



\`\`\`java

\(e.getKey\(\)==null   ? 0 : e.getKey\(\).hashCode\(\)\) ^

\(e.getValue\(\)==null ? 0 : e.getValue\(\).hashCode\(\)\)

\`\`\`



确保两个 Entry对象 equals返回true，则hashcode的值必然相同



\#\#\#\# equals 方法



当两个entry对象表示的是同一个映射关系时，返回true



规则如下



\`\`\`java

\(e1.getKey\(\)==null ? e2.getKey\(\)==null : e1.getKey\(\).equals\(e2.getKey\(\)\)\)   &&

\(e1.getValue\(\)==null ? e2.getValue\(\) ==null : e1.getValue\(\).equals\(e2.getValue\(\)\)\)

\`\`\`



\#\#\# 2. \`Node&lt;K, V&gt;\`

&gt; 作为HashMap中对 Map.Entry的实现，具体逻辑如下



\`\`\`java

static class Node&lt;K,V&gt; implements Map.Entry&lt;K,V&gt; {

    final int hash;

    final K key;

    V value;

    Node&lt;K,V&gt; next;



    Node\(int hash, K key, V value, Node&lt;K,V&gt; next\) {

        this.hash = hash;

        this.key = key;

        this.value = value;

        this.next = next;

    }



    public final K getKey\(\)        { return key; }

    public final V getValue\(\)      { return value; }

    public final String toString\(\) { return key + "=" + value; }



    public final int hashCode\(\) {

        return Objects.hashCode\(key\) ^ Objects.hashCode\(value\);

    }



    public final V setValue\(V newValue\) {

        V oldValue = value;

        value = newValue;

        return oldValue;

    }



    public final boolean equals\(Object o\) {

        if \(o == this\)

            return true;

        if \(o instanceof Map.Entry\) {

            Map.Entry&lt;?,?&gt; e = \(Map.Entry&lt;?,?&gt;\)o;

            if \(Objects.equals\(key, e.getKey\(\)\) &&

                Objects.equals\(value, e.getValue\(\)\)\)

                return true;

        }

        return false;

    }

}

\`\`\`



\*\*说明\*\*



- \`hash\` 这个字段是干嘛的

- 为什么要有一个\`next\`元素





\#\#\# 3. \`Node&lt;K,V&gt;\[\] table;\` 说明

&gt; 按我们的理解，map是一个kv结构，每个Node对象表示的就是一个kv对，那么这个\`table\`应该就是保存所有的kv对的数据结构了

&gt; 

&gt; 为什么会是一个数组？ 怎么根据key来定位kv对在数组中的位置?



\#\#\#\# a. 前置说明



table数组大小，必须为2的n次方，首次使用是初始化，必要时（如添加新的kv对时）可以扩充容量







要了解这个数组的使用过程，最佳的思路就是通过三个方法来定位了



1. \`new HashMap&lt;&gt;\(\)\` 创建对象时，数组的初始化

2. \`put\(k,v\)\` 添加kv时，数组的扩容以及塞值

3. \`get\(k\)\` 通过key获取value时，在数组中的定位



\#\#\#\# b. 创建对象



构造方法如下，主要是设置了阀值，\`loadFactory\` \(后面说其用处\)



\`\`\`java

public HashMap\(int initialCapacity, float loadFactor\) {

    // 参数合法性校验省略

    

    this.loadFactor = loadFactor;

    this.threshold = tableSizeFor\(initialCapacity\);

}



/\*\*

 \* 找到大于等于cap的最小的2的幂.

 \*/

static final int tableSizeFor\(int cap\) {

  int n = cap - 1;

  n \|= n &gt;&gt;&gt; 1;

  n \|= n &gt;&gt;&gt; 2;

  n \|= n &gt;&gt;&gt; 4;

  n \|= n &gt;&gt;&gt; 8;

  n \|= n &gt;&gt;&gt; 16;

  return \(n &lt; 0\) ? 1 : \(n &gt;= MAXIMUM\_CAPACITY\) ? MAXIMUM\_CAPACITY : n + 1;

}

\`\`\`



上面的构造，并没有如我们预期的初始化 \`table\` 数组，接下来看put方法，是否有设置 \`table\`数组呢



---



\#\#\#\# c. 添加kv : \`put\(k,v\)\`



实现如下，逻辑比较复杂，会直接在代码中给出一些注释



\`\`\`java

public V put\(K key, V value\) {

    return putVal\(hash\(key\), key, value, false, true\);

}



/\*\*

 \* Implements Map.put and related methods

 \*

 \* @param hash \(key的hash值，通过hash方法计算\)

 \* @param key the key

 \* @param value the value to put

 \* @param onlyIfAbsent true表示在不存在kv时，才塞入数据

 \* @param evict if false, the table is in creation mode.

 \* @return 返回原来的value（如果之前不存在，返回null）

 \*/

final V putVal\(int hash, K key, V value, boolean onlyIfAbsent,

               boolean evict\) {

    Node&lt;K,V&gt;\[\] tab; Node&lt;K,V&gt; p; int n, i;

    

    // 首先是将tab局部变量指向 table数组

    if \(\(tab = table\) == null \|\| \(n = tab.length\) == 0\) {

        // 当table数组没有初始化时，进行初始化，并返回数组长度

        n = \(tab = resize\(\)\).length;

    } 

    

    

    if \(\(p = tab\[i = \(n - 1\) & hash\]\) == null\) {

        // 根据数组长度和key的hash值，计算出key放入数组的位置

        // 若该位置没有值，则直接创建一个新的Entry（即Node），放在该位置即可

        tab\[i\] = newNode\(hash, key, value, null\);

    } else {

        Node&lt;K,V&gt; e; K k;

        if \(p.hash == hash &&

            \(\(k = p.key\) == key \|\| \(key != null && key.equals\(k\)\)\)\) {

        // 若根据key的hash值，从数组中获取的Entry对象

        // 其key正好是我们指定的key，则直接修改这个Entry的value值即可

            e = p;

        }

        // 下面则表示出现hash碰撞，虽然key的hash值相同，但是这个Entry的key并不是我们指定的key

        else if \(p instanceof TreeNode\) {

            e = \(\(TreeNode&lt;K,V&gt;\)p\).putTreeVal\(this, tab, hash, key, value\);

        } else {

        // 迭代Entry的next节点，知道找到Entry.Key 正好是我们指定的Key为止

            for \(int binCount = 0; ; ++binCount\) {

                if \(\(e = p.next\) == null\) { 

                // 若一直都不存在，则创建一个新的Entry对象，并塞入table

                    p.next = newNode\(hash, key, value, null\);

                    if \(binCount &gt;= TREEIFY\_THRESHOLD - 1\) // -1 for 1st

                        treeifyBin\(tab, hash\);

                    break;

                }

                if \(e.hash == hash &&

                    \(\(k = e.key\) == key \|\| \(key != null && key.equals\(k\)\)\)\)

                    break;

                p = e;

            }

        }

        if \(e != null\) { // existing mapping for key

            V oldValue = e.value;

            if \(!onlyIfAbsent \|\| oldValue == null\)

                e.value = value;

            afterNodeAccess\(e\);

            return oldValue;

        }

    }

    ++modCount;

    if \(++size &gt; threshold\)

        resize\(\);

    afterNodeInsertion\(evict\);

    return null;

}

\`\`\`



逻辑拆解：



- 判断 \`table\`数组是否初始化，否则进行初始化

- 计算key的hash值（通过\`hash\(\)\`方法获取）

- 以key的hash值计算索引，到\`table\`数组中查询\`Node\`节点

  - 若不存在，则新建一个Node节点，塞入该位置

  - 若存在，则继续判断该节点的key是否和传入的key相同or相等\(\`equals\(\)\`方法\)

    - 是，则直接修改这个Node节点的value值即可

    - 否，\*\*表示出现hash碰撞了\*\*

      - 需要遍历Node节点内部的next节点，

      - 直到到next节点为null（新建一个Node节点）

      - 或next节点就是我们希望的节点（更新该节点value值）







到这里就可以解决在介绍Node类结构的两个问题



1. Node中的hash字段干嘛的？



 - hash字段保存的是Key通过\`hash\(\)\`方法计算的值

 - 可以用于判断一个\`Node\`是否为我们查找的节点



2. Node中为什么有next节点

  

  - next节点存的是相同 \`hash\`值的kv键值对，由此可以看出\`HashMap\`的存储结构

  - 当出现hash碰撞时，即对于计算key的hash值相同的Node节点，以链表结构存在

  

!\[结构描述\]\(http://img.blog.csdn.net/20170922215801099?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGl1eXVleWkyNQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast\)



--- 



\#\#\#\# d. table数组初始化

&gt; \`push\(k,v\)\` 包含较多的内容，上面只给出了设计逻辑，具体实现有必要扣一扣，研究下其中一些有意思的点



从上面的的代码可以看出，调用 \`resize\(\)\` 方法进行的初始化（此外这个方法也负责数组的扩容）



源码实现比较长，这里主要关注初始化过程，以以下面这段逻辑进行实例分析



\`\`\`java

Map map = new HashMap&lt;&gt;\(\);

map.put\(xx, xx\);

\`\`\`



对resize方法中一些逻辑配合上面的使用方式进行简化处理, 抽出代码如下



\`\`\`java

final Node&lt;K,V&gt;\[\] resize\(\) {

    Node&lt;K,V&gt;\[\] oldTab = table; // null

    int oldCap = 0;

    int oldThr = 0;

    int newCap, newThr = 0;



    // zero initial threshold signifies using defaults

    newCap = 16; // DEFAULT\_INITIAL\_CAPACITY;

    newThr = 12; // \(int\)\(DEFAULT\_LOAD\_FACTOR \* DEFAULT\_INITIAL\_CAPACITY\);



 

    threshold = newThr;

    @SuppressWarnings\({"rawtypes","unchecked"}\)

    Node&lt;K,V&gt;\[\] newTab = \(Node&lt;K,V&gt;\[\]\)new Node\[newCap\];

    table = newTab;

    return newTab;

}

\`\`\`



上面是简化resize的内部逻辑，单独剥离出初始化 \`table\` 数组的代码块；



\*\*说明\*\*



 - 初始化的数组长度为16

 - threshold 阀值为12 ： \`0.75 \* 数组长度\`



---



\#\#\#\# e. hash方法

&gt; 计算key的hash值，这个直接决定hash碰撞的概率



实现如下



\`\`\`java

static final int hash\(Object key\) {

    int h;

    return \(key == null\) ? 0 : \(h = key.hashCode\(\)\) ^ \(h &gt;&gt;&gt; 16\);

}

\`\`\`



到这里自然就会有一个疑问



\*\*如何根据\`hash\`值与\`table\`数组进行关联，又如何保证碰撞较小？\*\*



这个问题单独成篇，再将这个，这里先记下



---



\#\# 小结



\#\#\# 1. 存储结构



HashMap 的底层数据结构是一个Node数组，配合Node链表的方式进行kv存储



\#\#\# 2. 初始化



数组的初始化延迟在首次向Map中添加元素时进行



默认数组长度为16，阀值为12



阀值定义为： \`The next size value at which to resize \(capacity \* load factor\).\`



\#\#\# 3. 数组长度要求



数组长度要求为2的n次方



\`tableSizeFor\` 方法实现获取正大于数字n的2的整数次幂 （这个实现比较有意思）



\#\#\# 4. 获取Entry对象



如何通过key获取对应的Entry对象呢？



- \`hash\(\)\`方法计算key的hash值

- hash值定位 \`table\`数组中的下标

- 取出数组中的 \`Node\` 节点

  - null，表示不存在

  - 非null，判断Node节点的key是否等同输入key

    - 是直接返回

    - 否则遍历 \`Node\`的 \`next\`节点，直到为null或者找到为止



\#\# 关注更多



扫一扫二维码，关注\`小灰灰blog\`



!\[https://static.oschina.net/uploads/img/201709/22221611\_Fdo5.jpg\]\(https://static.oschina.net/uploads/img/201709/22221611\_Fdo5.jpg\)





\#\# 参考



- \[HashMap源码注解 之 静态工具方法hash\(\)、tableSizeFor\(\)（四）\]\(http://blog.csdn.net/fan2012huan/article/details/51097331\)


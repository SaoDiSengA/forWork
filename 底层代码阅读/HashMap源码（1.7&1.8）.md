![img](http://img1.cppcns.com/images/2021/202111/zpjkkazdvac.jpg)

# HashMap类结构图

```java
public class HashMap<K,V>    extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable
    //AbstractMap和Map其实是重复的
    //AbstractMap实现了map接口,而HashMap也要实现map接口,其实是重复了
    //JDK作者也承认这是写错了
```

补充知识点：

**按位与运算符（&）**

参加运算的两个数据，按二进制位进行“与”运算。

运算规则：0&0=0; 0&1=0; 1&0=0; 1&1=1;

**按位或运算符（|）**

参加运算的两个对象，按二进制位进行“或”运算。

运算规则：0|0=0； 0|1=1； 1|0=1； 1|1=1；

**取反运算符（~）**

参加运算的一个数据，按二进制位进行“取反”运算。

运算规则：~1=0； ~0=1；

**异或运算符(^)**

😊N ^ N = 0; 0 ^ N = N; 异或运算满足交换律和结合律

🤣用于比较两个二进制数的相应位。在执行按位异或运算时，如果两个二进制数的相应位都为1或两个二进制数的相应位都为0，则返回 0；如果两个二进制数的相应位其中一个为1，另一个为0，则返回 1；（相同为0，不同为1）

**位移运算符(<<)和(>>)**

位移运算符分为左位移运算符“<<”和右位移运算符“>>”，分别用于向左和向右执行位移运算。对于X<<N 或 X>>N 形式的运算，含义是将 X 向左或向右移动 N 位，X 的类型可以是 int，uint，long，ulong，byte，sbyte，short 和ushort 。需要注意的是，byte，sbyte，short，和 ushort 类型的值在进行位移操作后值的类型讲自动转换成 int 类型。

**无符号移动(>>>)，没有(<<<)操作哈**

">>>"表示无符号右移，也叫逻辑右移，即若该数为正，则高位补0，而若该数为负数，则右移后高位同样补0

# JDK1.7.80

## 1、常用参数解释

```java
// 默认初始化数组大小，16，左移操作其实就是不断乘2，右移操作其实就是不断除2，因此这里是16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

// 默认最大数组大小，2的30次方，其实就是理解为一个很大的数即可
static final int MAXIMUM_CAPACITY = 1 << 30;

// 扩容时的负载因子(真正用来接收负载因子的变量为：final float loadFactor;)
static final float DEFAULT_LOAD_FACTOR = 0.75f; 

// HashMap的键值对数量
transient int size;

// 下一次要扩容的阀值 16*0.75=12
int threshold; (该值=capacity * load factor)

// 记录HashMap结构修改次数(实现fail-fast机制，迭代过程中有修改，直接抛异常)
transient int modCount; 

// 定义个全局的数组(空数组：static final Entry<?,?>[] EMPTY_TABLE = {};)
//entry类型其实就是数组中缩放的节点类型
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE; 

// 定义解决hash碰撞的掩码值
transient int hashSeed = 0;
```

## 2、节点的结构：Entry

类结构信息

```java
static class Entry<K,V> implements Map.Entry<K,V>
```

基本字段

```java
final K key;  //键
V value;  //值
Entry<K,V> next; // 采用拉链法处理冲突，这个是下一个节点的值
int hash; // 当前这个对象的hash值
```

重写的方法

```java
// equals方法
public final boolean equals(Object o) {
	if (!(o instanceof Map.Entry))
		return false;
    Map.Entry e = (Map.Entry)o;
    Object k1 = getKey();
    Object k2 = e.getKey();
    if (k1 == k2 || (k1 != null && k1.equals(k2))) {
        Object v1 = getValue();
        Object v2 = e.getValue();
        if (v1 == v2 || (v1 != null && v1.equals(v2)))
            return true;            
    }            
    return false;        
}

// hashCode方法   hashcode计算方法，键 异或 值
public final int hashCode() {
    return Objects.hashCode(getKey()) ^ Objects.hashCode(getValue());
}

// toString方法
public final String toString() {      
    return getKey() + "=" + getValue();
}
```

## 3、构造器方法

```java
// 默认构造
// 直接调用了当前类的有参构造方法，只不过默认值为16,0.75
public HashMap() {
    // 初始数组容量16，加载因子0.75
    this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
}

// 带参构造，初始化容量的单参数方法，也是直接跳到有参方法中去。
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}


// 指定构造器
// if块报异常的可以先不看，为了健壮性考虑
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)         
        throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
    
    if (initialCapacity > MAXIMUM_CAPACITY)
        nitialCapacity = MAXIMUM_CAPACITY; //控制最大容量为1<<30大小，一般不可能
    
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " + loadFactor);   
    
    this.loadFactor = loadFactor; //0.75
    
    threshold = initialCapacity; // 保存初始数组大小,threshold是控制下一次扩容的容量的
    
    init(); // 这里是空实现
}
```

## 4、put方法

```java
public V put(K key, V value) {
    //若数组为空，进行初始化
    if (table == EMPTY_TABLE) {
        inflateTable(threshold); //threshold在构造方法中赋值为初始大小
    }
    if (key == null)
        return putForNullKey(value); // 若key为null时，放入逻辑，键为空的只能有一个，值为空的可以有多个
    
    // 计算对应key的hash值
    int hash = hash(key); // hash计算方法，jdk1.7是4次右移
    
    // 计算数组的下标，当前这个对象应该放在数组哪个位置上
    int i = indexFor(hash, table.length);
    
    //循环遍历该数组槽位下的链表，判断新插入的节点key是否重复，重复的话value覆盖，返回oldValue
    //先将table[i]给e，若e为空，则证明当前数组位置没有存东西，那么直接放入
    // 若不为空则需要处理冲突
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        /**
         * 1、两个对象的hash值不相等，那么该两个对象一定不相等，hash值相等，对象不一定等
         * 2、两个对象的hash相等，那么该两个对象不一定相等
         * 下述写法：
         * 1、考虑到上述两条特性，2、防止我们重写的equals方法执行效率慢的问题
         */
        // && 左边的满足，右边是不会执行的，
        //1.7是头插，所以除了判断key相同的进行覆盖，其他的不需要操作
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {//key相同进行value的覆盖
            //若key相同直接进行下面操作，==比较的是地址，若地址相同，一定是同一对象，地址不同也有可能是同一对象，所以最后要调用equals
            V oldValue = e.value;
            e.value = value;//新值覆盖老值
            e.recordAccess(this);
            return oldValue; //直接return，下面addentry不执行。
        }
    }
    // 统计修改HashMap结构的次数，与fail-fast快速失败异常有关
    modCount++;
    // 添加Entry节点包含4个东西。。hash值，键值对，下一个节点
    addEntry(hash, key, value, i);
    return null;
}
```

## 5.addEntry

```java
		/**
         * put中key不存在调用，将Entry对象存入链表
         */
 
         void addEntry(int hash, K key, V value, int bucketIndex) {
 
            
             // 1. 插入前先判断是否需要扩容
            
             // 如果元素个数>=扩容阈值 并且 对应数组下标不为空
             if ((size >= threshold) && (null != table[bucketIndex])) {
            
             //扩容2倍
             resize(2 * table.length);
                 // 重新计算Key对应的hash值
             hash = (null != key) ? hash(key) : 0;
             // 重新计算该Key对应的hash值的存储数组下标位置
             bucketIndex = indexFor(hash, table.length);
     }
 
     //如果不需要扩容，则创建1个新的Entry并放入到数组中
     createEntry(hash, key, value, bucketIndex);
}
```

## 6.createEntry

```java
  	/**
     * 分析2：createEntry(hash, key, value, bucketIndex);
     * 作用： 若容量足够，则创建1个新的数组元素（Entry） 并放入到数组中
     */
void createEntry(int hash, K key, V value, int bucketIndex) {
 
     // 1. 把table中该位置原来的Entry保存
     Entry<K,V> e = table[bucketIndex];
 
     // 2. 在table中该位置新建一个Entry：将原头结点位置（数组上）的键值对 放入到（链表）后1个节点中、将需插入的键值对 放入到头结点中（数组上）-> 从而形成链表
     // 即 在插入元素时，是在链表头插入的，table中的每个位置永远只保存最新插入的Entry，旧的Entry则放入到链表中（即 解决Hash冲突）
     table[bucketIndex] = new Entry<>(hash, key, value, e);
 
     // 3. 哈希表的键值对数量计数增加
     size++;
}
```

## 初始化扩容数组操作：**inflateTable**

```java
private void inflateTable(int toSize) {
    // Find a power of 2 >= toSize(按照2的幂次方扩容)
    int capacity = roundUpToPowerOf2(toSize);
    // 记录下一次要扩容的设置threshold
    threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    // 创建长度位给定容量的数组，以Entry节点构成
    table = new Entry[capacity];
    // 初始化hashSeed值，默认为0，用来检测hash碰撞的
    initHashSeedAsNeeded(capacity);
}

// 获取扩容之后的长度
private static int roundUpToPowerOf2(int number) {
    // assert number >= 0 : "number must be non-negative"; 每次扩容左移一位(乘2)
    return number >= MAXIMUM_CAPACITY ? MAXIMUM_CAPACITY
        : (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
}

/**
* 传入数字i，它将返回小于等于这个数字的一个2的幂次方数
* 如：highestOneBit(15) -> 8, highestOneBit(16) -> 16, highestOneBit(17) -> 16
* 整型最长2^31,故下面计算1、2、4、8、16即可
*/
public static int highestOneBit(int i) {
    // HD, Figure 3-1
    i |= (i >>  1);
    i |= (i >>  2);
    i |= (i >>  4);
    i |= (i >>  8);
    i |= (i >> 16);
    return i - (i >>> 1);
}
```

## 当key为空时的put操作：**putForNullKey**

```java
private V putForNullKey(V value) {
    // 循环找到key为null节点元素，进行值的替换
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    // 在数组0下标添加一个key值为null的节点
    addEntry(0, null, value, 0);
    return null;
}
```

## 😍计算key的[hash](https://so.csdn.net/so/search?q=hash&spm=1001.2101.3001.7020)值方法：**hash**😍

```java
final int hash(Object k) {
    // hash的掩码值，initHashSeedAsNeeded方法中设置
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }
    //先计算hashCode值再进行异或操作
    h ^= k.hashCode();
    /**
     * 为啥下面要进行各种无符号右移之后再进行相应的异或运算呢？
     * 目的：为了解决hash碰撞，使每个节点都能均匀的落在每一个数组卡槽位置
     * 做法：通过无符号右移(高位补0只会为正数)和异或运算，是得数的二进制高低位都能参与运算，
     * 不然后面进行hash时，高位不管怎么变&运算只会都是0，最终参与hash的只是低n位参与hash，导致hash冲突较高
     */
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

## 😍计算数组下标方法：**indexFor**😍

```java
static int indexFor(int h, int length) {
    // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
    // 长度必须是2的n二次幂，这样&出来的下标值才会均匀落在[0,length-1]范围上
    //长度不为2的n二次幂，极大可能计算出来为0，可手动模拟
    // length-1的二进制低n位全为1，&运算比取余%运算效率高 ，手动模拟
    return h & (length-1);
}
```

## 数组扩容操作：**resize**

```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    // 判断数组大小是否已经是最大长度了
    if (oldCapacity == MAXIMUM_CAPACITY) {
        // 为最大长度时，直接更新下一扩容阀值并返回，不进行扩容了
        threshold = Integer.MAX_VALUE;
        return;
    }
    
    // 新生成一个扩容后的数组
    Entry[] newTable = new Entry[newCapacity];
    
    // 将老数组上所有元素转移到扩容后的新数组上去
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    
    // 指针指向新的数组
    table = newTable;
    
    // 计算下一个扩容阀值：新数组长度的0.75倍，或数组最大长度+1
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

// 头插法转移链表节点
//O(n^2)头插
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    // 依次遍历数组每个卡槽
    for (Entry<K,V> e : table) {
        // 依次遍历卡槽下的链表
        while(null != e) {
            // 并发插入引起死循环问题
            Entry<K,V> next = e.next;
            
            // 是否需要重新hash
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            
            // 重新计算数组下标
            int i = indexFor(e.hash, newCapacity);
            
            // 老表节点的next指针指向新表当前数组第一个节点newTable[i](卡槽节点，此时就是个null节点)
            e.next = newTable[i];
            
            // 新数组头结点指向e节点(节点下沉操作)
            newTable[i] = e;
            
            // 老数组头结点指向原先下一个节点
            e = next;
        }
    }
}
```

JDK1.7多线程下扩容可能会导致**死循环**问题，get操作时死循环会引起**CPU 100%问题**

线程A、B put操作时都达到扩容阈值，进行并发扩容，扩容后如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/c574ce2b9dd3496d9758414a861b9ffa.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6JCM6JCM55qEUFA=,size_20,color_FFFFFF,t_70,g_se,x_16)

线程B执行到扩容里transfer对应步骤时让出时间片,A线程执行完整段代码但是还没有将内部的table设置为新的newTable时，线程B继续执行。（头插法插入）

![在这里插入图片描述](https://img-blog.csdnimg.cn/3f3a7aebd31646e6bdffd0f51361d653.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6JCM6JCM55qEUFA=,size_20,color_FFFFFF,t_70,g_se,x_16)

线程B进行对应节点的转移操作，基于e2和next2指针进行转移。产出如下环形结构，在get时产生死循环，CPU 100%问题。

![在这里插入图片描述](https://img-blog.csdnimg.cn/10a0a12fea6c4bc9acbbdf48bf12f1f4.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6JCM6JCM55qEUFA=,size_20,color_FFFFFF,t_70,g_se,x_16)

## get方法

```java
public V get(Object key) {
    if (key == null)
        return getForNullKey();
    
    Entry<K,V> entry = getEntry(key);
    // 返回对应的Entry节点的value
    return null == entry ? null : entry.getValue();
}

// 获取空节点
private V getForNullKey() {
    if (size == 0) {
        return null;
    }
    // 循环遍历找到key为空的节点值返回
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null)
            return e.value; //空key的value
    }
    return null; //否则返回空
}

// 获取非空节点
final Entry<K,V> getEntry(Object key) {
    if (size == 0) {
        return null;
    }
    // 计算hash值
    int hash = (key == null) ? 0 : hash(key);
    // indexFor计算数组下标
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
         e != null;
         e = e.next) {
        Object k;
        // 判断逻辑与put查找逻辑一致
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```

## 其他常用方法

```java
// 通过维护一个size变量获得
public int size() {
    return size;
}
public boolean isEmpty() {
    return size == 0;
}
public boolean containsKey(Object key) {
    return getEntry(key) != null;     //o(n)
}
public boolean containsValue(Object value) {     //o(n^2)
    if (value == null)
        // 返回是否有value为空的节点
        return containsNullValue();
    
    Entry[] tab = table;
    // 遍历数组
    for (int i = 0; i < tab.length ; i++)
        // 遍历槽位下所有的节点
        for (Entry e = tab[i] ; e != null ; e = e.next)
            if (value.equals(e.value))
                return true;
    return false;
}
```


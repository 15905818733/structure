【版权申明】转载
https://blog.csdn.net/u011617742/article/details/54576890

先来复习一下我们常用的几个方法

	public class HashMapTest {
	 
		public static void main(String[] args) {
			// TODO Auto-generated method stub
			HashMap<String, String> hashMap=new HashMap<>();
			//添加方法
			hashMap.put("1", "chris");
	               //遍历方法1_for
			Set<String> keys=hashMap.keySet();
			for(String key:keys){
				System.out.println(key+"="+hashMap.get(key));
			}
			//遍历方法1_iterator(for和iterator实现原理相同)
	                Iterator iter = map.keySet().iterator(); 
	                while (iter.hasNext()) { 
	                String key = iter.next(); 
	                String value = map.get(key); 
	                } 
			//遍历方法2_for
	                Set<Entry<String, String>> entrys= hashMap.entrySet();
			for(Entry<String, String> entry:entrys){
				String key=entry.getKey();
				String value=entry.getValue();
			}
			//遍历方法2_iterator
			Iterator<Entry<String, String>> iterator=hashMap.entrySet().iterator();
			while(iterator.hasNext()){
				Map.Entry<String, String> entry=iterator.next();
				String key=entry.getKey();
				String value=entry.getValue();
			}
			//查询方法
			hashMap.get("1");
			//删除方法
			hashMap.remove("1");		
		}
	 
	}

2 HashMap类图结构

![](https://img-blog.csdn.net/20170116200932306)



3 HashMap数据结构


![](https://img-blog.csdn.net/20170116201004040)

我们知道在Java中最常用的两种结构是数组和模拟指针(引用)，几乎所有的数据结构都可以利用这两种来组合实现。数组的存储方式在内存的地址是连续的，大小固定，一旦分配不能被其他引用占用。它的特点是查询快，时间复杂度是O(1)，插入和删除的操作比较慢，时间复杂度是O(n)，链表的存储方式是非连续的，大小不固定，特点与数组相反，插入和删除快，查询速度慢。HashMap可以说是一种折中的方案吧。



4 HashMap重要概念


![](https://img-blog.csdn.net/20170116201521793)



5 HashMap源码分析

老规矩，按照使用的顺序来分析源码

1.HashMap<String, String> hashMap=new HashMap<>();

	public HashMap() {
	        this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
	    }
	其中默认容量DEFAULT_INITIAL_CAPACITY
	    static final int DEFAULT_INITIAL_CAPACITY = 4;//android N
	默认加载因子DEFAULT_LOAD_FACTOR
	    static final float DEFAULT_LOAD_FACTOR = 0.75f;//android N
	构造函数有几个，但最后都会落到HashMap(int initialCapacity, float loadFactor)
	public HashMap(int initialCapacity, float loadFactor) {  
	        //初始容量不能<0  
	        if (initialCapacity < 0)  
	            throw new IllegalArgumentException("Illegal initial capacity: "  
	                    + initialCapacity);  
	        //初始容量不能 > 最大容量值，HashMap的最大容量值为2^30  
	        if (initialCapacity > MAXIMUM_CAPACITY)  
	            initialCapacity = MAXIMUM_CAPACITY;  
	        //负载因子不能 < 0  
	        if (loadFactor <= 0 || Float.isNaN(loadFactor))  
	            throw new IllegalArgumentException("Illegal load factor: "  
	                    + loadFactor);  
	  
	        // 计算出大于 initialCapacity 的最小的 2 的 n 次方值。  
	        int capacity = 1;  
	        while (capacity < initialCapacity)  
	            capacity <<= 1;  
	          
	        this.loadFactor = loadFactor;  
	        //设置HashMap的容量极限，当HashMap的容量达到该极限时就会进行扩容操作  
	        threshold = (int) (capacity * loadFactor);  
	        //初始化table数组  
	        table = new Entry[capacity];  
	        init();  
	    }

其中涉及到位运算<<,，capacity <<= 1等价于capacity=capacity<<1，表示capacity左移1位
从源码中可以看出，每次新建一个HashMap时，都会初始化一个table数组。table数组的元素为Entry节点

	static class Entry<K,V> implements Map.Entry<K,V> {  
	        final K key;  
	        V value;  
	        Entry<K,V> next;  
	        final int hash;  
	  
	        /** 
	         * Creates new entry. 
	         */  
	        Entry(int h, K k, V v, Entry<K,V> n) {  
	            value = v;  
	            next = n;  
	            key = k;  
	            hash = h;  
	        }  
	        .......  
	    }

其中Entry为HashMap的内部类，它包含了键key、值value、下一个节点next，以及hash值，这是非常重要的，正是由于Entry才构成了table数组的项为链表

2.hashMap.put("1", "chris");

先来看看put的几种分支


![](https://img-blog.csdn.net/20170116204325385)


HashMap通过键的hashCode来快速的存取元素。当不同的对象hashCode发生碰撞时，HashMap通过单链表来解决，将新元素加入链表表头，通过next指向原有的元素。

先说说大概的过程：当我们调用put存值时，HashMap首先会获取key的哈希值，通过哈希值快速找到某个存放位置，这个位置可以被称之为bucketIndex。

对于一个key，如果hashCode不同，equals一定为false，如果hashCode相同，equals不一定为true。

所以理论上，hashCode可能存在冲突的情况，也叫发生了碰撞，当碰撞发生时，计算出的bucketIndex也是相同的，这时会取到bucketIndex位置已存储的元素，最终通过equals来比较，equals方法就是哈希码碰撞时才会执行的方法，所以说HashMap很少会用到equals。HashMap通过hashCode和equals最终判断出K是否已存在，如果已存在，则使用新V值替换旧V值，并返回旧V值，如果不存在 ，则存放新的键值对<K, V>到bucketIndex位置。

下面我们来看看put的源码

	public V put(K key, V value) {  
	        //当key为null，调用putForNullKey方法，保存null于table第一个位置中，这是HashMap允许为null的原因  
	        if (key == null)  
	            return putForNullKey(value);  
	        //计算key的hash值  
	        int hash = hash(key.hashCode());                  ------(1)  
	        //计算key hash 值在 table 数组中的位置  
	        int i = indexFor(hash, table.length);             ------(2)  
	        //从i出开始迭代 e,找到 key 保存的位置  
	        for (Entry<K, V> e = table[i]; e != null; e = e.next) {  
	            Object k;  
	            //判断该条链上是否有hash值相同的(key相同)  
	            //若存在相同，则直接覆盖value，返回旧value  
	            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {  
	                V oldValue = e.value;    //旧值 = 新值  
	                e.value = value;  
	                e.recordAccess(this);  
	                return oldValue;     //返回旧值  
	            }  
	        }  
	        //修改次数增加1  
	        modCount++;  
	        //将key、value添加至i位置处  
	        addEntry(hash, key, value, i);  
	        return null;  
	    }

通过源码我们可以清晰看到HashMap保存数据的过程为：
1)首先判断key是否为null，若为null，则直接调用putForNullKey方法

	private V putForNullKey(V value) {
	        for (HashMapEntry<K,V> e = table[0]; e != null; e = e.next) {
	            if (e.key == null) {
	                V oldValue = e.value;
	                e.value = value;
	                e.recordAccess(this);
	                return oldValue;
	            }
	        }
	        modCount++;
	        addEntry(0, null, value, 0);
	        return null;
	    }

从代码可以看出，如果key为null的值，默认就存储到table[0]开头的链表了。然后遍历table[0]的链表的每个节点Entry，如果发现其中存在节点Entry的key为null，就替换新的value，然后返回旧的value，如果没发现key等于null的节点Entry，就增加新的节点



2)计算key的hashcode（hash(key.hashCode())），再用计算的结果二次hash（indexFor(hash, table.length)），找到Entry数组的索引i，这里涉及到hash算法，最后会详细讲解



3)遍历以table[i]为头节点的链表，如果发现hash，key都相同的节点时，就替换为新的value，然后返回旧的value，只有hash相同时，循环内并没有做任何处理



4)modCount++代表修改次数，与迭代相关，在迭代篇会详细讲解



5)对于hash相同但key不相同的节点以及hash不相同的节点，就增加新的节点（addEntry()）

	void addEntry(int hash, K key, V value, int bucketIndex) {  
	        //获取bucketIndex处的Entry  
	        Entry<K, V> e = table[bucketIndex];  
	        //将新创建的 Entry 放入 bucketIndex 索引处，并让新的 Entry 指向原来的 Entry   
	        table[bucketIndex] = new Entry<K, V>(hash, key, value, e);  
	        //若HashMap中元素的个数超过极限了，则容量扩大两倍  
	        if (size++ >= threshold)  
	            resize(2 * table.length);  
	    } 

这里新增加节点采用了头插法，新节点都增加到头部，新节点的next指向老节点
这里涉及到了HashMap的扩容问题，随着HashMap中元素的数量越来越多，发生碰撞的概率就越来越大，所产生的链表长度就会越来越长，这样势必会影响HashMap的速度，为了保证HashMap的效率，系统必须要在某个临界点进行扩容处理。该临界点在当HashMap中元素的数量等于table数组长度*加载因子。但是扩容是一个非常耗时的过程，因为它需要重新计算这些数据在新table数组中的位置并进行复制处理。

	void resize(int newCapacity) {
	        HashMapEntry[] oldTable = table;
	        int oldCapacity = oldTable.length;
	        if (oldCapacity == MAXIMUM_CAPACITY) {
	            threshold = Integer.MAX_VALUE;
	            return;
	        }
	 
	        HashMapEntry[] newTable = new HashMapEntry[newCapacity];
	        transfer(newTable);
	        table = newTable;
	        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
	    }

从代码可以看出，如果大小超过最大容量就返回。否则就new 一个新的Entry数组，长度为旧的Entry数组长度的两倍。然后将旧的Entry[]复制到新的Entry[].

	void transfer(HashMapEntry[] newTable) {
	        int newCapacity = newTable.length;
	        for (HashMapEntry<K,V> e : table) {
	            while(null != e) {
	                HashMapEntry<K,V> next = e.next;
	                int i = indexFor(e.hash, newCapacity);
	                e.next = newTable[i];
	                newTable[i] = e;
	                e = next;
	            }
	        }
	    }
在复制的时候数组的索引int i = indexFor(e.hash, newCapacity);重新参与计算



3.Iterator iter = map.keySet().iterator();

keySet()方法可以获取包含key的set集合，调用该集合的迭代器可以对key值遍历

	public Set<K> keySet() {
	        Set<K> ks = keySet;
	        if (ks == null) {
	            ks = new KeySet();
	            keySet = ks;
	        }
	        return ks;
	    }

KeySet是HashMap中的内部类，继承AbstractSet，KeySet中获取的迭代器为KeyIterator

	private final class KeySet extends AbstractSet<K> {
	        public Iterator<K> iterator() {
	            return new KeyIterator();
	        }
	        ......
	    }

KeyIterator继承自HashIterator

	private final class KeyIterator extends HashIterator<K> {
	        public K next() {
	            return nextEntry().getKey();
	        }
	    }

	private abstract class HashIterator<E> implements Iterator<E> {
	        HashMapEntry<K,V> next;        // next entry to return
	        int expectedModCount;   // For fast-fail
	        int index;              // current slot
	        HashMapEntry<K,V> current;     // current entry
	 
	        HashIterator() {
	            expectedModCount = modCount;
	            if (size > 0) { // advance to first entry
	                HashMapEntry[] t = table;
	                while (index < t.length && (next = t[index++]) == null)
	                    ;
	            }
	        }
	 
	        public final boolean hasNext() {
	            return next != null;
	        }
	 
	        final Entry<K,V> nextEntry() {
	            if (modCount != expectedModCount)
	                throw new ConcurrentModificationException();
	            HashMapEntry<K,V> e = next;
	            if (e == null)
	                throw new NoSuchElementException();
	 
	            if ((next = e.next) == null) {
	                HashMapEntry[] t = table;
	                while (index < t.length && (next = t[index++]) == null)
	                    ;
	            }
	            current = e;
	            return e;
	        }
	 
	        public void remove() {
	            if (current == null)
	                throw new IllegalStateException();
	            if (modCount != expectedModCount)
	                throw new ConcurrentModificationException();
	            Object k = current.key;
	            current = null;
	            HashMap.this.removeEntryForKey(k);
	            expectedModCount = modCount;
	        }
	    }

4.Iterator<Entry<String, String>> iterator=hashMap.entrySet().iterator();

	public Set<Map.Entry<K,V>> entrySet() {
	        return entrySet0();
	}
	private Set<Map.Entry<K,V>> entrySet0() {
	        Set<Map.Entry<K,V>> es = entrySet;
	        return es != null ? es : (entrySet = new EntrySet());
	}

EntrySet是HashMap内部类，继承AbstractSet，EntrySet中获取的迭代器为EntryIterator

	private final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
	        public Iterator<Map.Entry<K,V>> iterator() {
	            return newEntryIterator();
	        }        ......
	    }

	Iterator<Map.Entry<K,V>> newEntryIterator()   {
	        return new EntryIterator();
	    }

	private final class EntryIterator extends HashIterator<Map.Entry<K,V>> {
	        public Map.Entry<K,V> next() {
	            return nextEntry();
	        }
	    }

	private abstract class HashIterator<E> implements Iterator<E> {
	        HashMapEntry<K,V> next;        // next entry to return
	        int expectedModCount;   // For fast-fail
	        int index;              // current slot
	        HashMapEntry<K,V> current;     // current entry
	 
	        HashIterator() {
	            expectedModCount = modCount;
	            if (size > 0) { // advance to first entry
	                HashMapEntry[] t = table;
	                while (index < t.length && (next = t[index++]) == null)
	                    ;
	            }
	        }
	 
	        public final boolean hasNext() {
	            return next != null;
	        }
	 
	        final Entry<K,V> nextEntry() {
	            if (modCount != expectedModCount)
	                throw new ConcurrentModificationException();
	            HashMapEntry<K,V> e = next;
	            if (e == null)
	                throw new NoSuchElementException();
	 
	            if ((next = e.next) == null) {
	                HashMapEntry[] t = table;
	                while (index < t.length && (next = t[index++]) == null)
	                    ;
	            }
	            current = e;
	            return e;
	        }
	 
	        public void remove() {
	            if (current == null)
	                throw new IllegalStateException();
	            if (modCount != expectedModCount)
	                throw new ConcurrentModificationException();
	            Object k = current.key;
	            current = null;
	            HashMap.this.removeEntryForKey(k);
	            expectedModCount = modCount;
	        }
	    }
显然entrySet()遍历的效率会比keySet()高，因为keySet获取key的集合后，还需要调用get（）方法，相当于遍历两次

5.hashMap.get("1");

	public V get(Object key) {  
	        // 若为null，调用getForNullKey方法返回相对应的value  
	        if (key == null)  
	            return getForNullKey();  
	        // 根据该 key 的 hashCode 值计算它的 hash 码    
	        int hash = hash(key.hashCode());  
	        // 取出 table 数组中指定索引处的值  
	        for (Entry<K, V> e = table[indexFor(hash, table.length)]; e != null; e = e.next) {  
	            Object k;  
	            //若搜索的key与查找的key相同，则返回相对应的value  
	            if (e.hash == hash && ((k = e.key) == key || key.equals(k)))  
	                return e.value;  
	        }  
	        return null;  
	    }

在这里能够根据key快速的取到value除了和HashMap的数据结构密不可分外，还和Entry有莫大的关系，在前面就提到过，HashMap在存储过程中并没有将key，value分开来存储，而是当做一个整体key-value来处理的，这个整体就是Entry对象。同时value也只相当于key的附属而已。在存储的过程中，系统根据key的hashcode来决定Entry在table数组中的存储位置，在取的过程中同样根据key的hashcode取出相对应的Entry对象



6.hashMap.remove("1");

	public V remove(Object key) {
	        Entry<K,V> e = removeEntryForKey(key);
	        return (e == null ? null : e.getValue());
	    }
	
	final Entry<K,V> removeEntryForKey(Object key) {
	        if (size == 0) {
	            return null;
	        }
	        int hash = (key == null) ? 0 : sun.misc.Hashing.singleWordWangJenkinsHash(key);
	        int i = indexFor(hash, table.length);
	        HashMapEntry<K,V> prev = table[i];
	        HashMapEntry<K,V> e = prev;
	 
	        while (e != null) {
	            HashMapEntry<K,V> next = e.next;
	            Object k;
	            if (e.hash == hash &&
	                ((k = e.key) == key || (key != null && key.equals(k)))) {
	                modCount++;
	                size--;
	                if (prev == e)
	                    table[i] = next;
	                else
	                    prev.next = next;
	                e.recordRemoval(this);
	                return e;
	            }
	            prev = e;
	            e = next;
	        }
	 
	        return e;
	    }


6 总结

1.HashMap结合了数组和链表的优点，使用Hash算法加快访问速度，使用散列表解决碰撞冲突的问题，其中数组的每个元素是单链表的头结点，链表是用来解决冲突的



2.HashMap有两个重要的参数：初始容量和加载因子。这两个参数极大的影响了HashMap的性能。初始容量是hash数组的长度，当前加载因子=当前hash数组元素/hash数组长度，最大加载因子为最大能容纳的数组元素个数（默认最大加载因子为0.75），当hash数组中的元素个数超出了最大加载因子和容量的乘积时，要对hashMap进行扩容，扩容过程存在于hashmap的put方法中，扩容过程始终以2次方增长。



3.HashMap是泛型类，key和value可以为任何类型，包括null类型。key为null的键值对永远都放在以table[0]为头结点的链表中，当然不一定是存放在头结点table[0]中。



4.哈希表的容量一定是2的整数次幂，我们在HashMap算法篇会详细讲解
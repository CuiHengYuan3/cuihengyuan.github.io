###                              Vector、ArrayList、LinkedList 的区别



很正很官方的回答是：

Vector 是 Java 早期提供的**线程安全**的动态数组，如果不需要线程安全，并不建议选择，因为同步是有额外开销的。Vector 内部是使用**数组**来保存数据，可以根据需要自动的增加容量。

ArrayList 是应用更加广泛的动态数组实现，**它本身不是线程安全的**，所以性能要好很多。与 Vector 近似，ArrayList 也是可以根据需要调整容量。

LinkedList 顾名思义是 Java 提供的双向链表，所以它不需要像上面两种那样调整容量，**它也不是线程安全的。** 



#### 看看Java集合类

 ![img](https://static001.geekbang.org/resource/image/67/c7/675536edf1563b11ab7ead0def1215c7.png) 

可以看到Collection是所有集合类型的父类，它分出了三个子类，**List ,  Set  ,Queue  .**

List就是有序集合，提供了访问，插入，删除等操作

Set里面的元素是不可以重复的，这是与List最大的区别

Queue是java提供的标准队列的实现。

看图可知，每一种通用的collection类型又被封装为一个抽象，比如AbstractList,AbstractSet等，但具体实现类也不是完全独立的，比如看图可知，LinkedList既是List也是队列。

#### 对应集合类，至少要了解其特性和经典使用场景

**TreeSet** 由于实现了SortedSet,所以支持自然顺序访问，但是添加、删除、包含等操作要相对低效（log(n) 时间）。

**HashSet** 则是利用**哈希算法**，理想情况下，如果哈希散列正常，可以提供常数时间的添加、删除、包含等操作，但是它不保证有序。

**LinkedHashSet**

看看**LinkedHashSet**源码

```java
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {
      ...................

       public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }
}
```

可以看到其构造器调用了其父类HashSet的有三个参数的构造器,如下

```java
  public class HashSet<E>
      extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
{
  HashSet(int initialCapacity, float loadFactor, boolean dummy) {
     //创建了一个LinkedHashMap
      map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
}
```

这就是其不一样的地方，其他的构造是创建了一个hashMap,这里是LinkedHashMap

所以其内部构建了一个记录**插入顺序**的双向链表，因此提供了按照插入顺序遍历的能力，与此同时，也保证了常数时间的添加、删除、包含等操作，这些操作性能略低于 HashSet，因为需要维护链表的开销。
在遍历元素时，HashSet 性能受自身容量影响，所以初始化时，除非有必要，不然不要将其背后的 HashMap 容量设置过大。而对于 LinkedHashSet，由于其内部链表提供的方便，遍历性能只和元素多少有关系



在集合类中，大部分的都不是线程安全的，但是这并不表示就不能用在并发场景中，因为Collections 工具类中，提供了一系列的synchronized 方法。

比如

```java
 static class SynchronizedCollection<E> implements Collection<E>, Serializable {
        private static final long serialVersionUID = 3053995032091335093L;
        final Collection<E> c;
        final Object mutex;

        SynchronizedCollection(Collection<E> var1) {
            this.c = (Collection)Objects.requireNonNull(var1);
            this.mutex = this;
        }

        SynchronizedCollection(Collection<E> var1, Object var2) {
            this.c = (Collection)Objects.requireNonNull(var1);
            this.mutex = Objects.requireNonNull(var2);
        }

        public int size() {
            synchronized(this.mutex) {
                return this.c.size();
            }
        }

        public boolean isEmpty() {
            synchronized(this.mutex) {
                return this.c.isEmpty();
            }
        }

        public boolean contains(Object var1) {
            synchronized(this.mutex) {
                return this.c.contains(var1);
            }
        }

        public Object[] toArray() {
            synchronized(this.mutex) {
                return this.c.toArray();
            }
        }
        }
```

这些实现其实就是简单粗暴的把各种集合方法加上synchronized关键字同步而已。



在Java9中，还提供了集合创建的静态实现

```java
List<String> simpleList = List.of("Hello","world");
```

这样创建出来的集合是不可变的，也不能扩容的，所以线程安全，空间更紧凑，而且语法简洁。




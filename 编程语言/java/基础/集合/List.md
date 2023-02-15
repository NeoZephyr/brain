Collection 表示的数据集合有基本的增、删、查、遍历等方法，但没有定义元素间的顺序或位置，也没有规定是否有重复元素

List 是 Collection 的子接口，表示有顺序或位置的数据集合，增加了根据索引位置进行操作的方法。它有两个主要的实现类，ArrayList 基于数组实现，LinkedList 基于链表实现

## ArrayList

1. ArrayList 实现了 List 接口，继承了 AbstractList 抽象类，底层是数组实现的，并且实现了自增扩容数组大小
2. ArrayList 实现了 Cloneable 接口和 Serializable 接口，可以实现克隆和序列化
3. ArrayList 实现了 RandomAccess 接口，表示能快速随机访问

由于 ArrayList 的数组是基于动态扩增的，所以并不是所有被分配的内存空间都存储了数据。如果采用外部序列化法实现数组的序列化，会序列化整个数组。ArrayList 为了避免这些没有存储数据的内存空间被序列化，内部提供了两个私有方法 writeObject 以及 readObject 来自我完成序列化与反序列化，从而在序列化与反序列化数组时节省了空间和时间。因此使用 transient 修饰数组，防止对象数组被其他外部方法序列化

当 ArrayList 新增元素时，如果所存储的元素已经超过其已有大小，它会计算元素大小后再进行动态扩容，数组的扩容会导致整个数组进行一次内存复制。因此，在初始化 ArrayList 时指定初始大小，有助于减少数组的扩容次数，从而提高系统性能

如果只是在数组末尾添加元素，那么在大量新增元素的场景下，ArrayList 的性能反而比其他 List 集合的性能要好

在使用迭代器遍历时，ArrayList 内部创建了一个内部迭代器，在使用 next() 方法来取下一个元素时，会使用一个记录修改次数的变量 modCount，与迭代器保存的 expectedModCount 变量进行比较，如果不相等则会抛出异常

在循环中调用 List 中的 remove() 方法，只做了 modCount++，而没有同步到 expectedModCount。当再次遍历调用 next() 方法时，发现 modCount 和 expectedModCount 不一致，就抛出了 ConcurrentModificationException 异常

### subList

List.subList 返回的子 List 不是一个普通的 ArrayList，而是原始 List 的视图，会和原始 List 相互影响

```java
List<Integer> list = IntStream.rangeClosed(1, 10).boxed().collect(Collectors.toList());
List<Integer> subList = list.subList(1, 4);
subList.remove(1);
System.out.println(list);
list.add(0);
subList.forEach(System.out::println);
```

遍历 SubList 的时候会先获得迭代器，比较原始 ArrayList modCount 的值和 SubList 当前 modCount 的值。获得了 SubList 后，为原始 List 新增了一个元素修改了其 modCount，所以判等失败抛出 ConcurrentModificationException 异常

List.subList 进行切片操作会导致 OOM。循环中的 1000 个具有 10 万个元素的 List 始终得不到回收，因为它始终被 subList 方法返回的 List 强引用

```java
List<List<Integer>> data = new ArrayList<>();

for (int i = 0; i < 10000; ++i) {
    List<Integer> list = IntStream.rangeClosed(1, 100000).boxed().collect(Collectors.toList());
    data.add(list.subList(0, 1));
}
```

### List 转数组

```java
public Object[] toArray() {
  return Arrays.copyOf(elementData, size);
}

public <T> T[] toArray(T[] a) {
  if (a.length < size)
    return (T[]) Arrays.copyOf(elementData, size, a.getClass());
  System.arraycopy(elementData, 0, a, 0, size);
  if (a.length > size)
    a[size] = null;
  return a;
}
```

### 数组转 List

不能直接使用 Arrays.asList 来转换基本类型数组

```java
// 只能把 int 装箱为 Integer，不可能把 int 数组装箱为 Integer 数组
// Arrays.asList 方法传入的是一个泛型 T 类型可变参数
// 最终 int 数组整体作为了一个对象成为了泛型类型 T
int arr1[] = {1, 2, 3};
List list1 = Arrays.asList(arr1);

Integer arr2[] = {1, 2, 3}
List list2 = Arrays.asList(arr2);
List list3 = Arrays.stream(arr2).boxed().collect(Collectors.toList());
```

Arrays.asList 返回的 List 不支持增删操作。这是因为返回的 List 并不是 java.util.ArrayList，而是 Arrays 的内部类 ArrayList。ArrayList 内部类继承自 AbstractList 类，并没有覆写父类的 add 方法，而父类中 add 方法的实现，就是抛出 UnsupportedOperationException

对原始数组的修改会影响到获得的那个 List，这是因为 ArrayList 直接使用了原始的数组

## LinkedList

1. LinkedList 实现了 List 接口、Deque 接口，同时继承了 AbstractSequentialList 抽象类
2. LinkedList 实现了 Cloneable 和 Serializable 接口，可以实现克隆和序列化
3. LinkedList 也自行实现 readObject 和 writeObject 进行序列化与反序列化

### Queue 接口

先进后出，在尾部添加元素，从头部删除元素

```java
public interface Queue<E> extends Collection<E> {
  boolean add(E e);
  boolean offer(E e);
  E remove();
  E poll();
  E element();
  E peek();
}
```

- add 与 offer 在尾部添加元素。队列满时，add 抛出异常，offer 返回 false
- element 与 peek 返回头部元素。但不改变队列。队列空时，element 抛出异常，peek 返回 null
- remove 与 poll 返回头部元素。并从队列中删除。队列空时，remove 抛出异常，poll 返回 null

### Deque 接口

```java
public interface Deque<E> extends Queue<E> {
  void addFirst(E e);
  void addLast(E e);

  E getFirst();
  E getLast();

  boolean offerFirst(E e);
  boolean offerLast(E e);

  E peekFirst();
  E peekLast();

  E pollFirst();
  E pollLast();

  E removeFirst();
  E removeLast();

  // 从后往前遍历
  Iterator<E> descendingIterator();
}
```

## ArrayList vs LinkedList vs Vector

- Vector 与 ArrayList 作为动态数组，内部元素以数组形式顺序存储，适合随机访问
- LinkedList 进行节点插入、删除比较高效，但是随机访问性能比动态数组慢
- ArrayList 与 LinkedList 为线程不安全的，Vector 为线程安全的，因此效率比 ArrayList 要低
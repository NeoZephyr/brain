## 空容器

返回的是一个静态不可变对象，节省创建新对象的内存和时间开销；注意，返回值不支持修改操作

```java
@SuppressWarnings("unchecked")
public static final <T> List<T> emptyList() {
  return (List<T>) EMPTY_LIST;
}

@SuppressWarnings("unchecked")
public static final <T> Set<T> emptySet() {
  return (Set<T>) EMPTY_SET;
}

@SuppressWarnings("unchecked")
public static final <K,V> Map<K,V> emptyMap() {
  return (Map<K,V>) EMPTY_MAP;
}

@SuppressWarnings("unchecked")
public static <T> Iterator<T> emptyIterator() {
  return (Iterator<T>) EmptyIterator.EMPTY_ITERATOR;
}
```

## 单一对象

将一个单独的对象转换为一个标准的容器接口对象；注意，返回值只能用于读取，不能写入

```java
public static <T> Set<T> singleton(T o) {
  return new SingletonSet<>(o);
}

public static <T> List<T> singletonList(T o) {
  return new SingletonList<>(o);
}

public static <K,V> Map<K,V> singletonMap(K key, V value) {
  return new SingletonMap<>(key, value);
}
```

## 写安全

```java
public static <T> Collection<T> unmodifiableCollection(Collection<? extends T> c);
public static <T> List<T> unmodifiableList(List<? extends T> list);
public static <K,V> Map<K,V> unmodifiableMap(Map<? extends K, ? extends V> m);
public static <T> Set<T> unmodifiableSet(Set<? extends T> s);
public static <K,V> SortedMap<K,V> unmodifiableSortedMap(SortedMap<K, ? extends V> m);
public static <T> SortedSet<T> unmodifiableSortedSet(SortedSet<T> s);
```

## 类型安全

使容器对象的方法在运行时检查类型的正确性

```java
public static <E> Collection<E> checkedCollection(Collection<E> c, Class<E> type);
public static <E> List<E> checkedList(List<E> list, Class<E> type);
public static <K, V> Map<K, V> checkedMap(Map<K, V> m, Class<K> keyType, Class<V> valueType);
public static <E> Set<E> checkedSet(Set<E> s, Class<E> type);
public static <K,V> SortedMap<K,V> checkedSortedMap(SortedMap<K, V> m, Class<K> keyType, Class<V> valueType);
public static <E> SortedSet<E> checkedSortedSet(SortedSet<E> s, Class<E> type);

// OK
List list = new ArrayList<Integer>();
list.add("hello");
System.out.println(list);

// Fail
List list = new ArrayList<Integer>();
list = Collections.checkedList(list, Integer.class);
list.add("hello");
```

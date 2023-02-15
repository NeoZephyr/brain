Set 是 Collection 的子接口，没有增加新的方法，但保证不含重复元素。它有两个主要的实现类，HashSet 和 TreeSet

## HashSet

HashSet 基于哈希表实现，要求键重写 hashCode 方法，效率较高，但元素间没有顺序

### 内部实现

与 HashMap 类似，HashSet 要求元素重写 hashCode 和 equals 方法，且两个对象如果 equals 相同则 hashCode 也必须相同

HashSet 内部通过 HashMap 实现，HashSet 只有键，值都是相同的固定值

#### 添加元素

```java
public boolean add(E e) {
  return map.put(e, PRESENT)==null;
}
```

#### 是否包含元素

```java
public boolean contains(Object o) {
  return map.containsKey(o);
}
```

#### 删除元素
```java
public boolean remove(Object o) {
  return map.remove(o)==PRESENT;
}
```

## TreeSet

TreeSet 基于排序二叉树实现，元素按比较有序，元素需要实现 Comparable 接口，或者创建 TreeSet 时提供一个 Comparator 对象。HashSet 还有一个子类 LinkedHashSet 可以按插入有序。还有一个针对枚举类型的实现类 EnumSet，它基于位向量实现，效率很高
## equals

```java
public boolean equals(Object obj) {
    if (this == obj) {
        return true;
    }
    
    if (!(obj instanceof A)) {
        return false;
    }
    
    // ...
}
```

## hashCode

> 各种类型的哈希值计算如下：
> 1. int 类型的哈希值计算：value
> 2. boolean 类型的哈希值计算：value ? 1 : 0
> 3. byte, char, short, int 类型的哈希值计算：(int)value
> 4. long 类型的哈希值计算：(int)(value ^ (value >>> 32))
> 5. float 类型的哈希值计算：Float.floatToIntBits(value)
> 6. double 类型的哈希值计算：Double.doubleToLongBits(value) 然后进行 long 类型的哈希值计算
> 7. 引用类型的哈希值计算：调用 value 的 hashCode 方法，若为 null 则返回 0
> 8. 数组类型的哈希值计算：计算每一个元素的哈希值

将最终每个域的哈希值合并计算
```java
public int hashCode() {
    result = attr1;
    result = 31 * result + attr2;
    result = 31 * result + attr3;
    
    return result;
}
```

lombok
```java
@EqualsAndHashCode(callSuper = true)
```

## 对象比较

1. 若两个对象相等，equals 返回 true，并且这两个对象的 hashCode 返回值一定相同
2. 若两个对象哈希值相同，equals 不一定返回 `true
3. 默契 equals 方法比较的是对象的 hashCode 值。若 equals 方法被覆盖，hashCode 方法也必须被覆盖
4. compareTo 和 equals 的逻辑保持一致

## clone

### 浅拷贝

```java
public class A implements Cloneable {
    public Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
```

### 深拷贝

```java
public class A implements Cloneable {
    public Object clone() throws CloneNotSupportedException {
        A a = (A) super.clone();
        
        // 拷贝可变成员变量
        a.value = value.clone();
        
        return a;
    }
}
```

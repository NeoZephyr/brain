## 切面概念

### 连接点

程序执行的某个特定位置

### 切入点

用于定位特定的连接点

```java
// 匹配所有 public 方法
execution(public * *(..))

// 匹配指定包下所有类方法，不包含子包
execution(* com.pain.flame.service.*(..))

// 匹配指定包下所有类方法，包含子包
execution(* com.pain.flame.service..*(..))

// 匹配指定类所有方法
execution(* com.pain.flame.service.UserService.*(..))

// 匹配实现特定接口所有类方法
execution(* com.pain.flame.service.GenericService+.*(..))

// 匹配所有 save 开头的方法
execution(* save*(..))
```

### 增强

织入目标类连接点上的一段程序代码

### 切面

切入点与增强组成

## 切面织入

## 织入时期

编译期织入：使用特殊的 java 编译器
类装载期织入：使用特殊的类装载器
动态代理织入：在运行期为目标类添加增强生成子类的方式

## 无法增强问题

1. JDK 动态代理，必须确保要拦截的目标方法在接口中有定义，否则将无法实现拦截
2. CGLIB 动态代理，必须确保要拦截的目标方法可被子类访问，即目标方法必须定义为非私有实例方法
3. 方法内部之间的调用，不会使用被增强的代理类，而是直接调用未被增强原类的方法
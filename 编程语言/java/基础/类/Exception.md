## Error

绝大部分的 Error 都会导致程序处于非正常的、不可恢复状态。既然是非正常情况，所以不便于也不需要捕获。常见的比如 OutOfMemoryError、StackOverflowError、NoClassDefFoundError 等

## 异常

1. 编译时异常：在编译前必须处理，或称为可检查异常，必须显式地进行捕获处理。常见的如 IOException 等
2. 运行时异常：可以在运行时处理，或称为不可检查异常。常见的如 NullPointerException, ArrayIndexOutOfBoundsException 等

通常情况下，程序中自定义的异常应为可检查异常，以便最大化利用 Java 编译器的编译时检查

异常实例的构造十分昂贵。这是由于在构造异常实例时，Java 虚拟机便需要生成该异常的栈轨迹。该操作会逐一访问当前线程的 Java 栈帧，并且记录下各种调试信息，包括栈帧所指向方法的名字，方法所在的类名、文件名，以及在代码中的第几行触发该异常

## 异常捕获

在编译生成的字节码中，每个方法都附带一个异常表。异常表中的每一个条目代表一个异常处理器，并且由 from 指针、to 指针、target 指针以及所捕获的异常类型构成。这些指针的值是字节码索引，用以定位字节码。其中，from 指针和 to 指针标示了该异常处理器所监控的范围，例如 try 代码块所覆盖的范围。target 指针则指向异常处理器的起始位置，例如 catch 代码块的起始位置

当程序触发异常时，Java 虚拟机会从上至下遍历异常表中的所有条目。当触发异常的字节码的索引值在某个异常表条目的监控范围内，Java 虚拟机会判断所抛出的异常和该条目想要捕获的异常是否匹配。如果匹配，Java 虚拟机会将控制流转移至该条目 target 指针指向的字节码

如果遍历完所有异常表条目，Java 虚拟机仍未匹配到异常处理器，那么它会弹出当前方法对应的 Java 栈帧，并且在调用者中重复上述操作。在最坏情况下，Java 虚拟机需要遍历当前线程 Java 栈上所有方法的异常表

finally 代码块的编译比较复杂。当前版本 Java 编译器的做法，是复制 finally 代码块的内容，分别放在 try-catch 代码块所有正常执行路径以及异常执行路径的出口中

Java 7 的 try-with-resources 语法糖，程序可以在 try 关键字后声明并实例化实现了 AutoCloseable 接口的类，编译器将自动添加对应的 close() 操作。try-with-resources 还会使用 Supressed 异常的功能（将一个异常附于另一个异常之上，抛出的异常可以附带多个异常的信息），来避免原异常被消失

Java 7 还支持在同一 catch 代码块中捕获多种异常。实际实现非常简单，生成多个异常表条目即可

## Finally 异常覆盖

因为一个方法无法出现两个异常，finally 中的异常会覆盖 try 代码块中的异常

```java
class Resource implements AutoCloseable {
    public void read() {
        throw new RuntimeException("read error");
    }

    public void close() {
        throw new RuntimeException("close error");
    }
}
```

```java
Resource resource = new Resource();

try {
    resource.read();
} finally {
    resource.close();
}
```

finally 中捕获异常
```java
Resource resource = new Resource();

try {
    resource.read();
} finally {
    try {
        resource.close();
    } catch (Exception ex) {
        System.out.println(ex.getMessage());
    }
}
```

添加附加异常
```java
Resource resource = new Resource();
Exception e = null;

try {
    resource.read();
} catch (Exception ex) {
    e = ex;
} finally {
    try {
        resource.close();
    } catch (Exception ex) {
        if (e != null) {
            e.addSuppressed(ex);
        } else {
            e = ex;
        }
    }
}

if (e != null) {
    throw e;
}
```

try-with-resources 模式
```java
try (Resource resource = new Resource()) {
    resource.read();
}
```

## 静态异常实例

异常定义为静态变量导致异常信息固化
```java
try {
    run1();
} catch (Exception ex) {
    ex.printStackTrace();
}

try {
    run2();
} catch (Exception ex) {
    ex.printStackTrace();
}

public static void run1() throws Exception {
    throw Exceptions.INTERNAL_EXCEPTION;
}

public static void run2() throws Exception {
    throw Exceptions.INTERNAL_EXCEPTION;
}

static class Exceptions {
    public static Exception INTERNAL_EXCEPTION = new RuntimeException("internal exception");

    public static Exception internalException() {
        return new RuntimeException("internal exception");
    }
}
```

## 线程池异常

抛出异常导致老线程退出，线程池只能重新创建一个线程。如果每个异步任务都以异常结束，那么线程池可能完全起不到线程重用的作用。因为没有手动捕获异常进行处理，ThreadGroup 会进行未捕获异常的默认处理，向标准错误输出打印了出现异常的线程名称和异常信息

```java
ExecutorService service = Executors.newFixedThreadPool(1, new ThreadFactoryBuilder()
                .setNameFormat("test-%d")
                .build());
IntStream.rangeClosed(1, 10).forEach(i -> service.execute(() -> {
    try {
        Thread.sleep(500);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    if (i == 5) {
        throw new RuntimeException("error");
    }
    System.out.println("Finish " + i + ", thread: " + Thread.currentThread());
}));
service.shutdown();
service.awaitTermination(1, TimeUnit.MINUTES);
```

在声明线程池时自定义线程池的未捕获异常处理程序
```java
ExecutorService service = Executors.newFixedThreadPool(1, new ThreadFactoryBuilder()
                .setNameFormat("test-%d")
                .setUncaughtExceptionHandler((thread, throwable) -> {
                    System.out.printf("Thread %s get exception %s\n", thread, throwable);
                })
                .build());
```

设置全局的默认未捕获异常处理程序
```java
static {
    Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> {
        System.out.printf("Thread %s get exception %s\n", thread, throwable);
    });
}
```

submit 方式提交任务，出现异常不会导致线程退出
```java
ExecutorService service = Executors.newFixedThreadPool(1, new ThreadFactoryBuilder()
                .setNameFormat("test-%d")
                .build());
List<? extends Future<?>> tasks = IntStream.rangeClosed(1, 10).mapToObj(i -> service.submit(() -> {
    try {
        Thread.sleep(200);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    if (i == 5) {
        throw new RuntimeException("error");
    }
    System.out.println("Finish " + i + ", thread: " + Thread.currentThread());
})).collect(Collectors.toList());

tasks.forEach(task -> {
    try {
        task.get();
    } catch (Exception e) {
        e.printStackTrace();
    }
});
```

## 异常处理

1. 尽量不要捕获类似 Exception 这样的通用异常，而是应该捕获特定异常
2. 不要生吞异常，这样很可能会导致出现的问题难以诊断
3. 应该将异常输出到日志系统中，而不是输出到标准出错
4. 对于不清楚如何处理的异常，应该保留原有异常的 cause 信息，直接抛出或者构建新的异常。在更高层面有了清晰的业务逻辑，再做处理
5. try/catch 会产生额外的性能开销，应该只捕获有必要的代码段
6. 每实例化一个 Exception，都会对当时的栈进行快照，这是一个相对比较重的操作。如果发生的非常频繁，这个开销可就不能被忽略了

## 异常设计

1. 考虑是否需要定义成 Checked Exception，这种类型设计的初衷更是为了从异常情况恢复
2. 在保证诊断信息足够的同时，避免包含敏感信息，以免导致潜在的安全问题
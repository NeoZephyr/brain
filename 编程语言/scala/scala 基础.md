## 变量

```scala
var name: String = "pain"
var age: Int = 28
val height: Float = 1.65f

var address = "wuhan"
```

```scala
lazy val res = sum(10, 20)
```

```scala
var name = StdIn.readLine()
var price = StdIn.readDouble()
```

插值
```scala
println(s"name=${name}")
println(f"price=${price}%.2f")
println(raw"name=${name} \n")
```

自定义转换规则
```scala
implicit def transform(value: Double): Int = {
    return value.toInt()
}

val value: Int = 5.0
```

## 循环

```scala
for (i <- 1 to 5) {
    println(s"i = ${i}")
}

for (i <- 1 until 5) {
    println(s"i = ${i}")
}

for (i <- 1 to 5 if i != 2) {
    println(s"i = ${i}")
}
```

```scala
for (i <- Range(1, 5, 2)) {
    println(s"i = ${i}")
}
```

```scala
Breaks.breakable {
    for (i <- 1 to 5) {
        if (i == 2) {
            Breaks.break()
        }

        println(s"i = ${i}")
    }
}
```

## 函数

```scala
def max(x: Int, y: Int): Int = {
    if (x > y) {
        return x
    } else {
        return y
    }
}

println(max(x = 100, y = 200))
```

```scala
def sum(numbers: Int*): Int = {
    var result = 0
    for (number <- numbers) {
        result += number
    }

    return result
}
```

```scala
def hello(name: String = "pain") : Unit = {
    println(s"hello, ${name}")
}

hello()
hello(name = "jack")
```

函数作为返回值
```scala
def hello(name: String = "pain") : Unit = {
    println(s"hello, ${name}")
}

def getHelloFunc() = {
    hello _
}

val helloFunc = getHelloFunc()
helloFunc("jack")
```

函数作为参数
```scala
def f(inner : (Int) => Int) : Int = {
    return inner(10) + 10
}

def inner(num : Int) : Int = {
    return num
}

f(inner)

f((x) => {
    println(x)
})
```

闭包
```scala
def f1(i : Int) = {
    def f2(j : Int) {
        return i * j
    }

    f2 _
}

f1(1)(2)
```

函数简化过程
```scala
def f(add : (Int, Int) => Int) : Int = {
    add(10, 20)
}

println(f((x: Int, y: Int) => { x + y }))

// 自动推断出类型
println(f((x, y) => { x + y }))

// 如果只有一个参数，省略括号
println(f((x, y) => x + y))

// 使用占位符
println(f(_ + _))
```

偏函数
```scala
def hello: PartialFunction[String, String] = {
    case "jack" => "jackson"
    case "johm" => "johnson"
    case _ => "no no no"
}
```


## 集合

数组
```scala
val players: Array[String] = Array("pain", "jack", "slog")

println(players(0))
println(players.mkString("|"))

for (player <- players) {
    println(player)
}

players.foreach(println)
```

```scala
var players: ArrayBuffer[String] = ArrayBuffer("pain", "tack")
players(0) = "jelin"
players.insert(0, "fony")
players += "taylor"
```

列表
```scala
var lines = List("spark streaming", "kafka streaming", "kafka spark", "spark hbase", "spark hive", "spark sql")
lines = lines.filter(lines => !lines.contains("hbase"))

// var nestedWords = lines.map(e => e.split(" "))
// var words = nestedWords.flatten

var words = lines.flatMap(e => e.split(" "))
val wordToListMap = flatWords.groupBy(word => word)
val wordToCount = wordToListMap.map(e => (e._1, e._2.size))
val sortWordList = wordToCount.toList.sortWith((left, right) => {left._2 > right._2})
println(sortWordList.take(3))
```

```scala
var nums = List(1, 2, 3, 4, 5)
println(nums.reduce((left, right) => left - right))
println(nums.reduce(_ + _))
println(nums.fold(100)(_ + _))
```

```scala
val ints = List(1, 2, 3, "hello").collect {
  case i: Int => i + 10
}

println(ints)
```

```scala
val tuples = list1.zip(list2)
```

```scala
val unionList = list1.union(list2)
```

```scala
val intersectList = list1.intersect(list2)
```

```scala
val diffList = list1.diff(list2)
```

集合
```scala
val set = Set(1, 1, 2, 3)
```

映射
```scala
val map = Map("jack" -> 100, "pain" -> 90)
```

元组
```scala
val tuple = ("jack", "28", "19000")
val (name, age, salary) = tuple
```

## 模式匹配
```scala
grade match {
    case "A" => println("Excellent")
    case "B" => println("Good")
    case _ if (name == "pain") => println("Good")
    case _ => println("Ok")
}
```

```scala
array match {
  case Array(_) => println("only one element")
  case Array(_, _) => println("two element")
  case Array(_*) => println("many element")
  case _ => println("unknown")
}
```

```scala
list match {
    case "jack"::Nil => println("one elem")
    case x::y::Nil => println(s"two elem, ${x}, ${y}")
    case "jack"::tail => println("many elem")
    case _ => println("unknow elem")
}
```

```scala
obj match {
    case x: Int => println("int")
    case s: String => println("string")
    case m: Map[_, _] => m.foreach(println)
    case _ => println("unknow type")
}
```

```scala
class Person
case class Student(name: String)
case class Teacher(name: String)

person match {
    case Student(name) => println("student")
    case Teacher(name) => println("teacher")
    case _ => println("other")
}
```

```scala
try {
    10 / 0
} catch {
    case e: ArithmeticException => println("can not be 0")
    case e: Exception => println(e.getMessage)
} finally {}
```

## 类
```scala
class User(name: String) {
    var name: String = _
    var age: Int = _

    println(s"name = ${name}")

    def this() {
        this("jack")
    }

    def this(name: String, age: Int) {
        this(name)
        this.age = age
    }

    def printName() : Unit = {
        println(name)
    }
}

var user: User = new User("jack")
```

```scala
abstract class Monitor {
    def alter()
}

class KafkaMonitor extends Monitor {
    override def alter: Unit = {}
}
```

抽象属性、重写属性
```scala
abstract class Person(name: String) {
    var name: String = _
    var gender: String
    val address: String = "china"
}

class Man(name: String) extends Person(name) {
    var gender: String = "male"
    override val address: String = ""

    def this() {
        this(name)
    }
}
```

```scala
class User(var name: String) {}

var user = new User("hello")
println(user.name)
```

伴生对象与伴生类
```scala
class User {
    var name: String = "pain"
}

object User {    
    def apply(name: String): User = new User
}
```

```scala
val user = User("jack")
```

## trait
```scala
trait Sleep {}
trait Eat {}

class User extends Persion with Sleep with Eat {}
```

```scala
trait InsertOption {
    def insert() {
        println("insert")
    }
}

trait FileSystem extends InsertOption {
    override def insert() {
        println("file")
        super[InsertOption].insert()
    }
}
```

```scala
val mysql = new Mysql() with InsertOption
mysql.insert()
```

## 泛型
```scala
def test[T <: User](t: T): Unit = {}

test[User](new User())

class Test[+User] {}
```
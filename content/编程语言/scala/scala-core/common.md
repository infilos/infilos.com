---
type: docs
title: "常见问题"
linkTitle: "常见问题"
weight: 37
---

## 根据类的不同属性进行排序-1

需要对类的集合按照不同的字段进行排序，排序的规则可以指定：

```scala
case class Item(good:String, bad:String, gross:Int, warn:String)

trait BaseSort{
  def sort[A, B: Ordering](data: List[A], desc: Boolean)(measure: A => B): List[A] = {
    val baseOrdering = Ordering.by(measure)
    val ordering = if (desc) baseOrdering.reverse else baseOrdering
    data.sorted(ordering)
  }
}

trait ItemsSort extends BaseSort{
  def sortItems(items:List[Item], by:String, desc:Boolean): List[Item] = by match {
    case "good" => sort(items, desc=desc)(_.good)
    case "bad" => sort(items, desc=desc)(_.bad)
    case "gross" => sort(items, desc=desc)(_.gross)
    case "warn" => sort(items, desc=desc)(_.warn)
    case _ => items
  }
}

val items:List[Item] = List(Item("a","a",1,"a"),Item("b","b",2,"b"),Item("c","c",3,"c"))
sortItems(items, "good",desc = true).foreach(println)
```

## 根据类的不同属性进行排序-2

另一种方式是首先预定义需要的排序规则，然后在需要的位置做为隐式参数引入：

```scala
case class Item2(id:Int, firstName:String, lastName:String)

object Item2{
  // 注意，因为`Ordering[A]`不是逆变的，如果`Item`的子类想要使用该排序方式，则必须声明为参数化类型
  implicit def orderByName[A <: Item2]: Ordering[A] =
    Ordering.by(e => (e.firstName,e.lastName))

  val orderingById: Ordering[Item2] = Ordering.by(_.id)
}

object CustomClassSort2 extends App{
  val items:List[Item2] = List(Item2(2,"ccc","ddd"),Item2(1,"aaa","bbb"))

  import Item2.orderByName						// 直接引入隐式参数
  items.sorted.foreach(println)

  implicit val ording = Item2.orderingById		// 引入排序规则后定义为隐式参数
  items.sorted.foreach(println)
}
```

## 隐式转换

## 将类自动转换为元组

有时候需要将一些`case class`自动转换为元组以方便处理：

```scala
case class Foo(a:String, b:String)
implicit def asTuple(foo:Foo):(String,String) = Foo.unapply(foo).get
val foo1 = Foo("aa","bb")
val (a:String,b:String):(String,String) = foo1
```

## 不可过度使用元组

对元组过度的使用会使代码难于理解，特别是元素特别多的元组，比如：

```scala
def bestByName(query:String, actors:List[(Int, String, Double)]) = 
  actors.filter { _._2 contains query}
		.sortBy { _._3}
		.map { _._1}
		.take(10)
```

而是应该讲使用频繁的元组作为一个`case class`，并且将各阶段的中间处理过程进行易于理解的命名：

```scala
case class Actor(id:Int, name:String, score:Double)

def bestByName(query:String, actors:List[Actor]) = {
  val candidates = actors.filter{ _.name contains query}
  val ranked = canditates.sortBy { _.score }
  val best = ranked take 10
  best map { _.id }
}
```

## 可变数据类型的选择

在一个函数或者私有类中时，如果一个可变的数据类型能够有效的缩减代码，这时就可以使用可变数据类型，比如`var`或者`colleciton.mutable`，因为没有外部的动作可以对他们造成改变。

## 函数重复参数传入

比如定义一个方法：

```scala
def echo(args:String*) = args.forearh(println)
```

该函数能够接受一个或多个 String 类型的参数，比如：

```scala
echo("aa")
echo("bb","cc")
```

这个`String*`实际上是一个`Array[String]`，但是当传入一个已存在的序列时，需要先将其展开：

```scala
val seq:Seq[String] = Seq("aa","bb","cc")
echo(seq:_*)
```

这个标注告诉编译器将序列的每个元素当做一个参数，而不是将整个序列做为一个单一的参数传入。直接传入序列将会报错。

## 并行集合

### 为并行集合指定线程数

[Configuring Parallel Collections](http://docs.scala-lang.org/overviews/parallel-collections/configuration.html)

```scala
scala> import scala.collection.parallel._
scala> val pc = mutable.ParArray(1, 2, 3)
scala> pc.tasksupport = new ForkJoinTaskSupport(newscala.concurrent.forkjoin.ForkJoinPool(2))
scala> pc map { _ + 1 }
res0: scala.collection.parallel.mutable.ParArray[Int] = ParArray(2, 3, 4)
```

## 异常

### NoSuchMethodError

此类异常多为使用了错误版本的 JDK 或 Scala。

## 获取代码运行时间

```scala
val t1 = System.nanoTime

/* your code */

val duration = (System.nanoTime - t1) / 1e9d
```

## 为Future添加一个基于时间的监控器

```scala
import scala.concurrent.duration._
import java.util.concurrent.{Executors, ScheduledThreadPoolExecutor}
import scala.concurrent.{Future, Promise}

val f: Future[Int] = ???

val executor = new ScheduledThreadPoolExecutor(2, Executors.defaultThreadFactory(), AbortPolicy)

def withDelay[T](operation: ⇒ T)(by: FiniteDuration): Future[T] = {
  val promise = Promise[T]()
  executor.schedule(new Runnable {
    override def run() = {
      promise.complete(Try(operation))
    }
  }, by.length, by.unit)
  promise.future
}

Future.firstCompletedOf(Seq(f, withDelay(println("still going"))(30 seconds)))
Future.firstCompletedOf(Seq(f, withDelay(println("still still going"))(60 seconds)))
```

## logback 避免日志重复

```scala
<logger name="data-logger" level="info" additivity="false">
```

Loggers are hierarchical, and any message sent to a logger will be sent to all its ancestors by default. You can disable this behavior by setting additivity=false.

## OkHttp异步请求

**BUG**：大量请求之后资源耗尽导致服务不可用，虽然已经关闭了`response`。

```scala
import java.util.concurrent.TimeUnit
import okhttp3._

trait OkHttpBuilder {

  val httpClient: OkHttpClient = new OkHttpClient().newBuilder()
    .connectTimeout(10, TimeUnit.SECONDS)
    .writeTimeout(10, TimeUnit.SECONDS)
    .readTimeout(45, TimeUnit.SECONDS)
    .retryOnConnectionFailure(true)
    .followSslRedirects(true)
    .followRedirects(true)
    .build()

  def cloneHttpClient() = httpClient.newBuilder().build()
}

object OkHttpBuilder extends OkHttpBuilder
```

```scala
private def asyncDownload(src: String, superior: ActorRef) = {
    try {
      val client: OkHttpClient = OkHttpBuilder.cloneHttpClient()
      val request = new Request.Builder().url(src).get().addHeader("User-Agent", web).build()
      client.newCall(request).enqueue(new Callback {
        override def onFailure(call: Call, e: IOException): Unit = {
          logger.error(s"ImageProcessor.DownloadErr - 1: $src, ${e.getMessage}")
          superior ! Redownload(src)
        }
        override def onResponse(call: Call, response: Response): Unit = {
          response.isSuccessful match {
            case false =>
              logger.error(s"ImageProcessor.DownloadErr - 2: $src, ${response.code()}")
              superior ! Discard(src)
              response.close()
            case true => try {
              val stream: InputStream = response.body().byteStream()
              val bytes: Array[Byte] = IOUtils.toByteArray(stream)
              save(src, bytes) match {
                case Some(path) =>
                  upload(path) match {
                    case Some(pathU) => superior ! Oss(src, pathU)
                    case None        => superior ! Redownload(src)
                  }
                case None => superior ! Discard(src)
              }
            } catch {
              case NonFatal(e) =>
                logger.error(s"ImageProcessor.DownloadErr - 3: $src, ${e.getMessage}")
                superior ! Redownload(src)
            } finally {
              response.close()
            }
          }
        }
      })
    } catch {
      case ex: IOException =>
        logger.error(s"ImageProcessor.DownloadErr- IOException: $src, ${ex.getMessage}")
        superior ! Redownload(src)
      case NonFatal(e) =>
        logger.error(s"ImageProcessor.DownloadErr - 0: $src, ${e.getMessage}")
        superior ! Redownload(src)
    }
  }
```

## 获取系统包路径

```bash
java -XshowSettings:properties -version		// java 8
```

或：

```java
public class PrintLibPath{
    public static void main(String[] args){
        System.out.println(System.getProperty("java.library.path"));
    }
}
```

```bash
javac PrintLibPath.java
java -cp /path PrintLibPath
/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib	# centos 6.5
```


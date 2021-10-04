---
type: docs
title: "Future-Promise"
linkTitle: "Future-Promise"
weight: 7
---

## 介绍

`Promise`是一个可以被一个值或一个异常完成的对象。并且只能完成一次，完成后就不再可变。称为单一赋值变量，如果再次赋值则会引发异常。

`Promise`值的类型为`Promise[T]`，可以通过其伴生对象中的`Promise.apply()`创建一个实例：

```scala
def apply[T](): Promise[T]
```

调用该方法后会立即返回一个`Promise`实例，它是非阻塞的。当新的`Promise`对象被创建后，不会包含值或异常，通过`success`和`failure`方法分别赋值为**值**或**异常**。

通过`complete`可以提供一个`Try[T]`实例来填充`Promise`，这时`Promise`的值是一个值还是一个异常，取决于`Try[T]`最终是一个包含值的`Sucess`还是一个包含异常的`Failure`对象。

每个`Promise`对象都明确对应一个`Future`对象，通过`future`方法获取关联的`Future`对象，并且无论调用该方法多少次，都会放回相同的`Future`对象。

与`success/failure/complete`对应的方法是`trySuccess/tryFailure/tryCompletee`，这些方法会尝试对`Promise`进行赋值并返回赋值操作是否成功的布尔值。而基本的`success/failure/complete`操作返回的是则是对原`Promise`的引用。

## object Promise

`Promise`的伴生对象，一共定义了四种构造器。

### apply

```scala
def apply[T](): Promise[T] = new impl.Promise.DefaultPromise[T]()
```

它不接受然和参数，创建一个能够被类型为`T`的值完成的`Promise`对象。即创建它时只需要设置预期被完成的值的类型：

```scala
val promise: Promise[User] = Promise[User]()
```

### fromTry

```
def fromTry[T](result: Try[T]): Promise[T] = new impl.Promise.KeptPromise[T](result)
```

提供一个`Try[T]`类型的值，并返回一个被完成后的`Promise`对象。这个结果`Promise`中是一个值还是异常，取决于传入的`Try[T]`最终是一个`Success`还是一个`Failure`。

### failed

```scala
def failed[T](exception: Throwable): Promise[T] = fromTry(Failure(exception))
```

通过传入一个异常创建一个被该异常完成的`Promise`对象。

### successful

```scala
def successful[T](result: T): Promise[T] = fromTry(Success(result))
```

通过传入一个值创建一个被该值完成的`Promise`对象。

## trait Promise

### isCompleted

判断该`Promise`是否已被完成，返回一个布尔值。

### complete & tryComplete

```scala
def complete(result: Try[T]): this.type =
    if (tryComplete(result)) this 
    else throw new IllegalStateException("Promise already completed.")

def tryComplete(result: Try[T]): Boolean
```

`tryComplete`方法尝试通过传入的`Try[T]`来完成该`Promise`，返回一个该操作成功失败的布尔值。**需要注意的是，虽然返回的是一个布尔值，但这个布尔值表示，该 Promise 之前没有被完成并且已经被当前的操作完成，或在调用该方法之间就已经被完成。**

因此，在`complete`方法中，通过传入一个`Try[T]`来完成一个`Promise`，实际上会在内部调用`tryComplete`，如果``tryComplete`返回`true`，表示该`Promise`还没有被完成并通过这次操作成功完成，同时返回该`Promise`的引用，否则则报错已经被完成过。

### completeWith & tryCompleteWith

```scala
final def completeWith(other: Future[T]): this.type = tryCompleteWith(other)
final def tryCompleteWith(other: Future[T]): this.type = {
    other onComplete { this tryComplete _ }
    this
  }
```

与`complete`和`tryComplete`类似，不过他接收的是一个`Future[T]`而不是一个`Try[T]`。

### success & trySuccess

```scala
def success(@deprecatedName('v) value: T): this.type = complete(Success(value))
def trySuccess(value: T): Boolean = tryComplete(Success(value))
```

`success`方法通过一个值来完成`Promise`，而`trySuccess`则首先将传入的值包装为一个`Success[T	]`然后调用前面的`tryComplete`方法，尝试完成`Promise`并返回一个布尔值。

### failure & tryFailure

```scala
def failure(@deprecatedName('t) cause: Throwable): this.type = complete(Failure(cause))
def tryFailure(@deprecatedName('t) cause: Throwable): Boolean = tryComplete(Failure(cause))
```

与`success`和`trySuccess`处理过程相同，只是通过一个异常而不是一个值来完成`Promise`

### future

```scala
def future: Future[T]
```

获取该`Promise`对应的`Future`。

## 总结

- 一个`Promise`有两种构造方式，构造为未完成的、构造为已完成的
- 一个`Promise`只能调用完成方法一次，无论是哪个完成方法，再次调用将抛出已完成异常
- 一个`Promise`只与一个`Future`对应，无论调用多少次`future`方法都会返回相同的`Future`


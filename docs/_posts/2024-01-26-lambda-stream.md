---
title: "java8中的lambda表达式与Stream类库"
categories:
  - java
tags:
  - java
  - lambda表达式
---

## lambda表达式

### 函数式接口

有且仅有一个抽象方法的接口（可以有默认方法和静态方法）

常用的函数接口：

- Consumer: 消费型接口，有入参，无返回

  ```java
  @FunctionalInterface
  public interface Consumer<T> {
    void accept(T t);

    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
  }

  Consumer consumer = System.out::println;
  ```

- Supplier: 供给型接口，有返回，无入参

  ```java
  @FunctionalInterface
  public interface Supplier<T> {
    T get();
  }

  Supplier spplier=ArrayList::new;
  ```

- Function: 函数型接口，有入参，有返回，做转换

  ```java
  @FunctionalInterface
  public interface Function<T, R> {
    R apply(T t);

    default <V> Function<V, R> compose(Function<? super V, ?   extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }
  
    default <V> Function<T, V> andThen(Function<? super R, ?   extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }
  
    static <T> Function<T, T> identity() {
        return t -> t;
    }
  }
  
  Function function=String::valueOf;
  ```

- Predicate: 断言型接口，有入参，返回为boolean类型

  ```java
  @FunctionalInterface
  public interface Predicate<T> {
    boolean test(T t);

    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }

    default Predicate<T> negate() {
        return (t) -> !test(t);
    }

    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }

    static <T> Predicate<T> isEqual(Object targetRef) {
        return (null == targetRef)
                ? Objects::isNull
                : object -> targetRef.equals(object);
    }
  }

  Predicate predicate = Objects::isNull;
  ```

### lambda表达式是什么？

1. 从语法上看，lambda表达式是一个类型为函数式接口的对象，如：`Function function=String::valueOf`;  这里lambda表达式是一个对象，被分配给一个Function类型的变量。函数接口类型的参数都可以传入lambda表达式作为参数。
2. 从语义上看，lambda表达式是一段行为逻辑，lambda表达式是一个代表行为的对象。

### lambda表达式中的外部变量

lambda表达式中的外部变量应为final或有效的final变量

> You may be asking yourself why local variables have these restrictions. First, there’s a key difference in how instance and local variables are implemented behind the scenes. Instance variables are stored on the heap, whereas local variables live on the stack. If a lambda could access the local variable directly and the lambda were used in a thread, then the thread using the lambda could try to access the variable after the thread that allocated the variable had deallocated it. Hence, Java implements access to a free local variable as access to a copy of it rather than access to the original variable.  
> This makes no difference if the local variable is assigned to only once hence the restriction. Second, this restriction also discourages typical imperative programming patterns that mutate an outer variable.

lambda表达式可能在新线程中运行：

1. 主线程和新线程同时对变量修改，会导致并发安全问题；(表达式内不可修改)
2. 局部变量存储在栈上，新线程无法访问到，如果是final的变量，可以直接把值copy一份存到堆上使用；（表达式外不可修改）

实际上，lambda表达式中的外部变量是堆上的一个固定值。

### lambda表达式的类型推断与方法重载

多数情况下，javac可以进行类型推断，推断出lambda表达式的类型

1. 只有一个目标（方法名相同，参数数量相同），有方法中的参数类型推导出
2. 多个可能目标，根据**最具体的类型**推导出(子比父具体，类比接口具体)
3. 多个可能目标，不确定最具体类型，需要人为指定

## Stream接口

> To perform a computation, stream operations are composed into a stream pipeline. A stream pipeline consists of a source (which might be an array, a collection, a generator function, an I/O channel, etc), zero or more intermediate operations (which transform a stream into another stream, such as filter(Predicate)), and a terminal operation (which produces a result or side-effect, such as count() or forEach(Consumer)). Streams are lazy; computation on the source data is only performed when the terminal operation is initiated, and source elements are consumed only as needed.

intermediate operations:中间型操作，只记录传入的行为，不会执行，返回Stream

![2024-01-26-lambda-stream-20240126112821](https://cdn.jsdelivr.net/gh/Legolas-PRC/picBed@main/blogPic/2024-01-26-lambda-stream-20240126112821.png)

termimal operations:终结型操作，执行流中的全部操作

![2024-01-26-lambda-stream-20240126113132](https://cdn.jsdelivr.net/gh/Legolas-PRC/picBed@main/blogPic/2024-01-26-lambda-stream-20240126113132.png)

### reduce

> Reduction operations  
A reduction operation (also called a fold) takes a sequence of input elements and combines them into a single summary result by repeated application of a combining operation, such as finding the sum or maximum of a set of numbers, or accumulating elements into a list. The streams classes have multiple forms of general reduction operations, called reduce() and collect(), as well as multiple specialized reduction forms such as sum(), max(), or count().

规约操作：  
规约操作获取一系列输入元素，并通过重复应用组合操作(例如找到一组数字的和或最大值，或将元素累加到列表中)将它们组合成单个汇总结果。流类具有多种形式的通用规约操作，称为reduce()和collect()，以及多种专用的规约形式，如sum()、max()或count()。

#### reduce()

> Performs a reduction on the elements of this stream, using the provided identity value and an associative accumulation function, and returns the reduced value. This is equivalent to:  

    T result = identity; 
    for (T element : this stream)      
        result = accumulator.apply(result, element)  
    return result;

> but is not constrained to execute sequentially.  
The identity value must be an identity for the accumulator function. This means that for all t, accumulator.apply(identity, t) is equal to t. The accumulator function must be an associative function.

associative:满足结合律的，可以并行执行

```java
T reduce(T identity, BinaryOperator<T> accumulator){
 	T result = identity;  
  for (T element : this stream)      
    result = accumulator.apply(result, element)  
  return result;  
}

// 从首个元素开始操作，没有元素则返回空
Optional<T> reduce(BinaryOperator<T> accumulator){
  boolean foundAny = false;  
  T result = null;  
  for (T element : this stream) {      
    if (!foundAny) {          
      foundAny = true;          
      result = element;      
    }      
    else          
      result = accumulator.apply(result, element);  
  }  return foundAny ? Optional.of(result) : Optional.empty();
}

// 并行操作时，combiner()将多个结果组合
<U> U reduce(U identity,
                 BiFunction<U, ? super T, U> accumulator,
                 BinaryOperator<U> combiner){
  U result = identity;  
  for (T element : this stream)      
    result = accumulator.apply(result, element)  
  return result;
}
```

### collect()

执行特殊的reduce操作，被reduce的值是一个存放可变结果的容器
> Performs a mutable reduction operation on the elements of this stream. A mutable reduction is one in which the reduced value is a mutable result container, such as an ArrayList, and elements are incorporated by updating the state of the result rather than by replacing the result. This produces a result equivalent to:

    R result = supplier.get();  
    for (T element : this stream)  
        accumulator.accept(result, element);  
    return result;

> Like reduce(Object, BinaryOperator), collect operations can be parallelized without requiring additional synchronization.

```java
<R> R collect(Supplier<R> supplier,BiConsumer<R, ? super T> accumulator,BiConsumer<R, R> combiner){
	R result = supplier.get();  
    for (T element : this stream)      
        accumulator.accept(result, element);  
    return result;
}

<R, A> R collect(Collector<? super T, A, R> collector);
```

### Collector

```java
public interface Collector<T, A, R> {
    Supplier<A> supplier();
    BiConsumer<A, T> accumulator();
    BinaryOperator<A> combiner();
    Function<A, R> finisher();
    Set<Characteristics> characteristics();
}
```

> A Collector is specified by four functions that work together to accumulate entries into a mutable result container, and optionally perform a final transform on the result. They are:  
creation of a new result container (supplier())  
incorporating a new data element into a result container (accumulator())  
combining two result containers into one (combiner())  
performing an optional final transform on the container (finisher())

---

参考：  
[万字长文详解Java lambda表达式](https://mp.weixin.qq.com/s/ud5TckFLXWrVpilmobynhQ)  
《java函数式编程》  

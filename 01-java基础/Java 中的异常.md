# 基本异常

异常情形是指阻止当前方法或作用域继续执行的问题。

解决方式：

> 可以通过创建一个代表错误信息的对象，并且将它从当前的环境中“抛出”，这样就把错误信息传播到了一个更大的环境中，这种被称之为 “抛出异常”。

```java
if(user == null){
    throw new NullPointException();
}
```
这样便抛出了异常，我们在当前环境中就不必再为这个问题操心了，它将会在别的地方被处理。

所有的标准异常类都有两个构造器，一个是默认构造器，另一个是接受字符串作为参数，以便能把相关信息放入异常对象的构造器。

如

```java
throw new NullPointException("这是一个空指针异常");
```


## 捕获异常

#### 1. try 块

如果在方法的内部抛出了异常（或者在方法内部调用的其他方法抛出了异常），这个方法将在抛出异常的过程中结束，如果不希望这个方法结束，可以在方法内部设置一个特殊的块来捕获异常。这种块称为 `try` 块。

```java
try{

}
```

#### 2. 异常处理程序
抛出的异常必须在某一处进行处理，这个 “地点” 就是 “异常处理程序”。

```java
try{

}catch(Type1 type1){

}catch(Type2 type2){

}

```

此时的 `catch` 中就是异常处理程序，注意：只有匹配的 `catch` 子句才能得到执行，这与 `switch` 不同，`switch` 语句遇到在每一个 `case` 后面跟上一个 `break`，以避免执行后续的 `case` 子句。

## 创建自定义异常

```java
//创建一个自定义异常
public class SimpleException extends Exception{

}

```

## finally 的执行时机
当设计 `break` 和 `continue` 语句的时候， `finally` 子句也会得到执行。不管 `try catch` 中添加了何种语句，最终 `finally` 都是会被执行的。
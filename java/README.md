# lambda 表达式

> 1. lambada 表达式实质上是一个匿名方法，但该方法并非独立执行，而是用于**实现由函数式接口定义的唯一抽象方法**
> 2. 使用 lambda 表达式时，会创建实现了函数式接口的一个匿名类实例
> 3. 可以将 lambda 表达式视为一个对象，可以将其作为参数传递



## 函数式接口

函数式接口是`仅含`一个抽象方法的接口，但可以指定 Object 定义的任何公有方法。

- 以下是一个函数式接口：

```java
@FunctionalInterface
public interface IFuntionSum<T extends Number> {
    T sum(List<T> numbers);      // 抽象方法
}
```

- 以下也是一个函数式接口：

```java
@FunctionalInterface
public interface IFunctionMulti<T extends Number> {
    void multi(List<T> numbers); // 抽象方法
    
    boolean equals(Object obj);  // Object中的方法
}
```

- 但如果改为以下形式，则不是函数式接口：

```java
@FunctionalInterface
public interface IFunctionMulti<T extends Number> extends IFuntionSum<T> {
    void multi(List<T> numbers);
    
    @Override
    boolean equals(Object obj);
}
// IFunctionMulti 接口继承了 IFuntionSum 接口，此时 IFunctionMulti 包含了2个抽象方法
```

!> **tip 1**: 可以用 **@FunctionalInterface** 标识函数式接口，非强制要求，但有助于编译器及时检查接口是否满足函数式接口定义

!> **tip 2**: 在 Java 8 之前，接口的所有方法都是抽象方法，在 Java 8 中新增了接口的默认方法



## lambda 表达式

lambda 表达式的2种形式
 包含单独表达式 ：parameters -> an expression



```java
list.forEach(item -> System.out.println(item));
```

包含代码块：parameters -> { expressions };



```java
list.forEach(item -> {
  int numA = item.getNumA();
  int numB = item.getNumB();
      System.out.println(numA + numB);
});
```

左侧指定 lambda 表达式需要的参数，右侧指定 lambda 方法体



上文提到，lambda 无法独立执行，它必须是实现一个函数式接口的唯一抽象方法。

**每个 lambda 表达式背后必定有一个函数式接口，该表达式实现的是这个函数式接口内部的唯一抽象方法。**



## lambda 表达式规约

lambda 表达式的参数可以通过上下文推断，如果需要显示声明一个参数的类型，则必须为所有的参数声明类型。

```java
@FunctionalInterface
public interface IFunctionMod {
  boolean (int n, int d);
}
  
IFunctionMod function = (n, d） -> (n % d) == 0          // 合理，n 和 d 的类型通过上下文推断
IFunctionMod function = (int n, int d) -> (n % d) == 0   // 合理，指定 n 和 d 的类型
IFunctionMod function = (int n, d) -> (n % d) == 0       // 不合理，须显示声明所有参数类型
```
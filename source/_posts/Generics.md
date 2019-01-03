---
title: Generics
date: 2018-08-02 23:04:52
tags:
    - JavaBasic
---

{% meting "419596411" "netease" "song" %}

## 使用泛型的好处
* 能在编译时进行强类型检查<br>如果代码类型安全检查不通过，编译器将抛出编译错误警告，一般来说，修复在编译期发现的错误比在修复在运行时抛出的错误更简单、
* 消除手动的类型转换<br>不使用泛型的代码要手动进行类型转换，容易出错，而使用泛型的代码，编译器将自动进行类型转换
* 使开发者能够实现通用的算法<br>比如Arrays的sort方法

## Terminology
* __Type Parameter and Type Argument__ 
>Many developers use the terms "type parameter" and "type argument" interchangeably, but these terms are not the same. When coding, one provides type arguments in order to create a parameterized type. Therefore, the T in Foo<T> is a type parameter and the String in Foo<String> f is a type argument.

## 泛型的格式
>A generic class is defined with the following format:

`class name<T1, T2, ..., Tn> { /* ... */ }`
>The type parameter section, delimited by angle brackets (<>), follows the class name. It specifies the type parameters (also called type variables) T1, T2, ..., and Tn.

例子：
```java
class GenClass<T>{
    T data;
    public void setData(T data);
    public T getData{(return this.data)};
}
```

## Type Parameter Naming Conventions
类型参数传统命名规则：
* E - Element (used extensively by the Java Collections Framework)
* K - Key
* N - Number
* T - Type
* V - Value
* S,U,V etc. - 2nd, 3rd, 4th types

## 实例化泛型类
`GenClass<String> genClass;`
使用具体的类型，比如String,Integer替换掉T，就得到了泛型类GenClass的引用。

`GenClass<String> genClass = new GenClass<String>()`
这样就实例化了泛型类GenClass，语法和普通类没有多大差别，只是多了<>。

在JDK1.7以及之后的版本，只需要`new GenClass<>()`就可以实例化泛型类GenClass。

如果省略<>，实例化的是GenClass的原生态类型，不是泛型类型，
这使得genClass的引用是泛型，但却指向了原生态类型的对象，这是不安全的，编译器会发出警告。
如果一个类不是泛型类，比如`class A{}`，它是不存在原生态类型这个概念的

## 原生态类型
```java
class A<T>{public void set(T t){/**/}}
A a = new A();//没有<>，这样就是原生态类型
A<Integer> a1 = new A<Integer>();//有<>,这样就是泛型类型
A<Integer> a2 = new A();// 合法，为了向后兼容，泛型类型的引用可以指向原生态类型，但编译器会发出警告
A a3 = a1;
a3.set(1);//合法，但编译器会发出警告。
//当你使用原生态类型的引用去调用它对应的泛型类的方法的时候，编译器就会发出警告
//使用原生态类型会跳过编译器的安全检查，这是不安全的，因此应该避免使用原生态类型
//使用原生态类型会失去泛型的优势
```

## Unchecked Error Messages
当你在代码中混合使用原生态类型和泛型时，
比如这样子：
```java
public class WarningDemo {
    public static void main(String[] args){
        Box<Integer> bi;
        bi = createBox();
    }
    static Box createBox(){
        return new Box();
    }
}
```
编译器就会发出这样的警告信息：
```java
Note: Example.java uses unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.
```
可以使用@SuppressWarnings注解去掉警告信息，当你必须确保你的代码是安全的

## 范型的限制
* 不能使用原始类型实例化泛型类<br>`new ArrayList<int>//错误`
* 不能实例化类型参数<br>`void <T> fun(T t){new T()//错误}`
* 静态域不能声明为参数化类型<br>`class A<T>{static T t;//错误}`
* 不能对参数化类型使用instanceof操作符或对没有使用通配符的类型参数进行类型转换<br>
```java
func(List<E> lists){
  for(E l:lists){
    if(l instanceof List<String>//错误)
    if(l instanceof List<?>//OK)
    if(l instanceof List//OK)
  }
  
}
func1(){
  List<Integer> list = new ArrayList<>();
  List<Number> numbers = (List<Number>) list;  //错误,List<Integer>和List<Numbe>不是同一个类型
  List<?> list1 = new ArrayList<>();
  List<Number> num = (List<Number>) list1  //OK,使用了通配符
  List<String> l1 = ...;
  ArrayList<String> l2 = (ArrayList<String>)l1;  // OK
}
```
* 不能实例化参数化类型的数组<br>`new ArrayList<String>[2] //错误`
* 不能创建、捕捉、抛出参数化类型的异常
```java
// Extends Throwable indirectly
class MathException<T> extends Exception { /* ... */ }    // compile-time error

// Extends Throwable directly
class QueueFullException<T> extends Throwable { /* ... */ // compile-time error

public static <T extends Exception, J> void execute(List<J> jobs) {
    try {
        for (J job : jobs)
            // ...
    } catch (T e) {   // compile-time error
        // ...
    }
}
You can, however, use a type parameter in a throws clause:

class Parser<T extends Exception> {
    public void parse(File file) throws T {     // OK
        // ...
    }
}
```
* 同一个类中不能有两个类型擦除后方法签名相同的方法<br>
```java
class A{
  public void func(List<String> l){}
  public void func(List<Integer> l){}
  //类型擦除后方法签名相同，编译错误
}
```

泛型的内容有点多，google翻了一下找到了一篇总结得比较简洁的博客，这博客我先[记下了](https://my.oschina.net/polly/blog/877647)

笔记来源： [Generics](https://docs.oracle.com/javase/tutorial/java/generics/why.html)

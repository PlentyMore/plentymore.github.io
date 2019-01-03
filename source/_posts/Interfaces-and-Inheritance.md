---
title: Interfaces and Inheritance
date: 2018-08-01 14:30:46
tags:
    - JavaBasic
---
## Interfaces in Java
>In the Java programming language, an interface is a _reference type_, similar to a class, that can contain only __constants__, __method signatures__, __default methods__, __static methods__, and __nested types__. Method bodies exist only for default methods and static methods. Interfaces cannot be instantiated—they can only be implemented by classes or extended by other interfaces. Extension is discussed later in this lesson.

接口中可以有instance fields，但只能为常量(final
```java
// 假设两个接口具有相同的成员变量（均为effectively final)
public interface Bike {
   // 不能改变a的值，改变后a就不是effectively final了 
   int a = 10;
}
public interface Car {
    int a = 10; //不能改变
}
public class TestInterface implements Car,Bike{
    public void f(){
    // 直接访问a提示Reference to 'a' is ambiguous
   // 只能通过Car.a或Bike.a访问
    System.out.print(Car.a);
    }
}

//编译后的TestInterface.class文件
public class TestInterface implements Car, Bike {
    public TestInterface() {
    }
    public void f() {
        // a已经被替换成字面值
        System.out.print(10);
    }
}

public interface OperateCar {

   // constant declarations, if any
  int CONSTANT_1 = 1; //never change the value
  final  int CONSTANT_2 = 2; //explicit final
  static int CONSTANT_3 = 3; 

   // method signatures
   
   // An enum with values RIGHT, LEFT
   int turn(Direction direction,
            double radius,
            double startSpeed,
            double endSpeed);
   int changeLanes(Direction direction,
                   double startSpeed,
                   double endSpeed);
   int signalTurn(Direction direction,
                  boolean signalOn);
   int getRadarFront(double distanceToCar,
                     double speedOfCar);
   int getRadarRear(double distanceToCar,
                    double speedOfCar);
         ......
   // more method signatures
}
```

## Inheritance
### What You Can Do in a Subclass
>A subclass inherits all of the public and protected members of its parent, no matter what package the subclass is in. If the subclass is in the same package as its parent, it also inherits the package-private members of the parent. You can use the inherited members as is, replace them, hide them, or supplement them with new members:
* The inherited fields can be used directly, just like any other fields.
* You can declare a field in the subclass with the same name as the one in the superclass, thus hiding it (not recommended).
* You can declare new fields in the subclass that are not in the superclass.
* The inherited methods can be used directly as they are.
* You can write a new instance method in the subclass that has the same signature as the one in the superclass, thus overriding it.
* You can write a new static method in the subclass that has the same signature as the one in the superclass, thus hiding it.
* You can declare new methods in the subclass that are not in the superclass.
* You can write a subclass constructor that invokes the constructor of the superclass, either implicitly or by using the keyword super.
###  Private Members in a Superclass
>A subclass does not inherit the private members of its parent class. (子类不继承父类的private修饰的成员和方法）However, if the superclass has public or protected methods for accessing its private fields, these can also be used by the subclass.

>A nested class has access to all the private members of its enclosing class—both fields and methods. （嵌套非静态内部类可以访问其外部类的所有私有成员和方法）Therefore, a public or protected nested class inherited by a subclass has indirect access to all of the private members of the superclass.
## Multiple Inheritance of State, Implementation, and Type
### 类不能多继承的原因
假设可以继承多个类，如果这多个类含有同名的成员变量，则会造成混淆，而接口中不包含成员变量（若包含也只是(effecitively) final的成员变量，相当于常量)，因此接口可以多继承
>One reason why the Java programming language does not permit you to extend more than one class is to avoid the issues of multiple inheritance of state, which is the ability to inherit fields from multiple classes. For example, suppose that you are able to define a new class that extends multiple classes. When you create an object by instantiating that class, that object will inherit fields from all of the class's superclasses. What if methods or constructors from different superclasses instantiate the same field? Which method or constructor will take precedence? Because interfaces do not contain fields, you do not have to worry about problems that result from multiple inheritance of state.

>Multiple inheritance of implementation is the ability to inherit method definitions from multiple classes. Problems arise with this type of multiple inheritance, such as name conflicts and ambiguity. When compilers of programming languages that support this type of multiple inheritance encounter superclasses that contain methods with the same name, they sometimes cannot determine which member or method to access or invoke. In addition, a programmer can unwittingly introduce a name conflict by adding a new method to a superclass. Default methods introduce one form of multiple inheritance of implementation. A class can implement more than one interface, which can contain default methods that have the same name. The Java compiler provides some rules to determine which default method a particular class uses.
### Interface Methods
>Default methods and abstract methods in interfaces are inherited like instance methods. However, when the supertypes of a class or interface provide multiple default methods with the same signature, the Java compiler follows inheritance rules to resolve the name conflict. These rules are driven by the following two principles:
* __Instance methods are preferred over interface default methods.__（实例方法优先级高于接口默认方法）
```java
// Consider the following classes and interfaces:

public class Horse {
    public String identifyMyself() {
        return "I am a horse.";
    }
}
public interface Flyer {
    default public String identifyMyself() {
        return "I am able to fly.";
    }
}
public interface Mythical {
    default public String identifyMyself() {
        return "I am a mythical creature.";
    }
}
public class Pegasus extends Horse implements Flyer, Mythical {
    public static void main(String... args) {
        Pegasus myApp = new Pegasus();
        System.out.println(myApp.identifyMyself());
        //output: I am a horse.
    }
}
```
* __Methods that are already overridden by other candidates are ignored. This circumstance can arise when supertypes share a common ancestor.__（被重写的父接口的默认方法将被忽略，将调用重写后的子接口方法）
```java
// Consider the following interfaces and classes:

public interface Animal {
    default public String identifyMyself() {
        return "I am an animal.";
    }
}
public interface EggLayer extends Animal {
    default public String identifyMyself() {
        return "I am able to lay eggs.";
    }
}
public interface FireBreather extends Animal { }
public class Dragon implements EggLayer, FireBreather {
    public static void main (String... args) {
        Dragon myApp = new Dragon();
        System.out.println(myApp.identifyMyself());
        //output: I am able to lay eggs.
    }
}
```
如果两个相互独立的默认方法冲突了，比如
```java
interface A{default void a(){}}
interface B {default void a(){} }
class C implements A,B{}
```
 或者一个默认方法与另一个抽象方法冲突了
```java
interface A{abstract void a();}
interface B{default void a(){};}
class C implements A,B{}
```
编译器会报错，必须重写父类型的方法
>If two or more independently defined default methods conflict, or a default method conflicts with an abstract method, then the Java compiler produces a compiler error. You must explicitly override the supertype methods.

>Consider the example about computer-controlled cars that can now fly. You have two interfaces (OperateCar and FlyCar) that provide default implementations for the same method, (startEngine):
```java
public interface OperateCar {
    // ...
    default public int startEngine(EncryptedKey key) {
        // Implementation
    }
}
public interface FlyCar {
    // ...
    default public int startEngine(EncryptedKey key) {
        // Implementation
    }
}
```
>A class that implements both OperateCar and FlyCar must override the method startEngine. You could invoke any of the of the default implementations with the super keyword.
```java
public class FlyingCar implements OperateCar, FlyCar {
    // ...
    public int startEngine(EncryptedKey key) {
        FlyCar.super.startEngine(key);
        OperateCar.super.startEngine(key);
    }
}
```
>The name preceding super (in this example, FlyCar or OperateCar) must refer to a direct superinterface that defines or inherits a default for the invoked method. This form of method invocation is not restricted to differentiating between multiple implemented interfaces that contain default methods with the same signature. You can use the super keyword to invoke a default method in both classes and interfaces.

从父类中继承的实例方法会重写接口中的方法（实例方法与接口抽象方法签名相同）
>Inherited instance methods from classes can override abstract interface methods. Consider the following interfaces and classes:
```java
public interface Mammal {
    String identifyMyself();
}
public class Horse {
/*当Horse为接口时，Mustang同时实现Horse和
Mammal借口，重写不会发生*/
    public String identifyMyself() {
        return "I am a horse.";
    }
}
public class Mustang extends Horse implements Mammal {
    public static void main(String... args) {
        Mustang myApp = new Mustang();
       /* The class Mustang inherits the method identifyMyself 
       from the class Horse, which overrides the abstract method of
        the same name in the interface Mammal. */
        System.out.println(myApp.identifyMyself());
        //output: I am a horse.
    }
}
```
## summary
>The following table summarizes what happens when you define a method with the same signature as a method in a superclass.

**Defining a Method with the Same Signature as a Superclass's Method**

 _  | Superclass Instance Method | Superclass Static Method
--- | -------------------------------------- | ---------------------------------
Subclass Instance Method | Overrides | Generates a compile-time error
Subclass Static Method | Generates a compile-time error | Hides
## Abstract Classes Compared to Interfaces
Abstract classes are similar to interfaces. You cannot instantiate them, and they may contain a mix of methods declared with or without an implementation. However, with abstract classes, you can declare fields that are not static and final, and define public, protected, and private concrete methods. **With interfaces, all fields are automatically public, static, and final**, and all methods that you declare or define (as default methods) are public. In addition, you can extend only one class, whether or not it is abstract, whereas you can implement any number of interfaces.

Which should you use, abstract classes or interfaces?
* Consider using **abstract classes** if any of these statements apply to your situation:
  * You want to share code among several closely related classes.
  * You expect that classes that extend your abstract class have many common methods or fields, or require access modifiers other than public (such as protected and private).
  * You want to declare non-static or non-final fields. This enables you to define methods that can access and modify the state of the object to which they belong.
* Consider using **interfaces** if any of these statements apply to your situation:
  * You expect that unrelated classes would implement your interface. For example, the interfaces Comparable and Cloneable are implemented by many unrelated classes.
  * You want to specify the behavior of a particular data type, but not concerned about who implements its behavior.
  * You want to take advantage of multiple inheritance of type.

笔记来源：[Interfaces and Inheritance](https://docs.oracle.com/javase/tutorial/java/IandI/index.html)
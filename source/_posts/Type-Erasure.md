---
title: Type Erasure
date: 2018-08-02 15:08:49
tags:
    - JavaBasic
---

{% meting "692193" "netease" "song" %}

## 类型擦除的目的
>Generics were introduced to the Java language to provide tighter type checks at compile time and to support generic programming. To implement generics, the Java compiler applies type erasure to:

>* Replace all type parameters in generic types with their bounds or Object if the type parameters are unbounded. The produced bytecode, therefore, contains only ordinary classes, interfaces, and methods.
>* Insert type casts if necessary to preserve type safety.
>* Generate bridge methods to preserve polymorphism in extended generic types.

>Type erasure ensures that no new classes are created for parameterized types; consequently, generics incur no runtime overhead.（类型擦除保证了不产生新的classes，使得泛型不会增加运行开销

## 通过class文件观察类型擦除
使用Intellij自带的反编译工具查看class文件时，由于这个工具过于智能，会将类型擦除后
的信息根据常量池和其他地方的信息还原回去，
导致最终看到的class文件内容和源代码几乎没有区别

源文件

![Cla.java](https://i.imgur.com/s6cdOt7.png)

class文件

![Cla.class](https://i.imgur.com/ExG6fqV.png)

因此，要查看类型擦除后的class文件，可以使用javap或者[ClassViewer](https://github.com/ClassViewer/ClassViewer.git)
* 使用javap查看class文件信息<br>在终端运行`javap -v -c 文件名.class`即可看到class文件详细信息，如下图：
![javap](https://i.imgur.com/iCuC3xx.png)

* 使用ClassViewer<br>为了方便可以直接下载release的[jar文件](https://github.com/ClassViewer/ClassViewer/releases)运行，界面如下图：
![CLassViewer](https://i.imgur.com/1ia1fBk.png)

你需要对class文件结构有一定的了解，[了解一下](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html)

接下来就可以通过class文件观察类型擦除了
源文件代码为：
```angularjs
public class Cla<T> {
    private T t;
    public Cla(){}
    public void setT(T t){
        this.t = t;
    }
    public T getT(){
        return this.t;
    }

    public static void main(String[] args) {
        Cla raw = new Cla();
        raw.setT("a");
        Object o = raw.getT();
        Cla<String> gen = new Cla<>();
        gen.setT("abc");
        String s = gen.getT();
    }
}
```
运行`javap -v -c Cla.class`后得到：
```angularjs
Classfile /home/plentymore/Downloads/basicjava/target/classes/com/pltm/basicjava/gc/Cla.class
  Last modified Aug 3, 2018; size 1149 bytes
  MD5 checksum d095b63ee57fb71c1fa4f47904c8693e
  Compiled from "Cla.java"
public class com.pltm.basicjava.gc.Cla<T extends java.lang.Object> extends java.lang.Object
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #10.#43        // java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#44         // com/pltm/basicjava/gc/Cla.t:Ljava/lang/Object;
   #3 = Class              #45            // com/pltm/basicjava/gc/Cla
   #4 = Methodref          #3.#43         // com/pltm/basicjava/gc/Cla."<init>":()V
   #5 = String             #46            // a
   #6 = Methodref          #3.#47         // com/pltm/basicjava/gc/Cla.setT:(Ljava/lang/Object;)V
   #7 = Methodref          #3.#48         // com/pltm/basicjava/gc/Cla.getT:()Ljava/lang/Object;
   #8 = String             #49            // abc
   #9 = Class              #50            // java/lang/String
  #10 = Class              #51            // java/lang/Object
  #11 = Utf8               t
  #12 = Utf8               Ljava/lang/Object;
  #13 = Utf8               Signature
  #14 = Utf8               TT;
  #15 = Utf8               <init>
  #16 = Utf8               ()V
  #17 = Utf8               Code
  #18 = Utf8               LineNumberTable
  #19 = Utf8               LocalVariableTable
  #20 = Utf8               this
  #21 = Utf8               Lcom/pltm/basicjava/gc/Cla;
  #22 = Utf8               LocalVariableTypeTable
  #23 = Utf8               Lcom/pltm/basicjava/gc/Cla<TT;>;
  #24 = Utf8               setT
  #25 = Utf8               (Ljava/lang/Object;)V
  #26 = Utf8               (TT;)V
  #27 = Utf8               getT
  #28 = Utf8               ()Ljava/lang/Object;
  #29 = Utf8               ()TT;
  #30 = Utf8               main
  #31 = Utf8               ([Ljava/lang/String;)V
  #32 = Utf8               args
  #33 = Utf8               [Ljava/lang/String;
  #34 = Utf8               raw
  #35 = Utf8               o
  #36 = Utf8               gen
  #37 = Utf8               s
  #38 = Utf8               Ljava/lang/String;
  #39 = Utf8               Lcom/pltm/basicjava/gc/Cla<Ljava/lang/String;>;
  #40 = Utf8               <T:Ljava/lang/Object;>Ljava/lang/Object;
  #41 = Utf8               SourceFile
  #42 = Utf8               Cla.java
  #43 = NameAndType        #15:#16        // "<init>":()V
  #44 = NameAndType        #11:#12        // t:Ljava/lang/Object;
  #45 = Utf8               com/pltm/basicjava/gc/Cla
  #46 = Utf8               a
  #47 = NameAndType        #24:#25        // setT:(Ljava/lang/Object;)V
  #48 = NameAndType        #27:#28        // getT:()Ljava/lang/Object;
  #49 = Utf8               abc
  #50 = Utf8               java/lang/String
  #51 = Utf8               java/lang/Object
{
  public com.pltm.basicjava.gc.Cla();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 24: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/pltm/basicjava/gc/Cla;
      LocalVariableTypeTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/pltm/basicjava/gc/Cla<TT;>;

  public void setT(T);
    descriptor: (Ljava/lang/Object;)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: putfield      #2                  // Field t:Ljava/lang/Object;
         5: return
      LineNumberTable:
        line 26: 0
        line 27: 5
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       6     0  this   Lcom/pltm/basicjava/gc/Cla;
            0       6     1     t   Ljava/lang/Object;
      LocalVariableTypeTable:
        Start  Length  Slot  Name   Signature
            0       6     0  this   Lcom/pltm/basicjava/gc/Cla<TT;>;
            0       6     1     t   TT;
    Signature: #26                          // (TT;)V

  public T getT();
    descriptor: ()Ljava/lang/Object;
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: getfield      #2                  // Field t:Ljava/lang/Object;
         4: areturn
      LineNumberTable:
        line 29: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/pltm/basicjava/gc/Cla;
      LocalVariableTypeTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/pltm/basicjava/gc/Cla<TT;>;
    Signature: #29                          // ()TT;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=5, args_size=1
         0: new           #3                  // class com/pltm/basicjava/gc/Cla
         3: dup
         4: invokespecial #4                  // Method "<init>":()V
         7: astore_1
         8: aload_1
         9: ldc           #5                  // String a
        11: invokevirtual #6                  // Method setT:(Ljava/lang/Object;)V
        14: aload_1
        15: invokevirtual #7                  // Method getT:()Ljava/lang/Object;
        18: astore_2
        19: new           #3                  // class com/pltm/basicjava/gc/Cla
        22: dup
        23: invokespecial #4                  // Method "<init>":()V
        26: astore_3
        27: aload_3
        28: ldc           #8                  // String abc
        30: invokevirtual #6                  // Method setT:(Ljava/lang/Object;)V
        33: aload_3
        34: invokevirtual #7                  // Method getT:()Ljava/lang/Object;
        37: checkcast     #9                  // class java/lang/String
        40: astore        4
        42: return
      LineNumberTable:
        line 33: 0
        line 34: 8
        line 35: 14
        line 36: 19
        line 37: 27
        line 38: 33
        line 39: 42
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      43     0  args   [Ljava/lang/String;
            8      35     1   raw   Lcom/pltm/basicjava/gc/Cla;
           19      24     2     o   Ljava/lang/Object;
           27      16     3   gen   Lcom/pltm/basicjava/gc/Cla;
           42       1     4     s   Ljava/lang/String;
      LocalVariableTypeTable:
        Start  Length  Slot  Name   Signature
           27      16     3   gen   Lcom/pltm/basicjava/gc/Cla<Ljava/lang/String;>;
}
Signature: #40                          // <T:Ljava/lang/Object;>Ljava/lang/Object;
SourceFile: "Cla.java"
```

我们重点观察这里
```angularjs
         0: new           #3                  // class com/pltm/basicjava/gc/Cla
         3: dup
         4: invokespecial #4                  // Method "<init>":()V
         7: astore_1
         8: aload_1
         9: ldc           #5                  // String a
        11: invokevirtual #6                  // Method setT:(Ljava/lang/Object;)V
        14: aload_1
        15: invokevirtual #7                  // Method getT:()Ljava/lang/Object;
        //由于使用了原生态类型，编译器不会插入类型转换
        18: astore_2
        19: new           #3                  // class com/pltm/basicjava/gc/Cla
        22: dup
        23: invokespecial #4                  // Method "<init>":()V
        26: astore_3
        27: aload_3
        28: ldc           #8                  // String abc
        30: invokevirtual #6                  // Method setT:(Ljava/lang/Object;)V
        33: aload_3
        34: invokevirtual #7                  // Method getT:()Ljava/lang/Object;
        37: checkcast     #9                  // class java/lang/String
        //可以发现编译器自动插入了一个类型转换，转换为String类型
        40: astore        4
```

再看看其他地方，比如这里
```angularjs
public void setT(T);
    descriptor: (Ljava/lang/Object;)V
    //可以发现T被替换成了Object类型，其他地方也一样
```

## Erasure of Generic Types
* ### 类型参数无边界时
```angular2html
class GenClass<T>{
    private T data;
    public void setData(T d){this.data = data;} 
    public T getData(){return this.data;}
}
类型擦除后，将变成：
class GenClass{
    private Object data;
    public void setData(Object d){this.data = data;} 
    public Object getData(){return this.data;}
}
```
* ### 类型参数有边界时
```angular2html
class GenBounded<T extends Number>{
    private T data;
    public void setData(T data){this.data = data;}
    public T getData(){return this.data;}
}
类型擦除后，将变成：
class GenBounded{
    private Number data;
    public void setData(Number data){this.data = data;}
    public Number getData(){ return this.data;}
}
```

### 使用泛型类时

```angular2html
public static void main(String[] args){
    GenBounded<Integer> gb = new GenBounded<>();
    gb.setData(1);
    Integer num = gb.getData(); 
}
编译后将变成：
public static void main(String[] args){
    GenBounded gb = new GenBounded();
    gb.setData(1);
    Integer num = (Integer) gb.getData(); 
}
```

## Erasure of Generic Methods
* ### 类型参数无边界时
```angular2html
public static void <T> method(T[] arrays){
    for(T a:arrays){}
}
类型擦除后：
public static void method(Object[] arrays){
    for(Object a:arrays){}
}
```
* ### 类型参数有边界时
```angular2html
public static <T extend Number> T methodBounded(T number){
    return number;
}
类型擦除后：
public static Number methodBounded(Number number){
    return number;
}
```

## 类型擦除导致的问题
```angular2html
class GenClass<T>{
    private T data;
    public void setData(T data){this.data = data;}
    public T getData(){return this.data;}
}
class SubGenClass extend GenClass<String>{
    public void setData(String s){super.setData(s)}
    public String getData(){return this.data;}  
}
public static void main(){
    SubGenClass sub = new SubGenClass();
    GenClass genClass = sub;
    genClass.setData(1);
    String str = sub.getData();
}

类型擦除后：
class GenClass<Object>{
    private Object data;
    public void setData(Object data){this.data = data;}
    public Object getData(){return this.data;}
}
class SubGenClass extend GenClass<String>{
    public void setData(String s){super.setData(s)}
    public String getData(){return this.data;}  
}
public static void main(){
    SubGenClass sub = new SubGenClass();
    GenClass genClass = sub;//genClass的引用为原生态类型，编译器将发出警告
    genClass.setData(1);//将直接调用GenClass中的setData(Object data)方法
    String str = (String) sub.getData();//编译器自动插入的类型转换，将抛出ClassCastException
}
```
sub.getData()将调用super.getData()，而类型擦除后super.getData()返回的对象为Object类型，
引用指向的对象为Integer类型，因此super.getData()将返回一个引用为Object类型的指向
Integer类型的对象，最终将Integer类型转换成String类型时将抛出异常

## Bridge Methods
桥接方法是编译器自动添加的，不需要开发者手动添加，
桥接方法是为了解决泛型类继承之后多态性丢失的问题
这里仍然用上面的GenClass类进行讲解

假设没有桥接方法，类型擦除后，GenClass类变成了这样子：
```angularjs
class GenClass<Object>{
    private Object data;
    public void setData(Object data){this.data = data;}
    public Object getData(){return this.data;}
}
```

SubGenClass类变成了这样子：
```angularjs
class SubGenClass extend GenClass<String>{
    public void setData(String s){super.setData(s)}
    public String getData(){return this.data;}  
}
```

这样的话，SubGenClass的setData和getData方法就没有重写到其父类GenClass的
setData和getData方法，因为这两个类在类型擦除之后方法签名不一致了，这样就失去了多态性
什么是多态性，举个例子：
```angularjs
//Animal为Bird和Cat的父类
Animal a;
Bird b = new Bird();
Cat c = new Cat();
//假设Bird和Cat都重写了Animal的某个方法，比如eat()
a = b;
a.eat();//这里将调用Bird的eat方法
a = c;
a.eat();//这里将调用Cat的eat方法
```
再回到上面的SubGenClass，__没有桥接方法时__，当我们这样子尝试使用多态特性时候：
```angularjs
GenClass<Integer> gen;
SubGenClass subGen = new SubClass();
gen = subGen;
gen.setData(1);
```
因为SubGenClass的setData方法接收的参数为Integer，和GenClass的方法签名不一致，SubGenClass没能重写GenClass的setData方法，
所以这里将调用GenClass的setData(Object)方法。

GenClass的setData方法经过类型擦擦后变成了setData(Object),导致SubGenClass没能成功重写setData方法，这样是不符合预期的。

我们的本意是调用SubGenClass的setData(Integer)方法,
现在要调用这个方法只能通过强制类型转换将gen转换成SubGenClass类型才能调用。

一般地我们会下意识地认为，GenClass的setData方法也应该是setData(Integer)这种形式的，这样SubGenClass
就一定能够重写GenClass的setData方法，就能实现多态了。

但是类型擦擦后GenClass的setData方法变成了setData(Object)，
因此GenSubClass并不能成功重写GenClass的setData方法,所以上面将直接调用GenClass的setData方法

编译器添加的桥街方法如下：
```angularjs
public void setData(Object s){
  setData((Integer) s)
}//桥街方法也为Object类型，这样SubGenClass的setData方法签名就和GenClass的一样了
//setData(Object)里面调用了原来的setData(Integer)方法
//这样就成功地重写了setData方法，保留了多态性
```

桥接方法是编译器自动添加的，无需开发者手动添加，我们可以通过`javap -c 文件名.class`查看
同样的，使用Intellij idea里面自带的java bytecode decompiler是看不到编译器添加的桥接方法的，
需要使用上面的命令或者ClassVidwer查看。

## 堆污染
当一个泛型类的引用指向一个原生态类型的对象时，就导致堆污染
```angularjs
List<String> list = new ArrayList();//将发生堆污染
```
堆污染可能会导致类型转换错误，看下面的例子：
```angularjs
public void set(List<String> l){
  l.set("str");
}
public static void main(String[] args){
  List list = new ArrayList();
  set(list);
  Inerator<Integer> iter = list.iteraotr();
  while(iter.hasNext()){
    Integer i = iter.next();
    //运行时将抛出ClassCastException
    //因为String类型不能转换成Integer类型
  }
}
```

## NonReifiableVarargsType
官方文档讲得比较简单，详情请看链接
[NonReifiableVarargsType](http://www.angelikalanger.com/GenericsFAQ/FAQSections/ProgrammingIdioms.html#FAQ300)


笔记来源： [Type Erasure](https://docs.oracle.com/javase/tutorial/java/generics/erasure.html)
参考资料： [Heap Pollution](https://blog.csdn.net/zxm317122667/article/details/78400398)

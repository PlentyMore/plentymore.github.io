---
title: JVM Basic
date: 2018-09-14 15:14:57
tags:
    - JVM
---

# 加载(loading)
## 加载的时机
[The Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/index.html)中并没有规定class文件的加载时机，因此实际的加载时机取决于具体的虚拟机实现。

## 加载的步骤
1. 通过类的全限定名获取该类的二进制字节流
2. 将该类的二进制表示结构转换成在方法区的运行时结构
3. 在堆中生成一个该类的Class对象，作为上面的方法区运行结构的访问入口

上面的第一步通过类加载器来完成，可以使用系统的类加载器，也可以使用用户自定义的类加载器


# 连接(linking)
连接包括验证(Verification)、准备(Preparation)和解析(Resolution)三个步骤

## Verification
验证保证了类或者接口的二进制流机构是正确的，验证操作可能会导致额外的类的加载，
但不会导致额外的类的验证和准备。验证操作是可选的，可以通过`-Xverify:none`这个JVM参数关闭类验证，关闭类验证可以减少类加载时间，
关闭类验证是有风险的，可能会导致JVM加载有害的字节流导致系统崩溃

## Preparation
准备阶段将会为类或接口的静态域分配内存（在方法区上分配）并设置默认值，这个阶段可以发生和任何时期，但是必须要在初始化（initialization）之前完成。

## Resolution
解析阶段将会把常量池内的符号引用替换为直接引用

# 初始化
## 初始化的时机
JVM规范规定，一个类或接口只有在以下几种情况下才可能被初始化：
* 执行`new`, `getstatic`, `putstatic`, 或者`invokestatic`这4条JVM指令时：执行`new`指令（使用new关键字创建对象）的时候，如果对应的类还没有被初始化，则先对它进行初始化。
执行`getstatic`, `putstatic`, 或者`invokestatic`指令（读取或设置一个类或接口的静态变量（被final修饰的且已经在编译期把结果放入常量池的静态变量除外），调用一个类的静态方法）的时候，
如果对应的类或接口还被初始化，则对它进行初始化

* The first invocation of a java.lang.invoke.MethodHandle instance which
  was the result of method handle resolution (§5.4.3.5) for a method handle
  of kind 2 ( REF_getStatic ), 4 ( REF_putStatic ), 6 ( REF_invokeStatic ), or 8
  ( REF_newInvokeSpecial ).

* 调用java.lang.reflect包里面的方法对类进行反射调用的时候

* 子类初始化的时候，如果父类还没有初始化，则会先触发父类的初始化

* 如果一个接口声明了一个非静态非抽象方法，有类直接或间接地实现了这个接口，那个在这个类初始化的时候会出发这个接口的初始化

* 当一个类被用作JVM的启动类的时候，JVM会先初始化这个类

>>Prior to initialization, a class or interface must be linked, that is, verified, prepared,
  and optionally resolved.

在被初始化之前，一个类或者接口必须已经被连接（已验证，已准备，已解析）

## 初始化的步骤
在初始化阶段，虚拟机才开始执行类或接口的Java代码。初始化阶段做的就是收集类变量的赋值语句和静态代码块，
然后组成一个`<clint>`方法，接着执行这个方法，就完成了初始化过程。该方法是虚拟机创建并执行的，在字节码里面是看不到的。
收集代码组成`<clint>`方法的时候是按照代码顺序由上至下收集的，比如：
```angularjs
class A{
  static int a = 2;
  static {
    System.out.println(a);
    //System.out.println(b); 这样做的话将产生编译错误，静态代码块只能访问它前面定义的静态变量
    //后面定义的静态变量，只能赋值，不能访问
  }
  static int b = 3;
}
```
上面的类将组成一个这样的`<clint>`方法：
```angularjs
{
  a = 2;
  System.out.println(a);  //a 被上面的语句赋值为2，因此现在的值为2
  b = 3;
}
```


# 运行时数据区(Run-Time Data Areas)
JVM定义了多个运行时数据区，在运行java程序的时候会用到这些数据区。一些数据区在JVM启动的时候创建，并在JVM退出的时候销毁。
另一些数据区是是每个线程都拥有的，在线程创建的时候创建，在线程退出的时候销毁。

## 程序计数器(The pc Register)
JVM中的每个线程都有自己的程序计数器，因此这块区域是线程**私有**的，不是共享的。如果当前线程执行的方法不是本地方法，则程序计数器的值为当前执行的JVM指令的地址。
由于程序计数器存储了当前执行的JVM指令的地址，在切线切换后线程的执行将不会受到影响，能继续从程序计数器保存的地址继续执行指令。

## Java虚拟机栈(Java Virtual Machine Stacks)
Java虚拟机栈和程序计数器一样，是线程似有的，生命周期和线程一样，它用来存放Frame，Frame里面存放了局部变量、动态链接、方法返回值等信息。
每一个方法从被调用到返回的过程，都和Frame的入栈和出栈相对应。
### 栈设置
* -Xss<size>        set java thread stack size

### 可能抛出的异常
* If the computation in a thread requires a larger Java Virtual Machine stack than is permitted, the Java Virtual Machine throws a StackOverflowError.

* If Java Virtual Machine stacks can be dynamically expanded, and expansion is attempted but insufficient memory can be made available to effect the expansion, or if insufficient memory can be made available to create the initial Java Virtual Machine stack for a new thread, the Java Virtual Machine throws an OutOfMemoryError.

## 本地方法栈(Native Method Stacks)
本地方法栈和Java虚拟机栈的作用基本相同，区别是本地方法栈是为执行本地方法(native关键字修饰)服务的，而Java虚拟机栈是为执行Java方法服务的。
### 可能抛出的异常
* If the computation in a thread requires a larger native method stack than is permitted, the Java Virtual Machine throws a StackOverflowError.

* If native method stacks can be dynamically expanded and native method stack expansion is attempted but insufficient memory can be made available, or if insufficient memory can be made available to create the initial native method stack for a new thread, the Java Virtual Machine throws an OutOfMemoryError.

## 堆(Head)
Java堆是所有线程共享的内存区域，它是为所有类实例和数组分配内存空间的运行时数据区，在JVM启动的时候创建。堆上为对象分配的存储空间由自动存储管理系统（又被称作垃圾收集器）管理。堆上的对象不会被显式地回收。
堆是垃圾收集器管理的主要区域，各种垃圾回收算法还将堆进一步细分成不同的区域，比如分成新生代和老年代。
### 堆设置
* -Xms:初始堆大小
* -Xmx:最大堆大小
* -XX:NewSize=n:设置年轻代大小
* -XX:NewRatio=n:设置年轻代和年老代的比值。如:为3，表示年轻代与年老代比值为1：3，年轻代占整个年轻代年老代和的1/4
* -XX:SurvivorRatio=n:年轻代中Eden区与两个Survivor区的比值。注意Survivor区有两个。如：3，表示Eden：Survivor=3：2，一个Survivor区占整个年轻代的1/5
* -XX:MaxPermSize=n:设置持久代大小
### 可能抛出的异常
* If a computation requires more heap than can be made available by the automatic storage management system, the Java Virtual Machine throws an OutOfMemoryError.

## 方法区(Method Area)
方法区和堆一样，也是被所有线程共享的内存区域，方法区的作用和传统语言存储编译后的代码的存储区或者操作系统进程中的'text'段类似，
它存储每个类的结构，比如运行时常量池、域和方法等数据，还有方法和构造器里面的代码（包括代码块）。方法区在JVM启动的时候创建，
即使方法区逻辑上是堆的一部分，一般的JVM实现通常不会去回收它上面的内存。
>>方法区被很多开发者称作“永久代(Permanent Generation)”，本质上两者并不等价，仅仅是因为HotSpot VM的设计团队把GC分代收集拓展至方法区，或者说使用永久代来实现方法区而已
### 可能抛出的异常
* If memory in the method area cannot be made available to satisfy an allocation request, the Java Virtual Machine throws an OutOfMemoryError.

## 运行时常量池(Run-Time Constant Pool)
运行时常量池是每个类或接口的class文件的constant_pool表的运行时表示。运行时常量池从方法区里面分配空间，因此它是方法区的一部分，
它在JVM创建类或者接口的时候创建。
### 可能抛出的异常
* When creating a class or interface, if the construction of the run-time constant pool requires more memory than can be made available in the method area of the Java Virtual Machine, the Java Virtual Machine throws an OutOfMemoryError.

# 判断对象是否已经死亡
## 引用计数算法
给对象添加一个引用计数器，当有一个地方引用该对象的时，计数器的值就增加1，当引用失效时，计数器的值就减少1，因此当计数器的值为0的时候，既可以认为对象已经死亡（没有再被引用）。
该算法实现简单，判定效率高，但是无法解决循环引用的问题，比如下面的例子：
```angularjs
class A{
  Ojbect ref;
  public static void main(String[] args){
      A a = new A();
      B b = new B();
      a.ref = b;
      b.ref = a;
      a = null; //a.ref还引用着b原来引用的实例
      b = null; //b.ref也引用着a原来引用的实例
      // a和b一开始引用的对象将不会被判定为死亡 
  }
}

```

## 根搜索算法
通过一系列名为"GC Roots"的对象作为起点，从这些起点开始往下搜索，搜索所走过的路径成为引用链。当一个对象到达"GC Roots"没有任何引用链相连，则说明该对象已死亡（不可用），该对象将会判定为可回收的。
Java虚拟机将以下对象作为"GC Roots"对象：
* 虚拟机栈用引用的对象（局部变量引用的对象），比如`method(A a = new A())`
* 方法区中静态属性引用的对象，比如`class A{static A a = new A();}`
* 方法区中常量引用的对象，比如`class A{final A a = new A();}`
* 本地方法栈中引用的对象

## 引用类型
* 强引用（Strong Reference）

`Object obj = new Object()`这样的就是强引用，只要强引用还存在，垃圾回收器永远不会回收掉被引用的对象。

* 软引用（Sofe Reference）

软引用用来描述一些还有用，但不是必须的对象，在内存溢出之前，虚拟机会尝试对这些对象进行回收。如果回收之后内存还是不够才会出现内存溢出情况。
在JDK1.2之后，提供了SofeReference类实现软引用。

* 弱引用（Weak Reference）

和软引用一样，都是用来描述非必须对象的，但它要比软引用更弱一些，被弱引用的对象只能生存到下一次垃圾回收之前，当垃圾收集器开始工作，无论当前内存是否足够，
都会回收掉只被软引用关联的对象。在JDK1.2之后，提供了WeakReference类实现弱引用。

* 虚引用（Phantom Reference）

它是最弱的一种引用关系，一个对象是否有虚引用的存在，完全不会对其生存时间造成影响，也无法通过虚引用来取得一个对象实例，为一个对象设置虚引用关联的唯一目的，
就是希望在该对象被垃圾收集器回收时收到一个系统通知。在JDK1.2之后，提供了PhantomReference类实现虚引用。

## 自我救赎
在根搜索算法中，当一个对象到达"GC Roots"没有任何引用链相连，会被判定为死亡，但并不是真正的死亡，此时这些对象处于“缓刑”阶段，还有一次机会进行自救。
当一个对象到达"GC Roots"没有任何引用链相连，它将会被第一次标记并进行一次筛选，筛选就判断对象是否有必要执行finalize方法，假设该对象没有重写finalize方法，
或者它的finalize方法已经执行过一次了，则没有必要执行finalize方法，对象就真正死亡了。否则，该对象将会被放进一个F-Queue队列中，并放在稍后由虚拟机自动创建的低优先级finalize线程中执行。
虚拟机会触发这个方法，当不保证等待它执行完成，这样做的原因是避免等待finalize方法执行完成的时间过长使得F-Queue队列的其他对象一直处于等待状态，最终使得垃圾回收系统无法正常运作。
稍后垃圾收集器将会对F-Queue中的对象进行第二次标记，如果对象在第二次标记前运行了finalize方法成功拯救了自己（比如，将自己赋值给某个类变量或者对象的成员变量），那么第二次标记的时候它将会被移除出即将被回收的集合，
否则，对象将被判定为真正的死亡，然后被回收。例子如下：
```angularjs
class A{
  public static A SAVE_POINT;
  
  public static void main(String[] args) throws Throwable{
    SAVE_POINT = new A();
    SAVE_POINT = null; //对象将被第一次标记
    System.gc(); 
    Thread.sleep(500);//等待finalize方法执行
    if(SAVE_POINT != null){
      //如果在被第二次标记之前成功执行完了finalize方法，则成功拯救自己
      System.out.println("self save successfully");
    }
    else {
      System.out.println("died.");
    }
    SAVE_POINT = null;  //因为上次成功拯救了自己，因此这次也算是第一次标记
    System.gc();
    Thread.sleep(500);//等待finalize方法执行
    //因为对象的finalizer方法已经执行过一次了，所以这次对象必定会判定为真正死亡，然后被回收，自救的机会只有一次
    if(SAVE_POINT != null){
        System.out.println("self save successfully");
    }
    else {
      //死定了
        System.out.println("died.");
    }
  }
  
  @Override
  protected void finalize() throws Throwable {
      super.finalize();
      A.SAVE_POINT = this;
      System.out.println("save self");
  }
}
```

# 垃圾收集算法
## 标记-清除算法
该算法分为两个阶段，首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。
该算法的缺点：1.标记和回收效率都不高 2.会产生内存碎片，使得无法给较大的对象分配空间

## 复制算法
复制算法将内存平均分成两块，每次只使用其中的一块给对象分配空间，当这块空间用完之后，
就将这块空间上的存活的对象复制到另一块空间的内存上，然后清楚这块内存的空间，这样就不会产生内存碎片，
缺点是每次只能使用一半的空间，浪费严重。
>>现在的商用虚拟机都采用这种算法回收新生代

## 标记-整理算法
该算法和标记-清除算法基本相同，只不过该算法的后续步骤不是直接对被标记的对象进行清理，
而是将未被标记的对象向一端进行移动，然后直接清理掉端边界意外的内存。

## 分代收集算法
该算法根据对象的存活周期不同将内存劃分为几块，一般将Java堆分为新生代和老年代，
这样就可以根据各个代不带的特点选择最合适的回收算法，

# 垃圾收集器

## Serial 收集器
负责新生代的垃圾收集，它是单线程的（只使用一个线程收集），串行的（垃圾收集器与用户程序交替执行），
它是Client模式下的默认新生代收集器，其收集几十到一两百兆的新生代停顿时间可以控制在几十到一百多毫秒以内，
因为分配给客户端程序的内存一般不会很大，因此在Client模式下的该收集器收集垃圾导致的停顿时间是可以接受的。
它的有点是简单高效（和其他垃圾收集器相比），因为它是单线程的，没有线程切换的开销

## ParNew 收集器
负责新生代的垃圾收集，它是Serial的多线程版本，除了采用多个线程进行垃圾收集意外，其他的和Serival收集器基本相同。
默认开启的线程数量与 CPU 数量相同，可以使用`-XX:ParallelGCThreads`来设置线程数。它是Server模式下的首选的新生代垃圾收集器，
其中一个重要的非性能原因是，除了Serial以外，只有该收集器能和CMS收集器配合工作。

## Parallel Scavenge 收集器
负责新生代的垃圾收集，它是多线程的，并行的。该收集器关注的重点和其他收集器不同，比如CMS收集器关注的是缩短垃圾回收时用户线程停顿的时间，
而Parallel Scavenge收集器关注的是达到一个可控制的吞吐量。这里的吞吐量指的是运行用户代码的时间与CPU总运行时间的比值，
即吞吐量=运行用户代码时间/（运行用户代码时间+垃圾收集时间），比如运行了用户代码9分钟，垃圾收集用了1分钟，则吞吐量为90%。
停顿时间越短就越适合需要与用户交互的程序，良好的响应速度能提升用户体验。而高吞吐量则可以高效率地利用 CPU 时间，尽快完成程序的运算任务，适合在后台运算而不需要太多交互的任务。

缩短停顿时间是以牺牲吞吐量和新生代空间来换取的：新生代空间变小，垃圾回收变得频繁，导致吞吐量下降。

可以通过开关参数`--XX:+UseAdaptiveSizePolicy`打开 GC 自适应的调节策略（GC Ergonomics），就不需要手工指定新生代的大小（-Xmn）、Eden 和 Survivor 区的比例、晋升老年代对象年龄等细节参数了。
虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量。

## Serial Old 收集器
Serial收集器的老年代版本，使用标记-整理算法，一般在Client模式下使用

## Parallel Old收集器
Parallel Scavenge的老年代版本，多线程，使用标记-整理算法，在注重吞吐量以及CPU资源敏感的场合使用。

## CMS 收集器
以获得最短回收时间为目标的收集器，它基于标记-清除算法，一般用于注重系统响应速度的场合。它分为4个步骤：
* 初始标记(CMS initial mark)
* 并发标记(CMS concurrent mark)
* 重新标记(CMS remark)
* 并发清除(CMS concurrent sweep)

其中初始标记和重新标记需要使用户线程停顿，初始标记仅仅标记一下GC Roots能直接关联到的对象，速度很快，
并发标记是进行GC Roots Tracing的过程，而重新标记则是为了修正并发标记期间由于用户线程继续运作而导致标记产生变动的那一部分对象的标记记录，
这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短。

缺点：
* 读CPU资源敏感
* 无法处理浮动垃圾，可能出现`Concurrent Mode Failure`失败而导致另一次Full GC的产生。
* 会产生空间碎片

## G1 收集器
面向服务端应用的收集器，计划在未来用该收集器取代CMS收集器。该收集器有以下特点：
* 并发与并行：能充分利用多CPU的硬件优势，使用多个CPU减少用户线程停顿时间
* 分代收集：分代的概念依旧存在，但是它不需要于其他收集器配合就能回收掉整个堆的垃圾
* 空间整合：和CMS的标记-清除算法不同，G1从整体来看是给予标记-整理算法的，从局部来看是给予复制算法的，
两种算法都以为着G1回收垃圾后不会产生内存碎片
* 可预测的停顿：这是G1和CMS相比的一大优势，两者的目标都是减少用户线程停顿时间，但G1除了降低停顿时间以外，还建立了可预测的停顿时间模型，
能让使用者明确指定在一个长度为M毫秒的时间片内，垃圾收集消耗的时间不得超过N毫秒。

G1 把堆划分成多个大小相等的独立区域（Region），新生代和老年代不再物理隔离，基本步骤如下：
* 初始标记
* 并发标记
* 最终标记：为了修正在并发标记期间因用户程序继续运作而导致标记产生变动的那一部分标记记录，虚拟机将这段时间对象变化记录在线程的 Remembered Set Logs 里面，最终标记阶段需要把 Remembered Set Logs 的数据合并到 Remembered Set 中。这阶段需要停顿线程，但是可并行执行。
* 筛选回收：首先对各个 Region 中的回收价值和成本进行排序，根据用户所期望的 GC 停顿时间来制定回收计划。此阶段其实也可以做到与用户程序一起并发执行，但是因为只回收一部分 Region，时间是用户可控制的，而且停顿用户线程将大幅度提高收集效率。
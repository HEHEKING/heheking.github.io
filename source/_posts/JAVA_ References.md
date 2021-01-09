---
title: JAVA 引用
date: 2020/12/4 15:50:21
categories:
- 笔记
tags:
- java
- jvm
- gc
---

Java 中没有指针，到处都是引用(除了基本类型)。

Java 内存管理分为内存分配和内存回收，都不需要程序员负责，垃圾回收的机制主要是看对象是否有引用指向该对象。
<!-- more -->

**Java 对象的引用包括：**

1. 强引用
2. 软引用
3. 弱引用
4. 虚引用

**Java中提供这四种引用类型主要有两个目的：**

1. 第一是可以让程序员通过代码的方式决定某些对象的生命周期；
2. 第二是有利于 JVM 进行垃圾回收。

## 引用类型

### 一、强引用（Strong References）

> 效果：存在强引用的对象，不会被JVM回收。
>
> 当对象不再被引用时，JVM 进行垃圾回收时才会回收对象。

我们每天都在用强引用（如果你每天都在用 Java 的话），

指创建一个对象并把这个对象赋给一个引用变量。

一段如下的代码：

```java
HashMap mapRef = new HashMap();
```

就是通过`new HashMap();`创建了一个对象（这东西在heap上），并把一个强引用存到了 mapRef 引用中。而强引用之为“强”的地方就在于其对垃圾回收器所产生的影响。如果一个对象可以经由一条强引用链可达（也就说这个对象是Strongly reachable），那么就说明这个类不适合被垃圾回收。我们也绝对不希望正在使用的对象一下子了无踪迹了。

强引用有引用变量指向时永远不会被垃圾回收，JVM 宁愿抛出 **OutOfMemory** 错误也不会回收这种对象。

一段如下的代码：

```java
public class Main {  
    public static void main(String[] args) {  
        new Main().fun1();  
    }  
       
    public void fun1() {  
        Object object = new Object();  
        Object[] objArr = new Object[1000];  
    } 
}
```

当运行至`Object[] objArr = new Object[1000];`这句时，如果内存不足，JVM 会抛出 OOM 错误也不会回收object指向的对象。不过要注意的是，当 fun1() 运行完之后，object和 objArr 都已经不存在了，所以它们指向的对象都会被 JVM 回收。

如果想中断强引用和某个对象之间的关联，可以显示地将引用赋值为null，这样一来的话，JVM 在合适的时间就会回收该对象。

如 Vector 类的 clear 方法中就是通过将引用赋值为null来实现清理工作。

### 二、弱引用（Week References）

> 效果：存在弱引用的对象，每次 JVM 进行垃圾回收时，该对象都会被回收。
>
> 应用：短时间**缓存**某些次要数据。

弱引用也是用来描述非必需对象的，当JVM进行垃圾回收时，无论内存是否充足，都会回收被弱引用关联的对象。在 java 中，用 java.lang.ref.WeakReference 类来表示。

下面是使用示例：

```java
public class test {  
    public static void main(String[] args) {  
        WeakReference<People>reference=new WeakReference<People>(new People("zhouqian",20));  
        System.out.println(reference.get());  
        System.gc();//通知GVM回收资源  
        System.out.println(reference.get());  
    }  
}

class People{  
    public String name;  
    public int age;  
    public People(String name,int age) {  
        this.name=name;  
        this.age=age;  
    }  
    @Override  
    public String toString() {  
        return "[name:"+name+",age:"+age+"]";  
    }  
}
```

输出结果：

> [name:zhouqian,age:20]
> null

第二个输出结果是 null ，这说明只要 JVM 进行垃圾回收，被弱引用关联的对象必定会被回收掉。不过要注意的是，这里所说的被弱引用关联的对象是指只有弱引用与之关联，如果存在强引用同时与之关联，则进行垃圾回收时也不会回收该对象（软引用也是如此）。

将上述代码做一个小的更改：

```java
public class test {
    public static void main(String[] args) {
        People people=new People("zhouqian",20);
        WeakReference<People>reference=new WeakReference<People>(people);//关联强引用
        System.out.println(reference.get());
        System.gc();
        System.out.println(reference.get());
    }
}
```

输出结果：

> [name:zhouqian,age:20]
>
> [name:zhouqian,age:20]

**弱引用**可以和一个**引用队列**（ReferenceQueue）联合使用，如果弱引用所引用的对象被 JVM 回收，这个弱引用就会被加入到与之关联的引用队列中。

### 三、软引用（Soft References）

> 效果：存在软引用的对象，在**内存不足**时，才会被JVM回收。
>
> 应用：缓存数据，提高数据的获取速度。

如果一个对象具有软引用，内存空间足够，垃圾回收器就不会回收它；

如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。

软引用可用来实现内存敏感的高速缓存,比如网页缓存、图片缓存等。使用软引用能防止内存泄露，增强程序的健壮性。
SoftReference的特点是它的一个实例保存对一个Java对象的软引用， 该软引用的存在不妨碍垃圾收集线程对该Java对象的回收。

也就是说，一旦SoftReference保存了对一个Java对象的软引用后，在垃圾线程对 这个Java对象回收前，SoftReference类所提供的get()方法返回Java对象的强引用。

另外，一旦垃圾回收器回收该Java对象之 后，get()方法将返回null。

示例：

```java
Object obj = new Object();
SoftReference softRef = new SoftReference(obj);
```

此时，对于这个 Object 对象，有两个引用路径，一个是来自 SoftReference 对象的软引用，一个来自变量 obj 的强引用，所以这个 Object 对象是强可及对象。

随即，我们可以结束 obj 对这个MyObject实例的强引用:

```java
obj = null;
```

此后，这个 Object 对象成为了软引用对象。如果垃圾收集线程进行内存垃圾收集，**并不会因为有一个 SoftReference 对该对象的引用而始终保留该对象**。
Java虚拟机的垃圾收集线程对软可及对象和其他一般Java对象进行了区别对待:软可及对象的清理是由垃圾收集线程根据其特定算法按照内存需求决定的。
也就是说，垃圾收集线程会在虚拟机抛出 OutOfMemoryError 之前回收软可及对象，而且虚拟机会尽可能优先回收长时间闲置不用的软可及对象，对那些刚刚构建的或刚刚使用过的“新”软可反对象会被虚拟机尽可能保留。

在回收这些对象之前，我们可以通过:

```java
//重新获得对该实例的强引用。
Object obj2 = (Object)softRef.get();
```

如果是在回收之后调用get()方法，则得到 null。

**软引用**与**引用队列**：

作为一个 Java 对象，SoftReference 对象除了具有保存软引用的特殊性之外，也具有 Java 对象的一般性。

所以，当软可及对象**被回收之后**，虽然这个 SoftReference 对象的 get() 方法返回 null ,但这个 SoftReference 对象已经不再具有存在的价值，需要一个适当的清除机制，避免大量 SoftReference 对象带来的**内存泄漏**。在 java.lang.ref 包里还提供了 ReferenceQueue。如果在创建 SoftReference 对象的时候，使用了一个 ReferenceQueue 对象作为**参数**提供给 SoftReference 的**构造方法**，如:

```java
ReferenceQueue queue = new  ReferenceQueue();  
SoftReference ref = new  SoftReference(obj, queue); 
```

那么当这个 SoftReference 所软引用的 obj 被垃圾收集器回收的同时，ref 所强引用的 SoftReference 对象被列入 ReferenceQueue。也就是说，ReferenceQueue 中保存的对象是 **Reference 对象**，而且是**已经失去了它所软引用的对象**的 Reference 对象。另外从ReferenceQueue 这个名字也可以看出，它是一个队列，当我们调用它的 poll() 方法的时候，如果这个队列中不是空队列，那么将返回队列前面的那个Reference对象。

> Queue 中 remove() 和 poll() 都是用来从队列头部删除一个元素。
> 在队列元素为空的情况下，remove() 方法会抛出 NoSuchElementException 异常，poll() 方法只会返回 null 。

在任何时候，我们都可以调用 ReferenceQueue 的 poll() 方法来检查是否有它所关心的非强可及对象被回收。如果队列为空，将返回一个null,否则该方法返回队列中前面的一个Reference对象。利用这个方法，我们可以检查哪个SoftReference所软引用的对象已经被回收。于是我们可以把这些失去所软引用的对象的SoftReference对象清除掉。

常用的方式为:

```java
SoftReference ref = null;  
while ((ref = (EmployeeRef) q.poll()) != null) {  
    // 清除ref
}
```

### 四、虚引用（Phantom References）

> 相当于无引用，使对象无法被使用，必须与引用队列配合使用。
>
> 使对象进入不可用状态，等待下次JVM垃圾回收，从而使对象进入引用列队中。

虚引用和前面的软引用、弱引用不同，它并不影响对象的生命周期。在 java 中用 java.lang.ref.PhantomReference 类表示。如果一个对象与虚引用关联，则跟没有引用与之关联一样，在**任何时候**都可能被垃圾回收器回收。

要注意的是，**虚引用必须和引用队列关联使用**，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会把这个虚引用加入到与之关联的引用队列中。

程序可以**通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收**。

如果程序发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

```java
import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;
public class Main {
    public static void main(String[] args) {
        ReferenceQueue<String> queue = new ReferenceQueue<String>();
        PhantomReference<String> phantomRef = new PhantomReference<String>(new String("hello"), queue);
        System.out.println(pr.get());
    }
} 
```

## 引用队列（ReferenceQueue）

引用队列可以配合软引用、弱引用及幽灵引用使用，当引用的对象将要被 JVM 回收时，会将其加入到引用队列中。

PS：为什么不叫回收队列呢？？？

通过引用队列可以了解JVM垃圾回收情况。

```java
// 引用队列
ReferenceQueue<String> rq = new ReferenceQueue<String>();
// 软引用
SoftReference<String> sr = new SoftReference<String>(new String("Soft"), rq);
// 弱引用
WeakReference<String> wr = new WeakReference<String>(new String("Weak"), rq);
// 虚引用
PhantomReference<String> pr = new PhantomReference<String>(new String("Phantom"), rq);
// 从引用队列中弹出一个对象引用
Reference<? extends String> ref = rq.poll();
```

## 整理来源

Java 的四种引用方式：https://blog.csdn.net/linzhiqiang0316/article/details/88591907

四种引用类型和引用队列：https://blog.csdn.net/cuixianlong/article/details/78617577

Java 中的引用：https://www.cnblogs.com/czx1/p/10665327.html

Queue 中 remove() 和 poll() 区别：https://blog.csdn.net/meism5/article/details/89884257
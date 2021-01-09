---
title: Java 对象序列化
date: 2021/1/7 14:02:49
categories:
- 笔记
tags:
- java
toc:
- true
---

简单的来说，Java 对象序列化就是把对象写入到输出流中，用来存储或传输；反序列化就是从输入流中读取对象。

注意对象的序列化是基于字节的，不能使用基于字符的流。

<!-- more -->

Java 允许我们在内存中创建可复用的 Java 对象，但一般情况下，这些对象的生命周期不会比 JVM 的生命周期更长。但在现实应用中，可能要求在 JVM 停止运行之后能够保存(持久化)指定的对象，并在将来重新读取被保存的对象

Java 对象序列化就能够帮助我们实现该功能。使用 Java 对象序列化，在保存对象时，会把其状态保存为一组字节，在未来再将这些字节组装成对象

**必须注意地是，对象序列化保存的是对象的 "状态" ，即它的成员变量。由此可知，对象序列化不会关注类中的静态变量**

除了在持久化对象时会用到对象序列化之外，当使用 RMI ，或在网络中传递对象时，都会用到对象序列化

Java 序列化 API 为处理对象序列化提供了一个标准机制，该 API 简单易用，但性能不是最好的

## 实例 Demo

```Java
// Gender类，表示性别
// 每个枚举类型都会默认继承类java.lang.Enum，而Enum类实现了Serializable接口，所以枚举类型对象都是默认可以被序列化的。
public enum Gender {  
    MALE, FEMALE  
} 

// Person 类实现了 Serializable 接口，它包含三个字段。另外，它还重写了该类的 toString() 方法，以方便打印 Person 实例中的内容。
public class Person implements Serializable {  
    private String name = null;  
    private Integer age = null;  
    private Gender gender = null;  

    public Person() {  
        System.out.println("none-arg constructor");  
    }  
 
    public Person(String name, Integer age, Gender gender) {  
        System.out.println("arg constructor");  
        this.name = name;  
        this.age = age;  
        this.gender = gender;  
    }  
 
    // 省略 set get 方法
    @Override 
    public String toString() {  
        return "[" + name + ", " + age + ", " + gender + "]";  
    }  
} 
 
// SimpleSerial类，是一个简单的序列化程序，它先将Person对象保存到文件person.out中，然后再从该文件中读出被存储的Person对象，并打印该对象。
public class SimpleSerial {  
    public static void main(String[] args) throws Exception {  
        File file = new File("person.out");  
        ObjectOutputStream oout = new ObjectOutputStream(new FileOutputStream(file)); // 注意这里使用的是 ObjectOutputStream 对象输出流封装其他的输出流
        Person person = new Person("John", 101, Gender.MALE);  
        oout.writeObject(person);  
        oout.close();  
 
        ObjectInputStream oin = new ObjectInputStream(new FileInputStream(file));  // 使用对象输入流读取序列化的对象
        Object newPerson = oin.readObject(); // 没有强制转换到Person类型  
        oin.close();  
        System.out.println(newPerson);  
    }  
} 
 
// 上述程序的输出的结果为：
arg constructor  
[John, 31, MALE]
```

## Serializable 接口

String类型的对象、枚举类型的对象、数组对象，都是默认可以被序列化的。如果一个类需要被序列化，需要实现 Serializable 接口进行自动序列化，或者实现 Externalizable 接口进行手动序列化，否则强行序列化该类的对象，就会抛出 NotSerializableException 异常，这是因为，在序列化操作过程中会对类型进行检查，要求被序列化的类必须属于 Enum、Array 和 Serializable 类型其中的任何一种（Externalizable也继承了Serializable）。

JVM 是否允许反序列化，不仅取决于类路径和功能代码是否一致，一个非常重要的一点是两个类的序列化 ID 是否一致（就是 private static final long serialVersionUID）

### 默认序列化机制

如果仅仅让某个类实现 Serializable 接口，而没有其它任何处理的话，则就是使用默认序列化机制。

使用默认机制在序列化对象时，不仅会序列化当前对象，还会对该对象引用的其它对象也进行序列化，同样地，这些其它对象引用的另外对象也将被序列化，以此类推。

**所以，如果一个对象包含的成员变量是容器类对象，而这些容器所含有的元素也是容器类对象，那么这个序列化的过程就会较复杂，开销也较大。**

### transient 关键字

有些时候不能使用默认序列化机制。比如，希望在序列化过程中忽略掉敏感数据，或者简化序列化过程。

当类的某个字段被 transient 修饰，默认序列化机制就会忽略该字段。此处将 Person 类中的 age 字段声明为 transient ，如下所示

```java
public class Person implements Serializable {  
    ...  
    transient private Integer age = null;  
    ...  
} 

// 再执行SimpleSerial应用程序，会有如下输出：
arg constructor  
[John, null, MALE]
```

transient 关键字的作用是控制变量的序列化，在变量声明前加上该关键字，可以阻止该变量被序列化到文件中，在被反序列化后，transient 变量的值被设为初始值，如 int 型的是 0，对象型的是 null。

FileOutputStream 类有一个带有两个参数的重载 Constructor —— FileOutputStream(String, boolean) 。若其第二个参数为 true 且 String 代表的文件存在，那么将把新的内容写到原来文件的末尾而非重写这个文件，故不能用这个版本的构造函数来实现序列化，也就是说必须重写这个文件，否则在读取这个文件反序列化的过程中就会抛出异常，导致只有第一次写到这个文件中的对象可以被反序列化，之后程序就会出错。

### writeObject() 与 readObject()

对于上述已被声明为 transitive 的字段 age，除了将 transient 关键字去掉外，是否还有其它方法能使它再次可被序列化？

方法之一就是在 Person 类中添加两个方法：writeObject() 与 readObject() ，如下所示：

```Java
public class Person implements Serializable {  
    ...  
    transient private Integer age = null;  
    ...  
 
    // writeObject()会先调用ObjectOutputStream中的defaultWriteObject()方法，该方法会执行默认的序列化机制，此时会忽略掉age字段。然后再调用writeInt()方法显示地将age字段写入到        
    // ObjectOutputStream中。
    private void writeObject(ObjectOutputStream out) throws IOException {  
        out.defaultWriteObject();  
        out.writeInt(age);  
    }  
 
    // readObject()的作用则是针对对象的读取，其原理与writeObject()方法相同。
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {  
        in.defaultReadObject();  
        age = in.readInt();  
    }  
} 
// 再次执行SimpleSerial应用程序，则又会有如下输出：
arg constructor  
[John, 31, MALE]
```

必须注意地是，writeObject() 与 readObject() 都是 private 方法，那么它们是如何被调用的呢？

毫无疑问，使用**反射**。详情可以看看 ObjectOutputStream 中的 writeSerialData 方法，以及 ObjectInputStream 中的 readSerialData 方法。这两个方法会在序列化、反序列化的过程中被自动调用。且不能关闭流，否则会导致序列化操作失败。

## Externalizable 接口

无论是使用 transient 关键字，还是使用 writeObject() 和 readObject() 方法，其实都是基于 Serializable 接口的序列化。

Java提供了另一个序列化接口 Externalizable，使用该接口之后，之前基于 Serializable 接口的序列化机制就将失效。Externalizable 接口继承于 Serializable 接口，当使用该接口时，序列化的细节需要由程序员去完成。将Person类作如下修改：

```Java
public class Person implements Externalizable {  
    private String name = null;  
    transient private Integer age = null;  
    private Gender gender = null;  
 
    public Person() {  
        System.out.println("none-arg constructor");  
    }  
 
    public Person(String name, Integer age, Gender gender) {  
        System.out.println("arg constructor");  
        this.name = name;  
        this.age = age;  
        this.gender = gender;  
    }  
 
    private void writeObject(ObjectOutputStream out) throws IOException {  
        out.defaultWriteObject();  
        out.writeInt(age);  
    }  
 
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {  
        in.defaultReadObject();  
        age = in.readInt();  
    }  
 
    @Override 
    public void writeExternal(ObjectOutput out) throws IOException {  
    }  
 
    @Override 
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {  
    }  
    ...  
} 

// 此时再执行SimpleSerial程序,会得到如下结果：
arg constructor  
none-arg constructor  
[null, null, null] 
 
// 从该结果，一方面可以看出Person对象中任何一个字段都没有被序列化。另一方面，这次序列化过程调用了Person类的无参构造器。
```

Externalizable 继承于 Serializable，当使用该接口时，序列化的细节需要由程序员去完成。

如上所示的代码，由于实现的 writeExternal() 与 readExternal() 方法未作任何处理，那么该序列化行为将不会保存/读取任何一个字段。这也就是为什么输出结果中所有字段的值均为空。

另外，使用 Externalizable 接口进行序列化时，读取对象会调用被序列化类的无参构造器去创建一个新的对象，然后再将被保存对象的字段的值分别填充到新对象中，这就是为什么在此次序列化过程中 Person 类的无参构造器会被调用。

**由于这个原因，实现 Externalizable 接口的类必须要提供一个无参构造器，且它的访问权限为public。**

对上述Person类做进一步的修改，使其能够对name与age字段进行序列化，但忽略 gender 字段：

```Java
public class Person implements Externalizable {  
    private String name = null;  
    transient private Integer age = null;  
    private Gender gender = null;  
 
    public Person() {  
        System.out.println("none-arg constructor");  
    }  
 
    public Person(String name, Integer age, Gender gender) {  
        System.out.println("arg constructor");  
        this.name = name;  
        this.age = age;  
        this.gender = gender;  
    }  
 
    private void writeObject(ObjectOutputStream out) throws IOException {  
        out.defaultWriteObject();  
        out.writeInt(age);  
    }  
 
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {  
        in.defaultReadObject();  
        age = in.readInt();  
    }  
 
    @Override 
    public void writeExternal(ObjectOutput out) throws IOException {  
        out.writeObject(name);  
        out.writeInt(age);  
    }  
 
    @Override 
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {  
        name = (String) in.readObject();  
        age = in.readInt();  
    }  
    ...  
} 
 
// 执行SimpleSerial之后会有如下结果：
arg constructor  
none-arg constructor  
[John, 31, null]
```

## 序列化时对象的引用

序列化并不保存静态变量

要想将父类对象也序列化，就需要让父类也实现 Serializable 接口

若一个类的字段有引用对象，那么在序列化该类的时候不仅该类要实现Serializable接口，这个引用类型也要实现Serializable接口。但有时我们并不需要对这个引用类型进行序列化，此时就需要使用transient关键字来修饰该引用类型保证在序列化的过程中跳过该引用类型。

通过序列化操作，可以实现对任何可 Serializable 对象的深度复制（deep copy），这意味着复制的是整个对象的关系网，而不仅仅是基本对象及其引用

如果父类没有实现 Serializable 接口，但其子类实现了此接口，那么这个子类是可以序列化的，但是在反序列化的过程中会调用父类的无参构造函数，所以在其直接父类（注意是直接父类）中必须有一个无参的构造函数。

## 关于安全性

服务器端给客户端发送序列化对象数据，序列化二进制格式的数据写在文档中，并且完全可逆。一抓包就能就看到类是什么样子，以及它包含什么内容。如果对象中有一些数据是敏感的，比如密码字符串等，则要对字段在序列化时，进行加密，而客户端如果拥有解密的密钥，只有在客户端进行反序列化时，才可以对密码进行读取，这样可以一定程度保证序列化对象的数据安全。 

比如可以通过使用 writeObject 和 readObject 实现密码加密和签名管理，但其实还有更好的方式。

如果需要对整个对象进行加密和签名，最简单的是将它放在一个 javax.crypto.SealedObject 或 java.security.SignedObject 包装器中。两者都是可序列化的，所以将对象包装在 SealedObject 中可以围绕原对象创建一种 “包装盒”。必须有对称密钥才能解密，而且密钥必须单独管理。同样，也可以将 SignedObject 用于数据验证，并且对称密钥也必须单独管理

## 反序列化时的对象是否与原来相同

只要将对象序列化到单一流中，就可以恢复出与我们写出时一样的对象网，而且只要在同一流中，对象都是同一个。否则，反序列化后的对象地址和原对象地址不同，只是内容相同

如果将一个对象序列化入某文件，那么之后又对这个对象进行修改，然后再把修改的对象重新写入该文件，那么修改无效，文件保存的序列化的对象仍然是最原始的。这是因为，序列化输出过程跟踪了写入流的对象，而试图将同一个对象写入流时，并不会导致该对象被复制，而只是将一个句柄写入流，该句柄指向流中相同对象的第一个对象出现的位置。为了避免这种情况，在后续的 writeObject() 之前调用 out.reset() 方法，这个方法的作用是清除流中保存的写入对象的记录

## ArrayList 序列化

ArrayList 实现了 java.io.Serializable 接口，但是其 elementData 是 transient 的，但是 ArrayList 是通过数组实现的，数组 elementData 用来保存列表中的元素。通过该属性的声明方式知道该数据无法通过序列化持久化。

但是如果实际测试，就会发现，ArrayList 能被完整的序列化，原因是在 writeObject 和 readObject 方法中进行了序列化的实现。

这样设计的原因是因为 ArrayList 是动态数组，如果数组自动增长长度设为 2000，而实际只放了一个元素，那就会序列化 1999 个 null 元素，为了保证在序列化的时候不会将这么多 null 元素序列化，ArrayList 把元素数组设置为 transient ，但是，作为一个集合，在序列化过程中还必须保证其中的元素可以被持久化，所以，通过重写 writeObject 和 readObject 方法把其中的元素保留下来，具体做法是：

writeObject 方法把 elementData 数组中的元素遍历到 ObjectOutputStream

readObject 方法从 ObjectInputStream 中读出对象并保存赋值到 elementData 数组

## 整理来源

Java对象序列化全面总结：https://www.cnblogs.com/kubixuesheng/p/10350523.html#_label2


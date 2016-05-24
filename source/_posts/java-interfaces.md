---
title: 详解java接口
tags:
  - java
date: 2016-05-24 18:02:44
---


java中有一种概念叫`interfaces`，java中的interface只能包含方法签名和成员变量，而不能包含方法的实现，只有方法签名（方法名，参数，异常）。

可以使用interface来实现java的多态性，后文将讲解多态的部分。

## java interface 实例

下面是一个简单的java interface例子：

```java
public interface MyInterface {
    public String hello = "Hello";

    public void sayHello();
}
```

接口使用`interface`关键字来定义，跟类一样，接口可以定义为`public`或者`package scope`（省略访问控制符）。

上面的接口例子中包含一个变量和一个方法，其中变量可以直接通过接口访问到：

```java
System.out.println(MyInterface.hello);
```

可以看到，访问接口的变量跟访问类的静态成员变量的方式很像。

接口中的方法，需要被其他的类来实现才能被访问，下面将讲述如何实现接口的方法。

<!--more-->

## 实现接口

在使用接口之前，必须在类中实现接口的方法。下面是实现`MyInterface`接口的类：

```java
public class MyInterfaceImpl implements MyInterface {
    public void sayHello() {
      System.out.println(MyInterface.hello);
    }
}
```

上面类的定义中`implements MyInterface`部分就是用来告诉编译器`MyInterfaceImpl`类实现了`MyInterface`接口。

实现接口的类必须实现接口中定义的所有方法，而且方法必须和接口中定义的是一致的签名（方法名+参数），类不需要定义接口中的变量。

### interface实例

当类实现了某个接口，可以使用实现类来定义一个接口的实例。下面是实例：

```java
MyInterface myInterface = new MyInterfaceImpl();

myInterface.sayHello();
```

上面的myInterface变量定义为`MyInterface`接口类型，而创建的对象是`MyInterfaceImpl`类的实例。java允许这么定义因为`MyInterfaceImpl`类实现了`MyInterface`接口。可以引用`MyInterfaceImpl`的实例作为`MyInterface`接口的实例。

## 实现多个接口

java中类可以实现多个接口，这种情况下类必须实现所有接口中的方法，下面是实例：

```
public class MyInterfaceImpl
    implements MyInterface, MyOtherInterface {

    public void sayHello(){
        System.out.println("Hello");
    }

    public void sayGoodbye(){
        System.out.println("Goodbye");
    }
}
```

上面的类实现了两个接口`MyInterface`和`MyOtherInterface`，在`implements`关键字后面列出实现的多个接口名称，以逗号分隔即可。

如果接口不是定义在相同的package里面，需要import这个接口，跟引入类一样使用`import`关键字来引入接口。比如：

```java
import com.jenkov.package1.MyInterface;
import com.jenkov.package2.MyOtherInterface;

public class MyInterfaceImpl implements MyInterface, MyOtherInterface {
    ...
}
```

下面是被上面类实现的两个接口:

```java
public interface MyInterface {
    public void sayHello();
}
```

```java
public interface MyOtherInterface {
    public void sayGoodbye();
}
```

每个接口包含一个方法，这些方法都被`MyInterfaceImpl`类实现了。

### 重载方法签名

如果类实现了多个接口，这里有一个风险就是这些接口中可能有相同的方法签名（方法名+参数），因为java的类只能实现一个方法签名，这样会导致错误。

java规范并没有给出这种问题的解决办法，这取决于开发者自己的处理方式。

## 接口变量

java中接口可以包含变量和常量，然而通常不会在接口中定义常量，在某些情况下通常是在接口中定义常量。特别是常量被实现类使用的情况，比如计算或者作为方法的参数，然而我的建议是尽量避免在接口在定义变量。

接口中所有的变量都是public的，即便你省略了`public`关键字定义。

## 接口方法

接口中可以定义一个或者多个方法，上面提到过，不能包含任何的方法实现，具体的实现细节有具体的实现类来完成。

接口中所有的方法都是public的，即便你省略了`public`关键字定义。

## 接口默认方法

在java8之前接口不能包含方法的实现，然而这会导致当API需要添加一个新的接口方法，那么所有实现该接口的类都得实现这个新加的方法，如果实现类都是定义在API之内没有什么问题，但是别的使用该API的client会导致错误。

举个🌰，下面的接口是一个开放的API供很多应用调用：

```java
public interface ResourceLoader {
    Resource load(String resourcePath);
}
```

假如一个项目使用这个API实现了`ResourceLoader`接口：

```java
public class FileLoader implements ResourceLoader {

    public Resource load(String resourcePath) {
        // in here is the implementation +
        // a return statement.
    }
}
```

如果API的开发者想要给`ResourceLoader`接口增加一个新的方法，那么当项目更新了新的API之后，`FileLoader`类将会报错。

为了避免上面的问题，java8中接口引入了`interface default methods`的概念，接口默认方法可以包含一个默认的实现，实现了该接口但是没有实现该方法的类将会自动调用这个默认方法实现。

可以在接口定义中使用`default`关键字来定义默认方法实现，下面是给`ResourceLoader`增加默认方法实现的例子：

```java
public interface ResourceLoader {
    Resource load(String resourcePath);

    default Resource load(Path resourcePath){
        // provide default implementation to load
        // resource from a Path and return the content
        // in a Resource object.
    }
}
```

上面的例子添加默认方法`load(Path)`，但方法的实现留空因为接口根本不需要关系接口的实现。

类可以重写接口中的默认方法，重写的方法将会覆盖接口的默认方法实现。

## 接口和继承

java中的接口可以继承别的接口，就像类可以继承别的类一样，可以使用`extends`关键字来定义继承关系，下面是接口继承的示例：

```java
public interface MySuperInterface {
    public void sayHello();
}
```

```java
public interface MySubInterface extends MySuperInterface {
    public void sayGoodbye();
}
```

`MySubInterface`接口继承自`MySuperInterface`接口，意味着`MySubInterface`接口继承了`MySuperInterface`接口所有的成员变量和方法，实现`MySubInterface`接口的类必须同时实现`MySubInterface`和`MySuperInterface`接口中定义的所有方法。

子接口中可能会定义跟父接口相同方法签名的方法，你会在你的代码中找到这样的设计吗？

跟类不同，接口可以直接继承多个父接口，`extends`关键字后面列出继承的接口，以逗号分隔，实现继承多个接口的接口的时候需要实现所有接口中的所有方法。

下面是接口继承多个父接口的示例：

```java
public interface MySubInterface extends SuperInterface1, SuperInterface2 {

    public void sayItAll();
}
```

当实现多个接口的时候，并没有统一的规范来处理多个接口中有相同方法签名的情况。

### 继承和默认方法

接口的默认方法增加了接口继承的复杂性。当一个类实现了多个接口，并且多个接口中可能相同方法签名的方法，但可能不是所有方法都有默认方法，也就是说两个接口包含相同方法签名的方法，而且其中一个定义了默认方法，那么这个类不能自动实现这两个方法。

这种情况也会出现在接口继承中。

上面的情况java编译器需要类实现接口中会导致问题的方法，类的实现会覆盖任何默认的实现。

## 继承和多态

java中接口是实现多态性的一种方式。多态的意思是一个类的实例可以被当做不同的类型（类或者接口）来使用。

![Two parallel class hierarchies used in the same application.](http://kikoroc.qiniudn.com/polymorphism.png)

上面的类展示了vehicles和drivers的不同类型，这些类的作用是构造现实生活中真实的实体。

现在想象你需要保存这些对象到数据库中，并且序列化为xml，json等数据格式，每个操作定义一个方法，对所有的`Car`,`Trunk`或者`Vehicle`对象可用，一个`store()`方法，`serializeToXML()`,`serializeToJSON()`。

先忘掉接口多态这些概念，直接在对象中定义这些方法并实现，这样会导致类层次结构的混乱，假设这就是你所想到的实现方法。

再看看上面的结构图，你会在哪里定义这三个方法，然后所有的类都能访问这些方法？

解决这个问题的一种方法就是为`Vehicle`和`Driver`类创建一个共用的父类，父类包含存储和序列化的方法，然而这样会导致概念混淆，类结构不再是vehicles和drivers模型。

更好的实现方式是创建包含存储和序列化方法的接口，然后让这些类实现这些接口。下面是这种方式的示例：

```java
public interface Storable {
    public void store();
}
```

```java
public interface Serializable {
    public void serializeToXML(Writer writer);
    public void serializeToJSON(Writer writer);
}
```

当类实现上面两个接口和里面的方法后，你可以通过将对象强转为接口的实例来访问这些接口，不需要知道所给的类是什么，只需要知道它实现了什么接口，下面的示例：

```java
Car car = new Car();

Storable storable = (Storable) car;
storable.store();

Serializable serializable = (Serializable) car;
serializable.serializeToXML(new FileWriter("car.xml"));
serializable.serializeToJSON(new FileWriter("car.json"));
```

现在你会发现接口提供一种比继承更清晰的方式来实现跨类的方法。

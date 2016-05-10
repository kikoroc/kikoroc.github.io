---
title: 详解java抽象类和模板方法设计模式
tags:
  - java
date: 2016-05-10 13:07:37
---


java中的抽象类是不能被实例化的，意味着你不能创建一个抽象类的实例。抽象类的目的是为了作为子类的基类，本文将介绍如何在java中创建抽象类以及使用抽象类的一些规则。

## 定义抽象类
java中定义抽象类很简单，在类的定义中加入`abstract`关键字即可。下面是java中定义抽象类的实例：
``` java
public abstract class MyAbstractClass {

}
```
上面就是java中定义抽象类的所有内容，现在你不能创建`MyAbstractClass`的实例，所以下面的代码不再合法：
``` java
MyAbstractClass myClassInstance =
    new MyAbstractClass(); // not valid
```
如果你尝试编译上面的代码，编译器将报错，告诉你不能实例化`MyAbstractClass`，因为它是一个抽象类。

## 抽象方法
抽象类中可以定义抽象方法，定义抽象方法也很简单，在方法的定义前面添加`abstract`关键字即可，下面是定义抽象方法的实例：
```java
public abstract class MyAbstractClass {
    public abstract void abstractMethod();
}
```
抽象方法是没有具体实现细节的，仅仅是方法的定义，跟`java interface`中的方法类似。

<!--more-->
如果一个类中包含抽象方法，那么这个类必须定义为`abstract`抽象类。但并不要求抽象类中所有方法都是抽象方法。抽象类中既可包含抽象方法也可以包含非抽象方法。

抽象类的子类必须实现(override)父抽象类中所有的抽象方法，非抽象方法直接继承到子类，如有需要也可以重写非抽象方法。
下面是`MyAbstractClass`抽象类子类的实例：
``` java
public class MySubClass extends MyAbstractClass {
    public void abstractMethod() {
        System.out.println("My method implementation");
    }
}
```
注意`MySubClass`必须实现抽象父类`MyAbstractClass`中的抽象方法`abstractMethod`。

抽象类的子类不用实现父类中的抽象方法唯一的情况是：子类也是抽象类。

## 抽象类的作用
抽象类的作用是为了作为子类的基类，这样子类很容易继承父类来做具体的细节实现。比如，假如一个流程需要3个步骤：

1. 具体动作之前的操作
2. 执行具体的动作
3. 具体动作结束后的操作

如果第一步和第三步的实现总是相同的，那么这个三步走的流程可以用java抽象类这么实现：
```java
public abstract class MyAbstractProcess {
    public void process() {
        stepBefore();
        action();
        stepAfter();
    }

    public void stepBefore() {
        //直接在抽象父类中实现
    }

    public abstract void action(); // 由子类来实现

    public void stepAfter() {
        //直接在抽象父类中实现
    }
}
```
注意到`action()`方法是抽象方法，`MyAbstractProcess`的子类现在可以继承`MyAbstractProcess`然后重写`action()`方法。

当子类的`process()`方法被调用，那么整个三步走的流程都会调用，包括父类中的`stepBefore()`和`stepAfter()`方法以及子类中实现的`action()`方法。

当然，`MyAbstractProcess`作为基类不一定非要是抽象类，`action()`也不是必须要是抽象方法的，你可以直接使用普通类来实现同样的逻辑，但是，让具体的实现方法和类抽象化，你可以清晰的告诉使用者这个类不能直接使用，它应该作为基类，然后让子类来实现抽象方法。

上面实例中`action()`方法没有默认的实现，是某些情况下父类也可以有默认的实现，子类也可以重写该方法，当然这种情况下这个方法就不能再定义为抽象方法，但是你依旧可以将类定义为抽象类，即使类里面没有抽象方法。

下面是一个更具体的实例用来打开一个URL，处理数据然后关闭链接。
```java
public abstract class URLProcessorBase {

    public void process(URL url) throws IOException {
        URLConnection urlConnection = url.openConnection();
        InputStream input = urlConnection.getInputStream();

        try{
            processURLData(input);
        } finally {
            input.close();
        }
    }

    protected abstract void processURLData(InputStream input)
        throws IOException;

}
```
`processURLData()`是个抽象方法，`URLProcessorBase`是个抽象类，那么`URLProcessorBase`的子类必须要实现父类的抽象方法`processURLData()`，因为它是一个抽象方法。

`URLProcessorBase`的子类可以直接处理url下载的数据，而不用担心如何打开和关闭url链接。因为这些都由父类实现了，子类只需要关注`processURLData()`方法中的`InputStream`对象，这样可以使得可以很方便的实现url数据处理的子类。

下面是一个子类的实例：
```java
public class URLProcessorImpl extends URLProcessorBase {

    @Override
    protected void processURLData(InputStream input) throws IOException {
        int data = input.read();
        while(data != -1){
            System.out.println((char) data);
            data = input.read();
        }
    }
}
```
注意到子类仅仅只需要重写`processURLData()`方法，其他的方法直接从父类继承过来。

下面是如何使用`URLProcessorImpl`类的实例：
```java
URLProcessorImpl urlProcessor = new URLProcessorImpl();

urlProcessor.process(new URL("http://jenkov.com"));
```
父类`URLProcessorBase`中实现的`process()`方法直接调用，然后会调用`URLProcessorImpl`类中的`processURLData()`方法
。

## 抽象类和模板方法设计模式
上面的`URLProcessorBase`类的实例代码实际上就是一种**模板方法设计模式**，模板方法设计模式提供了流程中一部分的实现逻辑，然后子类可以继承基类来完成全部的实现。

---
译自：[http://tutorials.jenkov.com/java/abstract-classes.html](http://tutorials.jenkov.com/java/abstract-classes.html)，能力有限，详细细节可以查阅原文。

---

## 模板方法设计模式简述
上面提到抽象类和模板方法设计模式的关系，这里也简单总结下模板设计方法。
### 概述
定义一个操作中的算法框架，而将具体的实现细节推迟到子类中实现。这样可以使子类不改变算法结构来重新定义算法的某些特定逻辑。
### 角色
抽象类：实现模板方法，定义算法框架。
具体类：实现抽象类中的抽象方法，完成完整的算法逻辑。
### 优缺点及适用场景
1. 优点
  * 模板方法通过将不变的行为在基类中实现，去除了子类中的重复代码；
  * 在子类中负责具体的细节实现，有利于算法的扩展；
  * 通过父类调用子类实现的操作，通过子类扩展增加新的行为，符合“开闭原则”。
2. 缺点
每个不同的实现都需要定义一个子类，这会导致类个数的增加，设计更加抽象。
3. 适用场景
  * 在某些类的算法中，用了相同的方法，造成代码的重复；
  * 控制子类的扩展，子类必须遵守算法的框架规则。

### 代码示例
抽象类：
```java
public abstract class AbstractClass {
    // 定义算法的一些抽象行为，让子类负责实现
    public abstract void step1();
    public abstract void step2();
    // 模板方法，定义算法的框架，调用抽象方法，他们将在子类中实现。
    public void TemplateMethod() {
        step1();
        step2();
        System.out.println("Completed.");
    }
}
```
具体类A：
```java
public class ConcreteClassA extends AbstractClass {
    @Override
    public void step1() {
        System.out.println("step1 of ConcreteClassA");
    }
    @Override
    public void step2() {
        System.out.println("step2 of ConcreteClassA");
    }
}
```
具体类B：
```java
public class ConcreteClassB extends AbstractClass {
    @Override
    public void step1() {
        System.out.println("step1 of ConcreteClassB");
    }
    @Override
    public void step2() {
        System.out.println("step2 of ConcreteClassB");
    }
}
```
调用：
```java
AbstractClass abstractClass = new ConcreteClassA();
// 调用A的实现
abstractClass.TemplateMethod();

abstractClass = new ConcreteClassB();
// 调用B的实现
abstractClass.TemplateMethod();
```
输出：
```
step1 of ConcreteClassA
step2 of ConcreteClassA
Completed.
step1 of ConcreteClassB
step2 of ConcreteClassB
Completed.
```

### 与抽象工厂模式的区别
模板方法是一种算法模板，针对算法的扩展重写；而抽象工厂模式是为了创建对象，针对对象的扩展重写。

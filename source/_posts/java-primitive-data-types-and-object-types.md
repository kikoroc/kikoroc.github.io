---
title: java为什么有基本数据类型？
date: 2016-05-08 00:18:49
tags: [java]
---

说到java语言的数据类型，我们都知道包括**基本数据类型**和**对象类型**，而且java是一门面向对象的语言，对象类型可以理解，基本数据类型会使得java的面向对象不是那么纯粹，那么为什么java还要保留基本数据类型呢？
先看基本类型和对象类型的一些区别：

## 区别
### 基本类型是基于值，对象类型是基于引用
``` java
int i1 = 100;
Integer i2 = new Integer(100);
Integer i3 = 100;
```
第一句是使用基本类型int，第二句使用int的包装类Integer，第三句使用了jdk5自动装箱的特性（简化了包装类的使用，减少编码量，但是底层的语义没有改变，对运行时也没有任何影响）。
i1直接持有一个整数的值，但是i2持有的是一个Integer对象的引用！
说到自动装箱有一点需要注意：
``` java
Integer i1 = null;
int i2 = i1;
```
基于自动装箱的特性，上面的代码并不会有编译错误，但是运行的时候你会发现会抛出`NullPointerException`，因为i1指向的是null，在自动拆箱的时候就会出现错误。

<!--more-->

### 比较相等性
对于基本类型可以直接使用`==`操作符，但是对于对象类型，一般需要调用`equals`方法(可以这么理解，对于对象类型来说`==`比较的是内存地址)，而基本类型是没有`equals`方法的。

### 默认值
基本类型和对象类型的默认值也是有区别的，以int和Integer为例，int默认值是0，但是Integer的默认值则是null。

### 内存的使用
 基本类型的内存占用都是固定的：
 >byte: 8bits  
 short: 16bits  
 int: 32bits  
 long: 64bits  
 float: 32bits  
 double: 64bits  
 char: 16bits unicode  
 boolean: 8bits

但是对象类型引用占用的内存字节数取决于jvm。对于64位jvm，一个引用占用64bits。
举个例子，64位jvm中：
一个int要占用4字节；一个Integer要占用20个字节——Integer对象的引用占8个字节+对象中int的值占4个字节+对象对Integer对象的引用占8个字节。除此之外，jvm需要额外的内存来支持对象的gc，但是对于基本类型则不需要。

### 计算性能
``` java
public static double[][] multiply(double[][] a, double[][] b){
    int nRows = a.length;
    int nCols = b[0].length;

    double[][] result = new double[nRows][nCols];

    for (int rowNum = 0;  rowNum < nRows;  ++rowNum){
        for (int colNum = 0;  colNum < nCols;  ++colNum){
            double sum = 0.0;

            for (int i = 0;  i < a[0].length;  ++i)
                sum += a[rowNum][i]*b[i][colNum];

            result[rowNum][colNum] = sum;
        }
    }

    return result;
}
```
分别使用double和Double计算1000*1000的矩阵，运行的结果差别很大：
```
double:4104ms.
Double:28485ms.
```

## 结论
基本数据类型对于数值计算来说在内存和性能上都有很大优势。

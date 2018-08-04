---
title: java范型
date: 2017-11-13 14:57:28
tags: [java, generic type, ]
categories: JAVA
---

## self-bounded type
形如
```java
public BasicHolder <T> {   
  T element;  
  void  set(T arg){element = arg; }  
  T get(){  return  element; }  
  void  f(){  
    System.out.println(element.getClass().getSimpleName());  
  }  
}

class  Subtype  extends  BasicHolder <Subtype> {}  
  
public class CRGWithBasicHolder {   
  public static void  main(String [] args){    
    Subtype st1 =  new  Subtype()，st2 =  new  Subtype();  
    st1.set(ST2);  
    Suntype st3 = st1.get();  
    st1.f();  
  }  
} 
/* Output: 
Subtype 
*/
```
在结果类中将使用确切的类型而不是基类型。所以在Subtype中，set()的参数和get()的返回类型都是子类型。
<!--more-->

```java
class SelfBounded<T extends SelfBounded<T>> {  
  T element;  
  SelfBounded<T> set(T arg) {  
    element = arg;  
    return this;  
  }  
  T get() { return element; }  
}
```
自我约束需要额外的步骤来强制泛型被用作自己的约束参数

自我约束所要求的就是在这种继承关系中使用类：`class A extends SelfBounded<A> {} `；

` class D  {}  class E extends SelfBounded<D> {}`这种方式的定义是错误的，因为D不是一个self-bounded类

自我约束仅限于保持继承关系：如果将SelfBounded类换成`class NotSelfBounded<T>{}`，则类型E的定义就可以通过。

```java
//: generics/SelfBoundingAndCovariantArguments.java  
  
interface SelfBoundSetter<T extends SelfBoundSetter<T>> {  
  void set(T arg);  
}  
  
interface Setter extends SelfBoundSetter<Setter> {}  
  
public class SelfBoundingAndCovariantArguments {  
  void testA(Setter s1, Setter s2, SelfBoundSetter sbs) {  
    s1.set(s2);  
    // s1.set(sbs); // Error:  
    // set(Setter) in SelfBoundSetter<Setter>  
    // cannot be applied to (SelfBoundSetter)  
  }  
}
```
编译器无法识别将基类型（即sbs）作为set()的参数传递的尝试，因为没有该签名的方法。实际上，set方法可接受的参数已经被重写了，不能再接受SelfBoundSetter类型的参数。但是，在非泛型代码中，参数类型不能随子类型而变化，故在非泛型代码中，上述s1.set(sbs)是可以通过的，调用的将是父类的set方法，即方法重载，而非方法重写。
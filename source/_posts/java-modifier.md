title: "Java中的private/protected/public/区别"
date: 2015-11-25 21:27:09
categories: java
tags: [java]
toc: true
description: 受不了老是忘记public、protected、public的区别，必须写一篇记住他们

---

>Java中的private、protected、public、default（friendly）区别

###0x00 定义
1. **public**: 具有最大的访问权限，可以访问任何一个在CLASSPATH下的类、接口、异常等。它往往用于对外的情况，也就是对象或类对外的一种接口的形式
2. **protected**: 主要作用用来保护子类，访问仅限于包含类或从包含类派生的类型,只有包含该成员的类以及继承的类可以存取。
3. **default(friendly)**: 它是针对本包访问而设计的，任何处于本包下的类、接口、异常等，都可以相互访问，即使是父类没有用protected修饰的成员也可以
4. **private**: 访问权限仅限于类的内部，是一种封装的体现，例如，大多数的成员变量都是修饰符为private的，它们不希望被其他任何外部的类访问


###0x01 作用域  

|  修饰符    | 类内部  | 包内部  | 子类    | 外部类  |
| --------- |:------:|:------:|:------:|:------:|
| public    | √      | √      | √      | √      |
| protected | √      | √      | √      | X      |
| default   | √      | √      | X      | X      |
| private   | √      | X      | X      | X      |
 

###0x02 区别
（1）public：可以被所有其他类所访问。   
（2）private：只能被自己访问和修改。   
（3）protected：自身，子类及同一个包中类可以访问。   
（4）default（默认）：同一包中的类可以访问，声明时没有加修饰符，认为是friendly。

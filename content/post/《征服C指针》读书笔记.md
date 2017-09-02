---
title: "《征服C指针》读书笔记"
date: 2017-08-21T01:35:21+08:00
draft: false
---

### 用英语来解读C的声明

1. 首先着眼于识别符（变量名或者函数名）。
2. 从距离识别符最近的地方开始，依照优先顺序解释派生类型（指针，数组，函数）。优先顺序说明如下：
   1. 用于整理声明内容的括弧
   2. 用于表示数组的[]，用于表示函数的()
   3. 用于表示指针的*

3. 解释完成派生型，使用“of”或“to”或“returning”将它们连接起来。
4. C里面不存在多维数组。

<!--more-->


| C                    | English                                  | 中文                             |
| -------------------- | ---------------------------------------- | ------------------------------ |
| int *hoge[10];       | hoge is array of pointer to int          | hoge是指向int的指针的数组               |
| int (*hoge)[3];      | hoge is pointer to array of int          | hoge是指向int的数组的指针               |
| int func(int a);     | func is function(argument is int a) returning int | func是返回int的函数(参数是int a)        |
| int (*func)(int a)；  | func is pointer to function(argument is int a) returning int | func_p是指向返回int的函数(参数为int a)的指针 |
| int hoge``[10][3]``; | hoge is array(10) of array(3) of int     | hoge是int数组(元素数10)的数组(元素数3)     |

### 解读const修饰符

const代表变量只读。在指针作为参数的时候，const常用于将指针指向的对象设定为只读。

* const修饰解读声明时const外一层修饰符。
* 若类型指定符左边存在const，则修饰该类型指定符。

```c
char const *src;//src is a pointer to read-only char
const char *src;//src is a pointer to read-only char
```

### 解读数组

```c
int hoge[10];//hoge == &hoge[0]
```

**由\*声明的指针是指针变量，它可以随意修改指向的方向**，包括用new建立一片空间然后让指针指向它；然而用[]声明的指针是指针常量（然而**它指向的量却可以是一个变量**），你不能修改指针常量，但是**你可以借助这个指针常量去修改它所指向的指针变量**。

#### 数组为`sizeof`运算符的操作数

通过“`sizeof`表达式”的方式使用`sizeof`运算符的情况下，如果操作数是“表达式”，数组会被当成指针，这时对数组使用`sizeof`，返回的是**数组全体的尺寸而不是指针长度**。

#### 数组为`&`运算符的操作数

通过对数组使用`&`，可以返回指向整体数组的指针。

### 关于运算符

运算符和声明不可混淆，声明左侧有返回类型。

**两侧表达式必须指针层级相等才可赋值。**

* ``*``在表达式中，是**解引用**运算符。返回指针所指向的对象或函数。表达式指针层级-1。

* ``[]``在表达式中，是**下标**运算符，是一种语法糖，所以和``*``的指针层级一样-1。

  ```c
  p[i];//等同于
  *(p + i);
  ```

* ``->``在表达式中，是通过指针访问结构体成员的语法糖。

  * ``->``左侧是指针
  * ``.``左侧是结构体实体

  ```c
  p->hoge;//等同于
  (*p).hoge;
  ```

* ``&``在表达式中，是**地址**运算符，返回该值的指针。表达式指针层级+1。


### 参考链接

* [《征服C指针》Web版](http://avnpc.com/pages/c-pointer)
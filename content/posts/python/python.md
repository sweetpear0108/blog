---
title: 'python入门'
date: 2024-08-08T00:00:00+08:00
# weight: 1
# aliases: ["/alias"]
tags: ["python"]
author: "sweetpear"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: ""
summary: "python基础语法，包括数据类型、函数、类等"
---
# python入门
## 动态语言
在运行时确定变量类型
* 优势：灵活、简洁、更快的开发周期
* 劣势：缺乏编译期类型检查容易出错

### 特性
动态添加属性和方法、动态类型绑定、反射和元编程

### 底层实现
有一个基类承接所有对象，从而实现动态类型

## 数据类型
### 基本类型
字符串、整数、浮点数、布尔值(True, False)、空值(None)

一些运算符：
* `//` 与 `/` ：前者保留商的整数部分，后者是float类型的商
* 逻辑运算符：`and` `or` `not`

### List
声明方式：`list = [1, '2', xxx]`

生成式声明：`list = [expr for iter_var in iterable if cond_expr]`

List中可以存各种类型的元素

#### 相关操作
List截取与go中切片语法类似 `[:]`

追加`.append` 

指定位置插入`.insert` 

追加另一个list`.extend` 或 `list1 + list2`

三种移除方式 `.pop`、`.remove`、`del list[idx]`

是否包含 `x in list`

### Tuple
声明方式： `tuple = (1, '2', xxx)` 单元素声明： `tuple = (1,)`

单元素声明要带逗号，否则会被认为是该单元素的数据类型

与List不同的是Tuple是不可变的，但也不是完全不可变：

不可变指：Tuple的每个元素，指向永远不变。但是可以改变Tuple中元素的元素（例如元素List中再追加、改变元素的属性等）

其他相关操作除了没有修改之外与List相同

### Dict
声明方式： `dict = {1 : xx, 'yy' : 2}`

key必须是不可变的：数字，字符串或元组

#### 相关操作
`.clear` 清空

`.items` 以列表返回可遍历的(键, 值) 元组数组

### Set
声明方式： `s = set([1, '2', xxx])` 使用List作为输入

#### 相关操作
添加 `.add`

移除一个元素 `.remove`

并集 `|` 交集 `&` 差集 `-`

## 函数参数
### 默认参数
```python
def print_user_info( name , age , sex = '男' ):
```

* 默认参数只能放在末尾
* 默认参数只能是不可变对象（字符串、数字、元组、布尔、None）

如果用可变对象做默认参数，会出现默认值被改变

### 关键字参数
```python
print_user_info(name='甜味梨', sex='女', age=18)
```

在调用时指定参数关键字进行赋值

如果想要对于某些形参**强制指定**需要使用关键字参数，可以在定义时通过 `*` 分隔形参，*后的形参为需要指定关键字的

```python
def print_user_info(name, *, age, sex='男'):
```

### 不定长参数（位置参数）
* 通过在形参前加 `*` 来声明不定长参数，实际上是用一个**元组**接收
* 如果想要在调用时也能使用**关键字参数**，需要在形参前加 `**`，这样会将接收到的参数存放到**字典**中

例子：
```python
def print_user_info(name, age, *args, sex='男', **kwargs):
    print('昵称：{}'.format(name) , end = ' ')
    print('年龄：{}'.format(age) , end = ' ')
    print('性别：{}'.format(sex) ,end = ' ' )
    print('args:{}'.format(args),end = ' ')
    print('kwargs:{}'.format(kwargs))

if __name__ == '__main__' :
    print_user_info('甜味梨', 18, 'arg1', 'arg2', hobby=('打篮球','打羽毛球','跑步'), height=179)

# output:
# 昵称：甜味梨 年龄：18 性别：男 args:('arg1', 'arg2') kwargs:{'hobby': ('打篮球', '打羽毛球', '跑步'), 'height': 179}
```

## lambda 函数
```python
lambda [arg1 [,arg2,.....argn]]:expression
```

lambda函数是一个**表达式**而非代码块，所以拥有自己的命名空间，且不能访问自有参数列表之外或全局命名空间里的参数。
换句话说，lambda用到的参数只能通过形参传入，不能用外部的变量，否则会出错。

坑：
```python
if __name__ == '__main__' :
    num2 = 100
    sum1 = lambda num1 : num1 + num2

    num2 = 10000
    sum2 = lambda num1 : num1 + num2

    num2 = 1

    print(sum1(1))
    print(sum2(1))
    # output: 2 2
```

原因：num2变量在**运行时**绑定值，而不是定义时就绑定

## 迭代器和生成器
### 迭代器
使用 `iter()` 生成一个迭代器，通过 `next()` 取下一个值，或者在for循环中使用 `in` 依次取值。

只有实现了 `__iter__` 方法的对象才能使用生成器，否则需要自己写。

反向迭代需要实现 `__reversed__` 方法。

### 生成器
在函数声明中使用 `yield` 返回一个生成器对象而不是像return一样返回结果。

`yield`返回后当前的这个生成器函数会记录当前状态，在下次调用时从暂停处继续。

优势：惰性求值的特点可以节约内存，适合处理大型数据集或创建复杂的数据流。

## 面向对象
### 实例方法和类方法
类中函数上加 `@classmethod` 注解表示这个方法是类方法，绑定在类上，调用时不需要实例化直接`类名.方法`调用。

### 实例属性和类属性
实例属性在 `__init__` 函数中声明，类属性在类中声明。

ps: 一个类只能有一个构造器，但是可以通过类方法变相实现多个构造器。

### 访问控制
在属性或方法前加两个下划线 `__` ，表明私有性，不能在类外访问且不能被继承。

* 私有性实现的原理是通过名称改写，如 `__var1` 私有属性会被改写为 `_类名称__var1`，因此按照定义的名称访问不到，
但是通过改写后的名称是可以访问的。所以所谓“私有”只是方便开发所做的优化，并不是真的私有。

### 动态修改
基于python动态语言的特性，在类外可以通过声明一个新方法并 `className.oldMethod = newMethod` 对旧方法进行重写，**修改会应用到所有已经创建的实例中**。

同理对于类属性的修改和增添也可以在类外动态修改。

实例属性可以动态增添修改，但不能对实例方法进行重写。

## 模块与包
### 导入
`import moduleName` 直接导入模块，调用时 `moduleName.xxx`

`from moduleName import xxx` 会将该模块中的指定类或方法或属性导入当前模块，调用时就不用加 `moduleName` 而是直接使用导入模块中的命名。

### 主模块
如果这个模块没有被引用过，那么就是主模块，否则为非主模块。有一个系统变量`__name__`就是用来记录模块属性的。

### 包
为了避免模块名冲突，python又引入了按目录来组织模块的方法，称为包（Package）。

对于拥有 `__init__.py` 文件的目录，python会将其视为一个包。

## 闭包
闭包是一个函数值，它引用了其函数体之外的变量。通过某种数据结构存储一个函数和一个关联的上下文环境，实现闭包的特性。

例子：
```python
time = 0

def study_time(time):
    def insert_time(min):
        nonlocal  time 
        time = time + min
        return time

    return insert_time


f = study_time(time)
print(f(2)) # 2
print(time) # 0
print(f(10)) # 12
print(time) # 0
```

python中在函数的`__closure__`属性中保存上下文环境，从而使得每个闭包都能拥有独立的函数外部变量。

ps:
* `nonlocal` 在函数内部声明一个外部函数的变量
* `global` 在函数内部声明一个全局变量

## 装饰器
装饰器利用闭包的特性实现

作用：不修改原有函数代码的基础上，动态地增加或修改函数的功能。解耦通用的功能。

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("Something is happening before the function is called.")
        result = func(*args, **kwargs)
        print("Something is happening after the function is called.")
        return result
    return wrapper

@my_decorator
def say_hello(name):
    print(f"Hello, {name}!")

say_hello("Alice")

# output:
# Something is happening before the function is called.
# Hello, Alice!
# Something is happening after the function is called.

```
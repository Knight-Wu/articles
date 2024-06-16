* 多线程

java 通过共享内存加锁来实现多线程, golang 通过线程间通信; 
go 的routine 更轻量级，内存占用少

* 错误处理

go 独有的panic 是一种严重错误提前发现的思想, 进程直接挂掉, 而不是java 的异常有可能会忽略

* 接口, 多态

go 通过实现接口的方法就隐式实现了这个接口, 更加灵活和简便, 支持多继承, java 则是需要显式实现接口, 并且不能多继承.

* 编译和部署

go 通过编译时就直接转换为对应平台的二进制机器码, 省略了运行时翻译这一步更加高效, java 需要借助jvm 翻译成机器码

* 传参

go 有显式传值或者传引用, java 总是pass by value, 通过拷贝引用的方式 ?? 很奇怪

* java 是面向对象语言, golang 按照官网说法是也不是(yes and no)

> 官网:Yes and no. Although Go has types and methods and allows an object-oriented style of programming, there is no type hierarchy. The concept of “interface” in Go provides a different approach that we believe is easy to use and in some ways more general. There are also ways to embed types in other types to provide something analogous — but not identical — to subclassing. Moreover, methods in Go are more general than in C   or Java: they can be defined for any sort of data, even built-in types such as plain, “unboxed” integers. They are not restricted to structs (classes).
> 翻译: 是和不是。 虽然 Go 有类型和方法，并允许面向对象的编程风格，但没有类型层次结构。 Go 中的“接口”概念提供了一种不同的方法，我们认为这种方法易于使用，并且在某些方面更通用。 还有一些方法可以将类型嵌入到其他类型中，以提供类似于 但不完全相同 - 子类化的东西。 此外，Go 中的方法比 C 或 Java 中的方法更通用：它们可以为任何类型的数据定义，甚至是内置类型，例如普通的“未装箱”整数。 它们不限于结构（类）。

其实能实现类似OOP 的对象的东西, 但是是没有类型层次结构的, 有父类, 但是没有严格的父类的父类这种层次结构. 并且接口也是更加通用的方式. 

* 什么是OOP , 对比面向过程
OOP 把对象作为程序的基本单元, 一个对象包含了属性(数据)和操作属性(数据)的函数,  
面向对象的程序设计把计算机程序视为一组对象的集合，而每个对象都可以接收其他对象发过来的消息(调用其他对象的方法)，并处理这些消息，计算机程序的执行就是一系列消息在各个对象之间传递, 或者说是对象的交互.

而面向过程则是一组函数的顺序执行作为程序, 函数就是指接受参数, 输出返回值, 没有自身的属性; 通过把大函数拆分成子函数来降低程序复杂度.

所以，面向对象的设计思想是抽象出Class, 根据Class创建Instance。

面向对象的抽象程度又比函数要高，因为一个Class既包含数据，又包含操作数据的方法。

例如:
我们以一个例子来说明面向过程和面向对象在程序流程上的不同之处。

假设我们要处理学生的成绩表，为了表示一个学生的成绩，面向过程的程序可以用一个dict表示：

std1 = { 'name': 'Michael', 'score': 98 }
std2 = { 'name': 'Bob', 'score': 81 }
而处理学生成绩可以通过函数实现，比如打印学生的成绩：

def print_score(std):
    print('%s: %s' % (std['name'], std['score']))

如果采用面向对象的程序设计思想，我们首选思考的不是程序的执行流程，而是Student这种数据类型应该被视为一个对象，这个对象拥有name和score这两个属性（Property）。如果要打印一个学生的成绩，首先必须创建出这个学生对应的对象，然后，给对象发一个print_score消息，让对象自己把自己的数据打印出来。
```
class Student(object):

    def __init__(self, name, score):
        self.name = name
        self.score = score

    def print_score(self):
        print('%s: %s' % (self.name, self.score))
```
给对象发消息实际上就是调用对象对应的关联函数，我们称之为对象的方法（Method）。面向对象的程序写出来就像这样：
```
bart = Student('Bart Simpson', 59)
lisa = Student('Lisa Simpson', 87)
bart.print_score()
lisa.print_score()
```
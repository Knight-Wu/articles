## java 8
* lambda 表达式, 可以把函数作为参数传递, 代码简洁, 常用的函数式接口: Supplier无接受参数返回一个值, Consumer 接受一个参数无返回值, Function<T,R> 接受参数T 返回参数R
```

 //使用java匿名内部类
  Comparator<Integer> cpt = new Comparator<Integer>() {
      @Override
      public int compare(Integer o1, Integer o2) {
          return Integer.compare(o1,o2);
      }
  };

  TreeSet<Integer> set = new TreeSet<>(cpt);

  System.out.println("=========================");

  //使用JDK8 lambda表达式
  Comparator<Integer> cpt2 = (x,y) -> Integer.compare(x,y);
  TreeSet<Integer> set2 = new TreeSet<>(cpt2); 

```
* stream API 方便操作集合
```
   // 循环过滤元素                                       
    proList.stream()
           .fliter((p) -> "红色".equals(p.getColor()))
           .forEach(System.out::println);

    // map处理元素然后再循环遍历
    proList.stream()
           .map(Product::getName)
           .forEach(System.out::println);

   // map处理元素转换成一个List
   proList.stream()
           .map(Product::getName)
           .collect(Collectors.toList());

```


* Optional 方便处理空指针异常

```
 public String getDesc(Test test){
          return Optional.ofNullable(test)
                  .map(Test::getDesc).else("");
      } 
```

* 接口中可以包含default 默认方法和 static 静态方法

在 Java 8之前，接口可以有常量变量和抽象方法。
我们不能在接口中提供方法实现。如果我们要提供抽象方法和非抽象方法（方法与实现）的组合，那么我们就得使用抽象类。
在 Java 8 中，一个接口中能定义如下几种变量/方法：

常量
抽象方法(abstract 修饰)
默认方法(default 修饰)
静态方法(static)
## jdk 9
* 集合工厂方法

```
List<String> fruits = List.of("apple", "banana", "orange");
Map<Integer, String> numbers = Map.of(1, "one", 2,"two", 3, "three"); 
```

* 接口中可以定义私有方法
Java 9 不仅像 Java 8 一样支持接口默认方法，同时还支持私有方法。

在 Java 9 中，一个接口中能定义如下几种变量/方法：

常量
抽象方法
默认方法
静态方法
私有方法
私有静态方法

例如，如果需要两个默认方法可以通过私有方法来共享代码，但不将该私有方法暴露给它的实现类调用.

在接口中使用私有方法有四个规则：

接口中private方法不能是abstract抽象方法。因为abstract抽象方法是公开的用于给接口实现类实现的方法，所以不能是private。
接口中私有方法只能在接口内部的方法里面被调用。
接口中私有静态方法可以在其他静态和非静态接口方法中使用。
接口中私有非静态方法不能在私有静态方法内部使用。

## jdk 10

* var 变量不需要指定变量类型

Java中var是Java10版本新出的特性，用它来定义局部变量。
使用var 定义变量的语法： var 变量名 = 初始值；
如果代码：
var a = 20；
var a =8.9；
这样的代码会报错 显示int到double的转换；
Java是强类型语言，每个变量都有固定的变量类型。
var是什么：
var不是关键字，它相当于是一种动态类型；
var动态类型是编译器根据变量所赋的值来推断类型；
var 没有改变Java的本质，var只是一种简便的写法，
就是说在定义局部变量时，任意什么类型都可以用var定义变量的类型会根据所赋的值来判断。
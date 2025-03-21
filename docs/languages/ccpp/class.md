# 类

## 基本定义

- 对象成员的调用顺序按照声明的顺序，<font color='red'>**对象成员的析构顺序与之相反**</font>；
- A构造函数调用 -->A的父类构造函数调用 --> A中对象成员的构造函数调用；
- 类的静态成员（除常量）都需要在**类定义之外进行初始化**；

```c++
// 对常量数据成员b和引用变量c赋值（该两类成员一定要赋值）；
// 数据成员初始化顺序按在类中的声明次序；
A(): b(1), c(x)
```





### 复制（拷贝）构造函数

> 默认的复制构造函数和赋值运算符进行的都是"shallow copy"

```c++
// 1. const只是为了不能修改a的内容
// 2. 类成员变量，需要在拷贝函数的成员初始化表中指定拷贝构造函数，否则是默认的拷贝函数；
A::A(const A &a): b(a.b) {
    
}
```

复制构造函数在什么时候被调用

- 对象在**创建时**使用其他的对象初始化

  ```c++
  Person p(q);  //此时复制构造函数被用来创建实例p
  Person p = q; //此时复制构造函数被用来在定义实例p时初始化p
  
  // 如果对象已经存在，然后将另一个已存在的对象赋给它，调用的就是赋值运算符(重载)
  p = q
  ```

- 对象以值传递的方式从函数返回

- 对象作为函数的参数进行值传递时



### 赋值重载函数



### 友元函数

将这些友元声明在类里；

- 使某全局函数、其他类或其他类的某成员函数可以访问该类的私有和保护成员，称为友元函数、友元类、友元成员函数；

- 友元不具有传递性



## 继承

> 尽量不使用菱形继承。

- 派生类不继承基类的赋值操作；
- 虚函数：virtual限定符；纯虚函数，virtual void f() = 0; 包含纯虚函数为抽象类；
  - 纯虚函数没有实习，虚函数父类可以有实现；


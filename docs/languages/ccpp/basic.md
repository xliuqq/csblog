# C++

## 数组

C99 标准中新增了变长数组（variable-length array，简称VLA）

- <font color='red'>**允许使用变量指定数组的大小**</font>：栈上分配

```c
int n = a + b;
int arr[n];
```



## 指针

`shared_ptr`

`unique_ptr`



## \# 和 \##

\#  -- 转换， 完成代码到字符串的转换

```c++
#define CONVERT(name) #name
 
int main(int argc, char* argv[])
{
    printf("You and %s are friends.\n", CONVERT(James));
    return 0;
}
```

\## -- 连接， 完成代码的连接

 ```c++
#define CAT(batman, robin) batman ## robin
 
#define make_friend(index)  printf("You and %s are friends.\n", CAT(james, index));
 
int main(int argc, char* argv[])
{
    char* james001="fake James 001";
    char* james007="James Bond";
    char* james110="fake James 110";
 
    make_friend(001); // print the james001 variable
    make_friend(007);
    make_friend(110);
    return 0;
}
 ```

## \__attribute__

`__attribute__((constructor))` 与 `__attribute__((destructor)) `是 GCC 中用来修饰函数的

- constructor 可以使被修饰的函数在 **main() 执行前被调用**；
- destructor 可以使被修饰的函数在 **main() 执行结束或 exit() 调用结束后**被执行。

## const 修饰

**指针参数为 `const`**

- 不会修改通过该指针传递的数据

```c++
void foo(const int* ptr) {  
    // *ptr = 42; // 编译错误，不能修改通过const指针传递的数据  
}
void foo(const int& ref) {  
    // ref = 42; // 编译错误，不能修改通过const引用传递的数据  
}
```

**返回值类型为 `const`**

- 用于返回指向常量数据的指针或引用，以确保调用者不会意外地修改这些数据

```c++
const int* get_const_pointer() {  
    static int x = 42;  
    return &x;  
}  

const int& get_const_reference() {  
    static int y = 42;  
    return y;  
}
```

**成员函数后的 `const`**

- 函数**不会修改调用它的对象的状态**（即，该函数是一个常量成员函数）

```c++
class MyClass {  
public:  
    int value;  
 
    int getValue() const { // 这是一个常量成员函数  
        return value;  
    }  
 
    // 以下函数不能是常量成员函数，因为它试图修改成员变量  
    // void setValue(int v) const { // 编译错误  
    //     value = v;  
    // }  
 
    void setValue(int v) { // 非常量成员函数  
        value = v;  
    }  
};  
 
const MyClass obj;  
int val = obj.getValue(); // 正确，因为getValue是常量成员函数  
// obj.setValue(42); // 编译错误，因为obj是常量对象，不能调用非常量成员函数
```



## Std Buffer

libc will **line-buffer when stdout to screen** and **block-buffer when stdout to a file**, but no-buffer for stderr.



## extern C

> C++ 的关键字，用于引入 C 的头文件

- 被 extern "C" 限定的函数或变量是<font color='red'>**`extern` 类型**</font>的
  - 表明函数和全局变量作用范围（可见性）的关键字，可以被外部模块使用
- 被 extern "C" 修饰的变量和函数是<font color='red'>**按照 `C` 语言方式编译和连接**</font>的。
  - C中的函数`C++` 编译后在符号库中的名字与 `C` 语言的有所不同（因为C++支持函数重载）

示例：C 中的库，如<stdlib.h>，都在其定义的时候，使用了`extern C`

- 如果 C++ 中使用 C 的库，如 stdlib，应该引入 \<cstdlib> ，因为会将函数限定在 `std` 的名空间；

```c
#ifndef __BEGIN_DECLS
# ifdef  __cplusplus
#  define __BEGIN_DECLS  extern "C" {
#  define __END_DECLS    }
# else
#  define __BEGIN_DECLS
#  define __END_DECLS
# endif
#endif
```



- - 
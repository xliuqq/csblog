# FFI

FFI（Foreign Function Interface）允许以一种语言编写的代码调用另一种语言的代码



## [LibFFI](https://github.com/libffi/libffi)

`Libffi`是一个外部函数接口库，提供了最底层的、与架构相关的、完整的FFI

- 提供了一个C编程语言接口，用于**在运行时（而不是编译时）给定有关目标函数的信息来调用本地编译函数**。
- 可以生成一个指向可以**接受和解码在运行时定义的参数组合的函数的指针**。

`libffi`的作用就相当于编译器，它为多种调用规则提供了一系列高级语言编程接口，然后通过相应接口完成函数调用，底层会根据对应的规则，完成数据准备，生成相应的汇编指令代码。



## Java FFI

[jffi](https://github.com/jnr/jffi)  基于 libffi-3.4.4 

[jna](https://github.com/java-native-access/jna/tree/master/native/libffi) 基于 libffi-3.4.4 



## Golang FFI(Cgo)

> [cgo doc](https://go.dev/src/cmd/cgo/doc.go)



## Python FFI

### [CFFI](https://github.com/python-cffi/cffi)

A Foreign Function Interface package for calling <font color='red'>**C**</font> libraries from Python. **基于 libffi 实现**

### Ctypes

Python内置，基于 **基于 libffi 实现**。

### [Pybind11](https://pybind11.readthedocs.io/en/stable/)

Seamless operability **between** <font color='red'>**C++11**</font> and Python.
# 模板方法模式

> 定义一套流程模板，根据需要实现模板中的操作。

## 意图

定义一个操作中的算法的骨架，而将一些步骤延迟到子类中。模板方法使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。



## 适用性

适用于以下场景。

1. 算法的整体步骤很固定，但其中个别部分易变时，这时候可以使用模板方法模式，将容易变的部分抽象出来，供子类实现。
2. 当多个子类存在公共的行为时，可以将其提取出来并集中到一个公共父类中以避免代码重复。首先，要识别现有代码中的不同之处，并且将不同之处分离为新的操作。最后，用一个调用这些新的操作的模板方法来替换这些不同的代码。
3. 当需要控制子类的扩展时，模板方法只在特定点调用钩子操作，这样就只允许在这些点进行扩展。



## 类

<img src=".pics/template/template_method.png" alt="模板方法模式的结构图" style="zoom: 67%;" />

## 优劣势

**优点：** 

- **封装不变部分，扩展可变部分**；
- 提取公共代码，便于维护；
- 行为由父类控制，子类实现。

**缺点：**

- 父类中的抽象方法由子类实现，子类执行的结果会影响父类的结果，这导致一种反向的控制结构；
- 继承关系自身的缺点，如果父类添加新的抽象方法，则所有子类都要修改；
# 规则引擎

**把业务规则从代码里剥离出来，规则可配置、可变更，不用改 Java 代码、不用重新发布服务**。

1. **普通规则 / 编排引擎（LiteFlow、Aviator 封装）**：按顺序遍历规则，命中就执行，没有自动推理；规则冲突要业务自己控制顺序。
2. **BRMS 推理引擎 (Drools、GoRules、NRules，Rete/PHREAK)**： 自动模式匹配，**不需要写遍历循环**；多条规则互相触发、正向推理；自动处理规则冲突、优先级、死循环检测。

## [Drools](https://kie.apache.org/drools/)

 的开源规则引擎，将规则与业务代码解耦。规则以脚本的形式存储在一个文件中，使规则的变化不需要修改代码，重新启动机器即可在线上环境中生效。

### 语法

`drl`文件

- package：包名，只限于逻辑上的管理，若自定义的查询或函数位于同一包名，不管物理位置如何，都可以直接调用。
- import：规则引用问题，导入类或静态方法。
- global：全局变量，使用时需要单独定义变量类型
- function：自定义函数，可以理解为Java静态方法的一种变形，与JavaScript函数定义相似。
- queried：查询。
- rule  end：规则内容中的规则体，是进行业务规则判断、处理业务结果的部分。

#### 规则体

```drl
rule "name"
attributes
    when
        LHS(条件部分)
    Then
        RHS（结果部分)
end
```

- rule：规则开始，参数是规则的唯一名称
- attributes：规则属性，是rule与when之间的参数，为可选项
- when：规则条件部分，默认为true
- then：规则结果部分
- end：当前规则结束

#### Fact对象

Fact 是指在Drools 规则应用当中，将一个普通的JavaBean 插入到规则的WorkingMemory当中后的对象。
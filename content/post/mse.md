---
title: "面向对象技术"
date: 2019-01-05T23:52:37+08:00
draft: false
---
[TOC]
## 面向对象

类型：人们出于不同的目的,按照客观事物的特性、行为或用途，把它们分成不同的种类,每一种类就是一个类型。

基本类型：固有的数据类型

值集描述：枚举法，限定法，构造法。

操作集描述：语法+语义

自定义类型：基本类型和类型构造符（struct enum type）

定义&构造：构造需要定义作用在值集上的所有操作

类型系统：一组相互关联的类型，它们的**实例**协同工作，实现一个共同的目标。

类型化：语言的内在属性。静态类型化语言（它的每个表达式的类型在静态程序分析(例如编译)时是确定的。在所有的程序构造中,每个类型在编译时是已知的。）；强类型化语言（它的每个表达式保证是类型一致的。即允许有些表达式的类型本身是静态未知的,但可以用引入运行时的类型检验(或推理)来做到。）

类型系统作用：抽象作用（分类地表示客观世界事物）、限制作用（防止逻辑单元间交互作用的不一致）。

声明：定义声明和非定义声明（先承认再核实）。

数组：同质数组，异质数组。记录-》struct

指针：其值是某块内存的地址。与某个类型T相关。值集是合法地址构成的集合。用途：1同一存储空间分时存储多个类型的值。2一组同类型数据不同操作，但是要求原始存储位置不动。3不需要知道数组元素下标的情况下使用指针访问。

```c++
1 常量与非常量指针
const int c1 = 1;//c1 : 其值是常量，必须在定义时给定值 
int const c2 = 2;//c2 : 同c1
const int * p1; //p1 : 指向“整型常量”的指针
int const * p2; //p2 : 同p1

2 指针常量
char a = ‘A’;
char * const cp =&a; // cp: const pointer to a char
char s[ ] = “张三”; // s: char array
const char *const cpc =s;
// cpc: const pointer to a const char
```



引用：另一个已存在对象的别名，二者代表同一块内存空间。**已存在**，**不能再引用其他对象**。1 作为函数形参避免实参被复制 2 作为函数返回值 避免返回值被复制（返回调用点需要变量**仍然存活**）

指针&引用：

都可以通过一个量访问另一个。但语法不同。引用是**直接**形式，指针是**间接**形式。

引用类型在定义时指定被引用对象之后，不能再引用其他变量。

作为函数形参时，引用类型的形参是相应实参的**别名**。而指针形参是另外一个量的**地址**。

结构体&联合体：

都有若干成员，定义相似，仅关键字不同。操作集相同。

结构各成员储存空间不同；联合类型共用存储空间。>= & MAX

#### 数据抽象

过程抽象（功能抽象）：只说明这个过程具有某种处理能力、使用规范、但不提供其细节。

数据抽象：设计抽象数据类型。

两个基本概念：信息隐蔽，封装。**封装蕴含了信息隐蔽**，封装使得**外界只能通过规定的抽象操作来操纵数据**对象，这意味着这些数据对象被隐藏了。**信息隐蔽并不必然蕴含封装。**

本质特征：使用与实现分离。把大的系统分解成许多个小的部分，每个部分有一个按所处理的数据而设计的接口。

基于封装的模块化机制。

抽象数据类型是数据抽象的一种类型化实现机制。

![1546780749783](/tmp/1546780749783.png)

### 多态

![1546914344998](/tmp/1546914344998.png)

参数多态：template，template\<class T, int MAX\>

包含多态：属性和操作继承

过载多态：函数和操作符重载

强制多态：强制类型转换 显示/隐式

### 面向对象

系统由对象组成，程序运行->系统演变，经历一连串状态变化。

对象具有**状态保持**能力和**自主计算**能力。

![1546915038742](/tmp/1546915038742.png)

类：数据成员，成员函数。访问控制（实现封装，出错定位，易读）。构造函数，缺省构造函数（无形参、有形参但均有缺省值）。析构函数～。

对象：消息传递。主动的数据。每个对象独立的数据存储空间，同一个类的所有对象共享操作代码。为对象分配连续的存储空间。支持**类属性**（同一个类共享数据，静态数据成员）。

- 消息传递：自身引用
- 继承：共性和差异。实例化，constructor 自上而下，destructor 自下而上。

![1546916693636](/tmp/1546916693636.png)

- 重置：修改父类已有构造。虚函数允许子类重置（默认关键词缺省）。
  - 核心：动态绑定机制：**虚函数跳转表**（若干个虚函数体入口），及其**指针**。
  - 对于含有至少一个虚函数的每一个类，为之生成vtbl，实例存储vptr。
  - 重置时保持接口一致。（否则成了名字隐藏）
- 多态
  - 友元friend：类的所有成员对它公开。

## 信息系统和基本特征

系统：一组 相互关联 部件，协同工作；递归

-  6个成分：边界、输入、处理、输出、控制、反馈

- ![1546703736087](/tmp/1546703736087.png)

- 由人使用，由人制造

- 三个额外成分：人、过程、数据

- **人**以某种方式和系统中成分交互

  与界面通过 过程 形式来描述

  与系统交流 数据 相关

- 自动进行 需要重复=》自动信息系统=》简称 **信息系统**

- 分类：事务处理、管理信息、决策支持

![1546703993596](/tmp/1546703993596.png)

操作对象是**数据**，履行 的业务是**功能**，关于请求和能够观察到的效果是**行为**

- 信息系统分析和设计十分棘手：边界、结构定义模糊；问题动态演变；无唯一解；需要多领域知识技能；需要不断学习；分析过程本质上是一种认知活动



## 传统开发方法的特点和存在的问题

- 开发方法：系统分析和设计中采用的方式、策略、方向、活动。

- 典型方法：功能分解、结构化、信息建模、**面向对象**。

  

  #### 功能分解

  将系统看成若干功能集合，每个功能可划分为加工（子功能），一个加工分解为若干加工步骤。

- 功能分解法 = 功能 + 子功能 + 功能接口

- 层次状结构图

- 很好的对应性

  ![1546704504922](/tmp/1546704504922.png)

- 弱点：需求不一定能直接、客观地映射到功能上；功能易变；局部修改会影响其他部分；使用小型系统

#### 结构化方法

数据流图，着眼于数据加工。

结构化方法 = 数据流 + 数据处理(加工)+
数据存储 + 端点 + 处理说明 + 数据字典

- 端点：数据源 / 数据池

- 偏重数据在系统处理过程中的改变

  ![1546704715104](/tmp/1546704715104.png)

- 弱点：需求不一定以数据流为主干；表现不了数据的结构和联系；数据流分解是指数级，**数据字典爆炸**

#### 信息建模方法

E-R分析实体及其间的联系，从数据角度对问题域建模。

信息建模法= 实体 + 属性 + 关系

- 实体：问题域中实物的数据表示以及一些实例
- 关系：实体之间在数据方面的关系

强调数据的机构特征，数据之间的联系。适用于数据密集，结构复杂，数据加工相对简单的系统。

![1546705003193](/tmp/1546705003193.png)

弱点：表示不了对操作的需求；没有把实体操作封装到实体中，只有属性没有操作；父子类型只有属性继承，没有操作继承；表示模型和设计有较大区别



传统方法共同问题：功能和数据模型无法做好统一；分析到设计转换困难；缺少行为建模支持。

### 多视图特征

视图的种类要考虑:
信息系统具有的特征(数据、功能、行为)。

软件开发过程的特征(分阶段、分层次、易于理解和交流、需求的可跟踪性等)。

UML采用的是 “4+1” 视图。

![1546705365750](/tmp/1546705365750.png)

![1546705411064](/tmp/1546705411064.png)

使用uml定义uml![1546705441822](/tmp/1546705441822.png)

UML 4+1 视图

- 动态特征图

用例视图(use case view):用例图(use case diagram);协同图(collaboration diagram);序列图(sequence diagram) 。


- 设计视图(design view): 类图(class diagram),对象图;协同图;序列图;状态图(state-chart diagram);活动图(activity diagram);

- 实现视图(Implementation view):构件图(component diagram);协同图;序列图;活动图;

- 进程视图(process view):与设计视图相同,但侧重描述表示进程、线程的主动类,以及它们之间的交互行为。

-  部署视图(Deployment view):部署图(deployment diagram)



package 包 出现在多种图中 表示抽象集合

dependency 表示实体间的依赖

note 实体附加信息说明



- 用例图 use-case diagram

  展示**一组用例**、**参与者**、以及他们之间的**关系**

  use-case表示系统提供的一个功能单元，对一组**动作序列**的描述

  actor 交互 角色单元

  用例之间的关系：泛化（继承）、包含（显示地使用）、延伸拓展（描述可选行为）。

  ![泛化](/tmp/1546742249655.png)

  上边是parent 下边是child

  ![包含](/tmp/1546742177378.png)

  ![1546742390656](/tmp/1546742390656.png)

  左边是base use case 右边是拓展

  ![拓展延伸](/tmp/1546742183381.png)

  ![1546742329957](/tmp/1546742329957.png)

- 类图

  类是基本单元。包括类名、属性集和操作集。简略表示只有类名。

  public + private - protected # package ~

  关注点：analysis（捕捉系统涉及的词汇，**概念、职责**）、design（对词汇进行详细说明，**系统结构**）、programming（词汇实现，**源代码**）。

  **模板**：隐式绑定和显式绑定。

  ![1546740809692](/tmp/1546740809692.png)

  **泛化**：传递、非对称继承。可用于类、用例、包等模型元素。

  ![1546740958249](/tmp/1546740958249.png)

  **聚集/聚合**：表示整体和部分的关系

  ![1546740998243](/tmp/1546740998243.png)

  **关联**：两个实例之间具有关联关系。关系可用属性或者另一个**类**来表示。

  ![1546741072335](/tmp/1546741072335.png)

  ![1546741092294](/tmp/1546741092294.png)

- 状态图

  单个对象。可能序列状态以及对应的响应动作。

  单个对象的控制流。

  Event [ Guard ] / Action

  ![1546741237131](/tmp/1546741237131.png)



**正交**（并发子状态）每个正交区域的组合状态

![1546741379106](/tmp/1546741379106.png)

**非正交**（顺序子状态），状态组合，一次只能处于一个。

![1546741347477](/tmp/1546741347477.png)



**分叉fork** 外部->正交区域

**汇合join** 正交区域->外部

![1546741526178](/tmp/1546741526178.png)

- 活动图

  - 程序流程图。**过程** **状态机**。

  - 工作流 非类或者对象。

    ![1546741615957](/tmp/1546741615957.png)

    **汇合**和**分岔**应平衡,即离开一个分岔的流的数目=进入对应的汇合的流的数目

  ![泳道](/tmp/1546741730530.png)

- 序列图

  - 描述对象之间以时间为序的交互。
  - 一个序列图对应一个控制流建模。

  ![1546741843176](/tmp/1546741843176.png)

![1546741887756](/tmp/1546741887756.png)



可用矩形表示高层控制：可选执行(opt),条件执行(alt),并发执行(par),循环执行(loop)



## OOA & OOD

OOA：认识并描述问题域。从问题域中客观存在事物来认识系统。

OOD：采用与OOA一致的概念和表示方法

系统分析将产生“分析模型”,其中主要包括:

- Use-case 图: 说明参与者(actors)、主要的功能需求;
- 初始的类图:问题域的主要类、重要关系;
- 序列图:确定对象交互协议。(后两条统称“概念模型”)

系统设计通常产生“实现模型”,

- 考虑非功能需要的约束、平台约束;
- 类图:增加新类 or 精细已有类(组合/分离类,调整关系等);
- 增加 泛化、 interface、对interface的实现类;
- 构件/子系统模型、部署图等。

#### OOA

- 需求分析（用例）：功能性（must provide）、非功能性。

获取用例之基本步骤:

1. 确定系统边界;
2. 识别出系统的所有参与者;
3. 识别Use-Case: 系统所提供的业务活动;
4. 标识每个参与者所参与的业务活动(即用例);
5. 用结构化的自然语言描述每个用例的事件序列;
6. 对用例进行分析和必要的重组,采用包含、扩展、泛化等
    表示用例之间的关系。

#### 协同图

![1546928741087](/tmp/1546928741087.png)

### 单例模式

![1546928857418](/tmp/1546928857418.png)


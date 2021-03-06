---
title: "[译] PHP虚拟机"
date: 2018-07-01T12:32:13+08:00
draft: false
---
标签： PHP

---

本篇文章旨在提供一个对PHP7版本中Zend虚拟机的概述，不会做到面面俱到的详细叙述，但尽力包含大多数重要的部分，以及更精细的细节。

这篇文章描述的主要背景是PHP版本7.2（当前正在开发版本），但几乎同样适用于PHP7.0/7.1版本中。然而，PHP5.x系列版本的虚拟机之间差别比较显著，笔者不会去比较。

本文的大部分内容将在指令列表级别进行考虑，最后只有几节讨论VM的实际C语言实现。但是，我确实想提供一些组成VM前端的主要文件：

 - zend\_vm\_def.h：VM定义文件
 - zend\_vm\_execute.h：`生成的虚拟机文件`
 - zend\_vm\_gen.php：生成脚本
 - zend\_execute.c：大多数直接支持文件
 
## 操作码 （Opcodes）
首先我们来谈一下操作码，“操作码”是指完整的VM指令(包括操作数)，但也可以只指定“实际”操作代码--一个决定指令类型的小整数。从上下文来看，指令的意图应该是清晰的。在文件源码中，完整的指令通常被称为Opline。

一个独立的指令遵循如下zend_op结构体：
```
struct _zend_op {
    const void *handler;
    znode_op op1;
    znode_op op2;
    znode_op result;
    uint32_t extended_value;
    uint32_t lineno;
    zend_uchar opcode;
    zend_uchar op1_type;
    zend_uchar op2_type;
    zend_uchar result_type;
};
```
如上，操作码实质上是“三地址”指令格式。其中‘opcode’决定指令类型，还有两个输入的操作数‘op1’和‘op2’以及一个输出操作数‘result’。

并不是所有的指令都会用到所有的操作数。ADD指令（代表+操作符）会用到三个操作数。BOOL\_NOT指令（代表！操作符）只用到op1和result。ECHO指令只会用到op1操作符。有些指令甚至可以使用或者不使用操作符。例如，DO\_FCALL可以使用或者不使用result操作符，具体取决于是否使用函数调用的返回值。有些指令需要两个以上输入操作数，在这种情况下，只需要使用~~第二个~~虚拟指令/伪指令（OP_DATA）来携带额外的操作数。

除了这三个标准操作数之外，还有一个附加的数值extended_value 字段，可以用来保存附加的指令修饰符。例如，对于强制转换（CAST），它可能包含要强制转换的目标类型。

每个操作数都有对应的一个类型，分别存储在op1\_type, op2\_type和result\_type中。可能的类型有IS\_UNUSED, IS\_CONST, IS\_TMPVAR, IS\_VAR and IS\_CV。

后三种类型指定变量操作数(有三种不同类型的VM变量)，IS\_CONST表示常量操作数(5或“String”或偶数[1，2，3])，而IS\_UNUSED表示实际未使用的操作数，或作为32位数值(汇编术语中的“立即”)使用的操作数。例如，跳转指令将跳转目标存储在未使用的操作数中。

## 获取操作指令（Obtaining opcode dumps)
接下来，笔者将经常列出PHP代码生成的操作码序列。目前有三种方法可以将这些操作码转储：
```
# Opcache, since PHP 7.1
php -d opcache.opt_debug_level=0x10000 test.php

# phpdbg, since PHP 5.6
phpdbg -p* test.php

# vld, third-party extension
php -d vld.active=1 test.php
```

其中，Opcache提供了最高质量的输出。本文中使用的清单基于opcache转储，并进行了一些语法调整。魔术数字0x10000是“优化前”的缩写，因此我们看到PHP编译器生成的操作码。0x20000会给你优化的操作码。Opcache还可以生成更多的信息，例如0x40000将生成一个`CFG`，而0x200000将生成`类型和范围推断的SSA表单`。但这已经超越了我们想要的：普通的、原本的线性化操作码转储对我们来说就足够了。

## 变量类型（Variable types）
在研究虚拟机时，可能需要理解的最重要的一点在于它使用的三种不同的变量类型：
CV是“compiled variable”的缩写，而且指向一个“真正的”PHP变量。如果函数使用变量$a，就会有$a对应的CV。

CV可以有UNDEF类型，用来指向未定义变量。如果UNDEF CV在一个指令中用到，在大多数情况下会抛出“未定义变量（undefined variable）”提示。在函数入口处所有非参数CV会被初始化为UNDEF。

CV不会被指令`消耗`(consumed)，例如，指令ADD \$a,\$b 不会销毁存储在CV中的变量\$a和\$b。然而所有的CV都会在`事务(scope)`退出时一起销毁。这也暗示所有的CV在整个函数的域内都将‘live’--包含一个有效值。

TMPVARS和VARS是虚拟机的临时变量。它们通常被用作一些操作指令的结果操作数。例如$a = $b + $c + $d会输出如下类似指令序列：

```
T0 = ADD $b,$c
T1 = ADD T0,$d
ASSIGN $a,T1
```
TMP/VAR是在使用前定义的，因此不能保存UNDEF值。与CV不同，这些变量类型是由它们所使用的指令所消耗的。在上面的示例中，第二个ADD将破坏T0操作数的值，在此之后不能使用T0(除非事先写入)。类似地，ASSIGN将消耗T1的值，使T1无效。

因此，TMP/VAR通常寿命很短。在许多情况下，一个临时变量只存在一个指令的空间。在这个短暂的活动之外，临时变量就成了垃圾。

那么TMP和VAR有什么区别呢？不多。这种区别是从PHP5继承的，TMP是分配在VM栈中的，而VAR是分配在堆中的。在PHP7中，所有变量都是分配在栈中。因此，现在TMP和VAR的主要区别是只允许后者包含引用(REFERENCEs)(这允许我们在TMP上删除DEREF)。此外，VAR可能包含两种特殊值，即`类条目`(class entries)和间接值(INDIRECT values)。后者用于处理特殊的任务。

下方图表分析了主要的区别：
|-     |UNDEF|REF |INDIRECT|Consumed?|Named?|
|---   |---  |----|----    |----     |----  |
|CV    |yes  |yes |no      |no       |yes   |
|TMPVAR|no   |no  |no      |yes      |no    |
|VAR   |  no | yes|   yes  |yes      | no   |

## 指令数组（Op arrays）
所有PHP函数都表示为具有公共Zend_Function的 Header结构。这里的“函数”应该有一些广义的理解，包括从“真正的”函数到方法，到独立的“伪主(pseudo-main)”代码和“eval”代码。

用户级函数使用zend\_op\_Array结构。它有30多个成员，所以我现在先从一个简化版本开始：
```
struct _zend_op_array {
    /* Common zend_function header here */

    /* ... */
    uint32_t last;
    zend_op *opcodes;
    int last_var;
    uint32_t T;
    zend_string **vars;
    /* ... */
    int last_literal;
    zval *literals;
    /* ... */
};
```
最重要的部分当然是opcodes，它是一个包含操作指令的数组。‘last’是数组中操作指令的数量，注意这里的术语可能令人感到困惑，‘last’看起来像是最后一个操作指令的索引，但是这里实际上是操作数的数量（比最后一个操作数的索引大一）。这同样适用于op数组结构中的所有其他‘last_ *’值。

‘last_var’是CV的数量，‘T’是TMP和VAR的数量（在大多数情况下两者没有明显的区别）。‘vars’是CV的命名。
‘literals’是出现在代码中字面值的数组，这个数组是CONST操作数引用。根据**ABI①**，每个CONST操作数要么储存指向次文本表的引用，要么存储相对于其开始的偏移量。

关于op数组（op array）的内容不止于此。

## 栈框架布局（Stack frame layout）
除了一些全局指令EG(excutor globals)，所有的执行状态都存储在虚拟机栈中。虚拟机栈以大小为256KB的page分配，通过链表进行连接。

每一个函数调用时，将在VM栈上分配一个新的栈帧，布局如下：

```
+----------------------------------------+
| zend_execute_data                      |
+----------------------------------------+
| VAR[0]                =         ARG[1] | arguments
| ...                                    |
| VAR[num_args-1]       =         ARG[N] |
| VAR[num_args]         =   CV[num_args] | remaining CVs
| ...                                    |
| VAR[last_var-1]       = CV[last_var-1] |
| VAR[last_var]         =         TMP[0] | TMP/VARs
| ...                                    |
| VAR[last_var+T-1]     =         TMP[T] |
| ARG[N+1] (extra_args)                  | extra arguments
| ...                                    |
+----------------------------------------+
```
该框架以一个zend_execute_data结构开始，接着是一个变量槽（variable slots）的数组。这些变量槽都相同（simple zvals），但是用作不同的用途。第一个‘last\_var’槽中内容是CV，其中第一个num\_args存放函数参数。CV插槽之后是TMP/ VAR的‘T’插槽。最后，有时可以在帧的末尾存储“额外”参数。这些用于处理func_get_args()。

指令中的CV和TMP/VAR操作数被编码为相对于堆栈起始位置的偏移量，因此读取某个变量只是从execute_data位置读取的偏移量。

帧开始处的执行数据定义如下：
```C
struct _zend_execute_data {
    const zend_op       *opline;
    zend_execute_data   *call;
    zval                *return_value;
    zend_function       *func;
    zval                 This;             /* this + call_info + num_args    */
    zend_class_entry    *called_scope;
    zend_execute_data   *prev_execute_data;
    zend_array          *symbol_table;
    void               **run_time_cache;   /* cache op_array->run_time_cache */
    zval                *literals;         /* cache op_array->literals       */
};
```
最重要的是，这个结构包含opline（当前执行的指令）和func（它是当前执行的函数）。此外：

 - return\_value是一个指向将存储返回值的zval的指针。
 - ‘This’是$this对象，但也会编码一些未使用的zval空间中函数参数的数目和一些调用元数据标志。
 - called\_scope是static ::在PHP代码中引用的范围。
 - prev\_execute\_data指向前一个栈帧，在此函数完成运行后，执行将返回到该帧。
 - symbol\_table是一个通常未使用的符号表，用于某些疯狂的人实际使用变量变量或类似功能的情况。（$$a）
 - run\_time\_cache缓存op数组运行时缓存，以便在访问此结构时避免一个指针间接寻址（稍后讨论）。
 - 由于相同的原因，literals缓存op数组字面量表。
 
## 函数调用（Function calls）
笔者跳过了execute\_data结构中的一个字段--‘call'，因为它需要关于函数调用如何工作的更多上下文。所有调用都使用相同指令序列的变体。全局作用域中的var_dump（\$a，\$b）将编译为：
```
INIT_FCALL (2 args) "var_dump"
SEND_VAR $a
SEND_VAR $b
V0 = DO_ICALL   # or just DO_ICALL if retval unused
```
有八种不同类型的INIT指令，取决于它是什么类型的调用。INIT\_FCALL与调用相关，用来释放我们在编译时识别的函数。同样，根据参数和函数的类型，有十个不同的SEND操作码。只有数量较少的四个DO\_CALL操作码，其中ICALL用于调用内部函数。

虽然具体指令可能不同，但结构总是相同的：INIT，SEND，DO。调用序列必须解决的主要问题是嵌套函数调用，它们编译如下所示：
```
# var_dump(foo($a), bar($b))
INIT_FCALL (2 args) "var_dump"
    INIT_FCALL (1 arg) "foo"
    SEND_VAR $a
    V0 = DO_UCALL
SEND_VAR V0
    INIT_FCALL (1 arg) "bar"
    SEND_VAR $b
    V1 = DO_UCALL
SEND_VAR V1
V2 = DO_ICALL
```
INIT操作码在堆栈上压入一个调用帧，该帧包含了函数中所有变量的足够空间以及我们所知道的参数的数量（如果涉及参数解包，我们可能会以更多参数结束）。

指向新帧的指针存储到execute\_data-> call中，其中execute\_data是调用函数的帧。在下面，我们将把这些访问表示为EX（call）。值得注意的是，新帧的prev\_execute\_data被设置为旧的EX（call）值。例如，调用foo的INIT\_FCALL会将prev_execute_data设置为va\_dump的堆栈帧（而不是周边函数的栈帧）。因此，在这种情况下，prev\_execute\_data形成一个“未完成”调用的链表，而通常它会提供回溯链(双向链表)。

SEND操作码然后继续将参数推送到EX（call）的变量槽中。在这一点上，参数都是连续的，并且可以从为参数指定的部分溢出到其他CV或TMP中。这将在稍后修复。

最后DO\_FCALL执行实际的调用。EX（call）成为当前函数，prev\_execute\_data被重新链接到调用函数。除此之外，调用过程取决于它是什么类型的功能。内部函数只需要调用处理函数，而用户级函数需要完成栈帧的初始化。

这个初始化涉及修复参数栈。PHP允许传递比函数期望更多的参数（func\_get\_args依赖于这个功能）。但是，只有实际声明的参数才具有相应的CV。除此以外的任何参数都会写入为其他CV和TMP保留的内存。因此，这些参数将在TMP之后移动，最终将参数分成两个不连续的块。

为了清楚地说明，用户级函数调用不涉及虚拟机级别的递归。它们只涉及从一个execute\_data切换到另一个execute\_data，但虚拟机继续以线性循环运行。递归虚拟机调用仅在内部函数调用用户空间回调（例如通过array\_map）时才会发生。这就是为什么PHP中的无限递归通常会导致内存限制或OOM错误的原因，通过递归使用回调函数或魔术方法可能引发栈溢出。

## 参数传递（Argument sending）
PHP使用大量不同的参数发送操作码，这些操作码的区别可能会造成混淆。
SEND\_VAL和SEND\_VAR是最简单的变体，它处理在编译时已知是按值传递时进行的按值传递参数。SEND\_VAL用于CONST和TMP操作数，而SEND_VAR用于VAR和CV。

相反，SEND_REF用于在编译期间已知为引用传递的参数传递。由于只有变量可以通过引用发送，这个操作码只接受VAR和CV。

SEND\_VAL\_EX和SEND\_VAR\_EX是SEND\_VAL / SEND\_VAR的变体，用于无法静态确定参数是按值还是按引用传递的情况。这些操作码将根据arginfo检查参数的种类并相应地执行操作。在大多数情况下，不使用实际的arginfo结构，而是直接在函数结构中使用紧凑的位向量表示。

然后是SEND\_VAR\_NO\_REF\_EX。不要试图名称上了解这个指令。这个操作码用于传递一些不是真正的“变量”，但是会返回一个VAR到一个静态未知参数的东西。使用它的两个特定示例是将函数调用的结果作为参数传递，或者传递赋值的结果。**②**

这种情况需要一个独立的操作码，原因有两个：首先，如果尝试通过ref传递类似于赋值的内容它会生成熟悉的“只能通过引用传递变量”通知（如果使用SEND\_VAR\_EX，则会被默许）。其次，这个操作码处理的情况是，你可能想要将引用返回函数的结果传递给一个引用参数（它不应该抛出任何东西）。该操作码的SEND\_VAR\_NO\_REF变体（不带_EX）是一种特殊的变体，用于静态地知道引用是预期的情况（但我们不知道该变量是否为一个）。

SEND\_UNPACK和SEND\_ARRAY操作码分别处理参数解包和内联call\_user\_func\_array调用。它们都将数组中的元素推入参数栈，并在各种细节上有所不同（例如，解包支持Traversables，而call\_user\_func\_array则不支持）。如果使用unpacking/cufa，则可能需要将栈框架扩展超出其以前的大小（因为函数参数的实数在初始化时是未知的）。在大多数情况下，只需移动堆栈顶部指针就可以进行扩展。但是，如果这会跨越堆栈页面边界，则必须分配新页面，并且需要将整个调用帧（包括已经推入的参数）复制到新页面（我们无法处理穿过页面的调用帧边界）。

最后一个操作码是SEND\_USER，用于内联调用call\_user\_func并处理它的一些特性。

虽然我们尚未讨论不同的变量获取模式，但这似乎是介绍FUNC_ARG获取模式的好地方。考虑一个简单的调用func(\$a\[0]\[1]\[2])，在编译时我们不知道这个参数是通过值还是通过引用传递的。在这两种情况下，行为将会大不相同。如果传递是按值并且$a以前是空的，则可能必须生成一堆“未定义索引”通知。如果传递是通过引用的话，我们必须默默地初始化嵌套数组。

FUNC_ARG获取模式将通过检查当前EX(call)函数的arginfo来动态选择两种行为之一（R或W）。对于func(\$a\[0]\[1][2])的例子，操作码序列可能是这个样子：
```
INIT_FCALL_BY_NAME "func"
V0 = FETCH_DIM_FUNC_ARG (arg 1) $a, 0
V1 = FETCH_DIM_FUNC_ARG (arg 1) V0, 1
V2 = FETCH_DIM_FUNC_ARG (arg 1) V1, 2
SEND_VAR_EX V2
DO_FCALL
```

## 取参模式（Fetch modes）
PHP虚拟机有四类提取操作码：
```
FETCH_*             // $_GET, $$var
FETCH_DIM_*         // $arr[0]
FETCH_OBJ_*         // $obj->prop
FETCH_STATIC_PROP_* // A::$prop
```
这些完全符合人们期望它们做的事情，但要注意基本的FETCH_*变体仅用于访问变量变量和超全局变量：正常变量访问通过更快的CV机制来代替。

这些提取操作码每个都有六种变体：
```
_R
_RW
_W
_IS
_UNSET
_FUNC_ARG
```
我们已经知道，\_FUNC\_ARG根据函数参数是按值还是按引用来选择\_R和\_W。让我们尝试创建一些我们期望不同的获取类型出现的情况：
```
// $arr[0];
V2 = FETCH_DIM_R $arr int(0)
FREE V2

// $arr[0] = $val;
ASSIGN_DIM $arr int(0)
OP_DATA $val

// $arr[0] += 1;
ASSIGN_ADD (dim) $arr int(0)
OP_DATA int(1)

// isset($arr[0]);
T5 = ISSET_ISEMPTY_DIM_OBJ (isset) $arr int(0)
FREE T5

// unset($arr[0]);
UNSET_DIM $arr int(0)
```
不幸的是，这个产生的唯一真正的提取是FETCH\_DIM\_R：其他的东西都是通过特殊的操作码处理的。请注意，ASSIGN\_DIM和ASSIGN\_ADD都使用额外的OP\_DATA，因为它们需要两个以上的输入操作数。使用特殊操作码（如ASSIGN\_DIM）而不是像FETCH\_DIM\_W + ASSIGN之类的原因是（除了性能），这些操作可能被重载，例如，在ASSIGN\_DIM情况下，通过实现ArrayAccess::offsetSet()的对象实现。要实际生成不同的提取类型，我们需要增加嵌套的级别：
```
// $arr[0][1];
V2 = FETCH_DIM_R $arr int(0)
V3 = FETCH_DIM_R V2 int(1)
FREE V3

// $arr[0][1] = $val;
V4 = FETCH_DIM_W $arr int(0)
ASSIGN_DIM V4 int(1)
OP_DATA $val

// $arr[0][1] += 1;
V6 = FETCH_DIM_RW $arr int(0)
ASSIGN_ADD (dim) V6 int(1)
OP_DATA int(1)

// isset($arr[0][1]);
V8 = FETCH_DIM_IS $arr int(0)
T9 = ISSET_ISEMPTY_DIM_OBJ (isset) V8 int(1)
FREE T9

// unset($arr[0][1]);
V10 = FETCH_DIM_UNSET $arr int(0)
UNSET_DIM V10 int(1)
```
在这里我们看到，尽管最外层的访问使用专用的操作码，嵌套的索引将使用具有适当获取模式的FETCH进行处理。fetch模式的基本区别在于a）如果索引不存在，它们是否生成“未定义偏移量”通知，以及它们是否获取写入值：

|      | Notice? | Write?   |
|------|---------|----------|
|R     |  yes    |  no      |
|W     |  no     |  yes     |
|RW    |  yes    |  yes     |
|IS    |  no     |  no      |
|UNSET |  no     |  yes-ish |

UNSET的情况有点奇怪，因为它只能读取现有的偏移量以便写入，并且保留单独的未定义的偏移量。正常的写取操作会初始化未定义的偏移量。

## 写和内存安全（Writes and memory safety）

Write获取可能包含正常zval或指向另一个zval的INDIRECT指针的返回VAR。当然，在前一种情况下，应用于zval的任何更改都将不可见，因为该值只能通过虚拟机暂时访问。虽然PHP禁止表达[][0] = 42，但我们仍然需要处理这种情况 call()[0] = 42。取决于是call()按值还是按引用返回，此表达式可能会或可能不会有显著效果。

更典型的情况是当提取返回一个INDIRECT时，它包含一个指向正在被修改的存储位置的指针，例如哈希表数据数组中的某个位置。不幸的是，这样的指针是脆弱的东西，容易失效：任何并发写入数组可能会触发重新分配，留下一个悬挂指针。因此，防止在创建INDIRECT值的位置与消耗的位置之间执行用户代码至关重要。

考虑这个例子：
`$arr[a()][b()] = c();`
其中产生：

```
INIT_FCALL_BY_NAME (0 args) "a"
V1 = DO_FCALL_BY_NAME
INIT_FCALL_BY_NAME (0 args) "b"
V3 = DO_FCALL_BY_NAME
INIT_FCALL_BY_NAME (0 args) "c"
V5 = DO_FCALL_BY_NAME
V2 = FETCH_DIM_W $arr V1
ASSIGN_DIM V2 V3
OP_DATA V5
```

值得注意的是，这个序列首先执行从左到右的所有内嵌函数，然后才执行任何必要的写入读取（我们在此将FETCH\_DIM\_W称为“延迟opline”）。这确保了写访存和`消费指令`直接相邻。

考虑另一个例子：

`$arr[0] =& $arr[1];`

这里我们遇到了一些问题：两边的复制必须提取值才能写入。但是，如果我们抓取\$arr\[0] 写入然后写入\$arr\[1]，后者可能会使前者无效。这个问题解决如下：

```
V2 = FETCH_DIM_W $arr 1
V3 = MAKE_REF V2
V1 = FETCH_DIM_W $arr 0
ASSIGN_REF V1 V3
```

这里\$arr\[1]先取得写入，然后转换成使用MAKE\_REF的引用。MAKE\_REF的结果不再是间接的并且不会失效，因为这样的获取$arr[0]可以安全地执行。

## 异常处理（Exception handling）
异常是万恶之源。

~~异常是通过将异常写入EG(异常)来生成的，其中EG指的是执行者全局(Executor Globals)。C代码中抛出异常不涉及堆栈展开，相反，`执行退出`（abortion）将通过返回值失败代码或检查EG(异常)向上传播。只有当控制器重新进入虚拟机代码时，才会实际处理异常。~~

在某些情况下，几乎所有的VM指令都可能直接或间接导致异常。例如，如果使用自定义错误处理程序，则任何“未定义的变量”通知都可能导致异常。我们希望避免检查EG(exception)每个VM指令后设置。相反，使用一个小窍门：

当抛出一个异常时，当前执行数据的当前选择行被替换为虚拟HANDLE\_EXCEPTION opline（这显然不会修改op数组，它只是重定向一个指针）。备份发生异常的对象EG(opline\_before\_exception)。

这意味着当控制返回到主虚拟机调度循环时，将调用HANDLE\_EXCEPTION操作码。这个方案存在一个小问题：它要求 a)存储在执行数据中的opline实际上是当前执行的opline（否则opline\_before\_exception将会是错误的）并且 b)虚拟机使用来自执行数据的opline来继续执行（否则HANDLE\_EXCEPTION不会被调用）。

虽然这些要求可能听起来微不足道，但它们不是。原因是虚拟机可能正在处理与执行数据中存储的opline不同步opline变量。在PHP 7之前，这只发生在很少使用的GOTO和SWITCH虚拟机中，而在PHP 7中，这实际上是默认的操作模式：如果编译器支持它，则opline存储在全局寄存器中。

因此，在执行任何可能会抛出的操作之前，必须将本地对齐线写回执行数据（SAVE\_OPLINE操作）。同样，在任何可能的抛出操作之后，必须从执行数据填充本地对象（主要是CHECK_EXCEPTION操作）。

现在，这个机制是在引发异常之后导致HANDLE\_EXCEPTION操作码执行的原因。但它有什么作用？首先，它确定是否在try块内引发异常。为此，op数组包含一个try\_catch\_elements数组，用于跟踪try，catch和finally块的opline偏移量：
```
typedef struct _zend_try_catch_element {
	uint32_t try_op;
	uint32_t catch_op;  /* ketchup! */
	uint32_t finally_op;
	uint32_t finally_end;
} zend_try_catch_element;
```

现在我们会假装最后的块不存在，因为它们是一个完全不同的。假设我们确实在try块内，VM需要清理在抛出opline之前开始的所有未完成的操作，并且不会跨越try块的末尾。

这涉及释放当前在使用中的所有调用的栈帧和相关数据，以及释放临时变量。在大多数情况下，临时变量存在时间很短，甚至达到消费指令直接跟随着生成指令。然而，它可能发生在跨越多个指令的现场：

```
# (array)[] + throwing()
L0:   T0 = CAST (array) []
L1:   INIT_FCALL (0 args) "throwing"
L2:   V1 = DO_FCALL
L3:   T2 = ADD T0, V1
```

在这种情况下，T0变量在指令L1和L2中是活动的，因此如果函数调用抛出时需要销毁T0变量。一种特定的临时类型往往具有特别长的活动范围：循环变量。例如：
```
# foreach ($array as $value) throw $ex;
L0:   V0 = FE_RESET_R $array, ->L4
L1:   FE_FETCH_R V0, $value, ->L4
L2:   THROW $ex
L3:   JMP ->L1
L4:   FE_FREE V0
```

这里“循环变量”V0从L1到L3（通常总是跨越整个循环体）。实时范围使用以下结构存储在操作数组中：
```
typedef struct _zend_live_range {
    uint32_t var; /* low bits are used for variable type (ZEND_LIVE_* macros) */
    uint32_t start;
    uint32_t end;
} zend_live_range;
```

这里var是范围适用的（操作数编码的）变量，start是开始选择线偏移量（不包括生成指令），而end如果结束选择线偏移量（包括消耗指令）。当然，如果暂时没有立即消耗，则仅存储活动范围。

低位var用于存储变量的类型，可以是以下之一：

- ZEND\_LIVE\_TMPVAR：这是一个“正常”变量。它拥有一个普通的zval值。释放这个变量的行为就像一个免费的操作码。
- ZEND\_LIVE\_LOOP：这是一个foreach循环变量，它不仅包含简单的zval。这对应于FE_FREE操作码。
- ZEND\_LIVE\_SILENCE：用于实现错误抑制运算符。旧的错误报告级别备份到临时数据库中，稍后恢复。如果抛出异常，我们显然希望恢复它。这对应于END_SILENCE。
ZEND\_LIVE\_ROPE：用于绳索串连接，在这种情况下，临时数据是位于zend\_string*堆栈上的固定大小的指针数组 。在这种情况下，所有已经被填充的字符串都必须被释放。大致对应于END_ROPE。
在这种情况下要考虑的一个棘手问题是，如果它们的产生或消费指令抛出，是否应该释放临时对象。考虑下面的简单代码：
```
T2 = ADD T0, T1
ASSIGN $v, T2
```

如果ADD引发异常，T2临时应该自动释放还是ADD指令对此负责？同样，如果ASSIGN抛出，T2应该自动释放，还是ASSIGN必须自己处理？在后一种情况下，答案是明确的：即使抛出异常，指令总是负责释放其操作数。

结果操作数的情况比较棘手，因为这里的答案在PHP 7.1和7.2之间改变了：在PHP 7.1中，指令负责在发生异常时释放结果。在PHP7.2中，它被自动释放（并且该指令负责确保总是填充结果）。这种变化的动机是很多基本指令（如ADD）的实施方式。他们通常的结构大致如下：
```
1. read input operands
2. perform operation, write it into result operand
3. free input operands (if necessary)
```

这是有问题的，因为PHP处于非常不幸的位置，不仅支持异常和析构函数，而且还支持抛出析构函数（这是编译器工程师惊恐地哭泣的地步）。因此，第3步可以抛出，在这一点上结果已经填充。为避免这种边缘情况下的内存泄漏，释放结果操作数的责任已从指令转移到异常处理机制。

一旦我们执行了这些清理操作，我们就可以继续执行catch块。如果没有catch（最后也没有），我们展开堆栈，也就是销毁当前的堆栈帧并在处理异常时给父帧一个`shot`。

因此，您可以充分理解整个异常处理业务的丑陋程度，我将介绍与抛出析构函数相关的另一个小技巧。这在实践中并不是很相关，但我们仍然需要处理它以确保正确性。考虑这个代码：
```
foreach (new Dtor as $value) {
    try {
        echo "Return";
        return;
    } catch (Exception $e) {
        echo "Catch";
    }
}
```

现在想象这Dtor是一个带有析构函数的Traversable类。此代码将导致以下操作码序列，并且为了可读性而缩进循环主体：

```
L0:   V0 = NEW 'Dtor', ->L2
L1:   DO_FCALL
L2:   V2 = FE_RESET_R V0, ->L11
L3:   FE_FETCH_R V2, $value
L4:       ECHO 'Return'
L5:       FE_FREE (free on return) V2   # <- return
L6:       RETURN null                   # <- return
L7:       JMP ->L10
L8:       CATCH 'Exception' $e
L9:       ECHO 'Catch'
L10:  JMP ->L3
L11:  FE_FREE V2                        # <- the duplicated instr
```

重要的是，请注意“返回”被编译为循环变量的FE\_FREE和RETURN。现在，如果FE\_FREE抛出，会发生什么情况，因为Dtor抛出析构函数？通常，我们会说这个指令在try块内，所以我们应该调用catch。但是，在这一点上，循环变量已经被破坏！该catch抛弃异常，我们将尝试继续迭代已经死循环变量。

造成这个问题的原因是，当引发FE\_FREE在try块内时，它是L11中FE\_FREE的副本。从逻辑上讲，这是发生异常的地方。这就是为什么中断生成的FE\_FREE被注释为FREE\_ON\_RETURN的原因。这指示异常处理机制将异常源移至原始释放指令。因此，上面的代码不会运行catch块，它会生成一个未捕获的异常。

## Finally处理（Finally handling）
PHP与finally块相关的历史有点麻烦。PHP 5.5首先引入了最终块，或者更确切地说：最终块的一个非常错误的实现。PHP 5.6,7.0和7.1中的每一个都随着最终实现的重写而发布，每个都修复了大量错误，但未能完全实现完全正确的实现。

在编写本节时，我很惊讶地发现，从当前的实施和我目前的理解来看，最终处理实际上并不复杂。事实上，在许多方面，通过不同的迭代实现变得更简单，而不是更复杂。这表明对问题的理解不足可能会导致过于复杂和错误的实现（虽然公平地说，PHP 5实现的复杂性的一部分直接源于AST的缺乏）。

通常只要控件退出try块，正常（例如使用返回）或异常（通过抛出）就会运行Finally块。有几个有趣的边缘案例需要考虑，我将在进入实施之前快速阐述。请考虑：
```
try {
    throw new Exception();
} finally {
    return 42;
}
```

怎么了？最后获胜并且函数返回42。请考虑：
```
try {
    return 24;
} finally {
    return 42;
}
```

finally再次赢了，函数返回42。finally总是赢。

PHP禁止跳出finally块。例如以下是禁止的：
```
foreach ($array as $value) {
    try {
        return 42;
    } finally {
        continue;
    }
}
```

上面的代码示例中的“continue”将生成一个编译错误。重要的是要明白，这个限制是纯粹`示例代码`(cosmetic)，并可以通过使用“众所周知的”catch控制委托模式轻松解决：
```
foreach ($array as $value) {
    try {
        try {
            return 42;
        } finally {
            throw new JumpException;
        }
    } catch (JumpException $e) {
        continue;
    }
}
```

存在的唯一真正的限制是，它是不可能跳跃到 finally块，例如执行来自finally外部的goto跳转到finally内标签是被禁止的。

使用这个方式，我们可以初步的看看finally如何工作。该实现使用两个操作码FAST\_CALL和FAST\_RET。粗略地说，FAST\_CALL用于跳到finally块，FAST\_RET用于跳出它。让我们考虑最简单的情况：
```
try {
    echo "try";
} finally {
    echo "finally";
}
echo "finished";
```

此代码编译到以下操作码序列：
```
L0:   ECHO string("try")
L1:   T0 = FAST_CALL ->L3
L2:   JMP ->L5
L3:   ECHO string("finally")
L4:   FAST_RET T0
L5:   ECHO string("finished")
L6:   RETURN int(1)
```

FAST\_CALL将自己的位置存储到T0中，并跳转到L3处的finally块中。当达到FAST_RET时，它跳回到T0中存储的位置（之后）。在这种情况下，L2围绕finally块跳转。这是没有特殊控制流程（返回或异常）发生的基本情况。现在我们来看一下这个特例：
```
try {
    throw new Exception("try");
} catch (Exception $e) {
    throw new Exception("catch");
} finally {
    throw new Exception("finally");
}
```

处理异常时，我们必须考虑抛出的异常相对于最接近的周围try / catch / finally块的位置：

 1. 从try中抛出，并匹配catch：填充$e并跳入catch。
 2. 从try或catch中抛出，如果存在finally块：跳转到finally块，并且这次将异常备份到FAST\_CALL临时变量（而不是在那里存储返回地址）。
 3. 从finally抛出：如果备份异常存在临时FAST_CALL中的，则将其作为先前抛出异常的异常链接。继续将异常冒泡到下一个try / catch / finally。
 4. 否则：继续将异常冒泡到下一个try / catch / finally。

在这个例子中，我们将通过前三个步骤：首先尝试抛出，引发跳入catch。Catch也会抛出，触发到finally块的跳转，除非在FAST_CALL临时中备份。然后finally块也会抛出，这样，“finally”异常将会像之前的异常一样被设置为“catch”异常。

前面例子中的一个小变化是以下代码：
```
try {
    try {
        throw new Exception("try");
    } finally {}
} catch (Exception $e) {
    try {
        throw new Exception("catch");
    } finally {}
} finally {
    try {
        throw new Exception("finally");
    } finally {}
}
```

这里的所有内部最终块都是异常输入，但通常保持不变（通过FAST\_RET）。在这种情况下，先前描述的异常处理过程从父try / catch / finally块开始继续。这个父try / catch存储在FAST_RET操作码中（这里是“try-catch（0）”）。

这基本上涵盖finally和exceptions的关系。但finaly的返回呢？

```
try {
    throw new Exception("try");
} finally {
    return 42;
}
```

操作码序列的相关部分是这样的：

```
L4:   T0 = FAST_CALL ->L6
L5:   JMP ->L9
L6:   DISCARD_EXCEPTION T0
L7:   RETURN 42
L8:   FAST_RET T0
```

额外的DISCARD_EXCEPTION操作码负责放弃在try块中抛出的异常（记住：finally获胜的返回）。那么try的返回呢？
```
try {
    $a = 42;
    return $a;
} finally {
    ++$a;
}
```

这里的例外返回值是42，而不是43.返回值由return\$a行确定，任何进一步的修改$a都不重要。代码结果：
```
L0:   ASSIGN $a, 42
L1:   T3 = QM_ASSIGN $a
L2:   T1 = FAST_CALL ->L6, T3
L3:   RETURN T3
L4:   T1 = FAST_CALL ->L6      # unreachable
L5:   JMP ->L8                 # unreachable
L6:   PRE_INC $a
L7:   FAST_RET T1
L8:   RETURN null
```

其中两个操作码无法访问，因为它们在返回后发生。这些将在优化期间被删除，但我在这里显示未优化的操作码。这里有两件有趣的事情：首先，
\$a使用QM\_ASSIGN（基本上是“复制到临时变量”指令）复制到T3中。这是防止后来修改$a 影响返回值的原因。其次，T3也传递给FAST\_CALL，它将备份T1中的值。如果try块的返回值稍后被丢弃（例如，因为最后抛出或返回），则此机制将用于释放未使用的返回值。

所有这些单独的机制都很简单，但是在组成时需要谨慎。考虑下面的例子，其中又Dtor是一些带有抛出析构函数的Traversable类：
```
try {
    foreach (new Dtor as $v) {
        try {
            return 1;
        } finally {
            return 2;
        }
    }
} finally {
    echo "finally";
}
```

该代码生成以下操作码：
```
L0:   V2 = NEW (0 args) "Dtor"
L1:   DO_FCALL
L2:   V4 = FE_RESET_R V2 ->L16
L3:   FE_FETCH_R V4 $v ->L16
L4:       T5 = FAST_CALL ->L10         # inner try
L5:       FE_FREE (free on return) V4
L6:       T1 = FAST_CALL ->L19
L7:       RETURN 1
L8:       T5 = FAST_CALL ->L10         # unreachable
L9:       JMP ->L15
L10:      DISCARD_EXCEPTION T5         # inner finally
L11:      FE_FREE (free on return) V4
L12:      T1 = FAST_CALL ->L19
L13:      RETURN 2
L14:      FAST_RET T5 try-catch(0)
L15:  JMP ->L3
L16:  FE_FREE V4
L17:  T1 = FAST_CALL ->L19
L18:  JMP ->L21
L19:  ECHO "finally"                   # outer finally
L20:  FAST_RET T1
```

第一次返回的序列（来自内部try）是FAST\_CALL L10，FE\_FREE V4，FAST\_CALL L19，RETURN。这将首先调用内部finally块，然后释放foreach循环变量，然后调用外部finally块并返回。第二次返回的序列（最终来自内部）是DISCARD\_EXCEPTION T5，FE\_FREE V4，FAST\_CALL L19。首先放弃内部try块的异常（或这里：返回值），然后释放foreach循环变量并最终调用外部finally块。请注意，在这两种情况下，这些指令的顺序是源代码中相关块的反向顺序。

## 生成器（Generators）
发生器功能可能会暂停和恢复，因此需要特殊的VM栈管理。这是一个简单的生成器：
```
function gen($x) {
    foo(yield $x);
}
```
这会产生以下操作码：
```
$x = RECV 1
GENERATOR_CREATE
INIT_FCALL_BY_NAME (1 args) string("foo")
V1 = YIELD $x
SEND_VAR_NO_REF_EX V1 1
DO_FCALL_BY_NAME
GENERATOR_RETURN null
```

在到达GENERATOR\_CREATE之前，这是在普通VM堆栈上作为普通函数执行的。GENERATOR\_CREATE然后创建一个Generator对象，以及一个堆分配的execute\_data结构（像往常一样包括变量和参数的槽），VM栈上的execute_data被复制到其中。

当生成器再次恢复时，执行器将使用堆分配的execute_data，但将继续使用主VM堆栈来推送调用帧。一个明显的问题是，如前面的例子所示，在调用过程中可能会中断发生器。这里YIELD是在调用foo()的调用帧已经被压入VM栈的时候执行的。

这种相对不常见的情况是通过在产生控制时将调用帧复制到发生器结构中并在发生器恢复时恢复它们来处理。

这个设计自PHP 7.1起使用。以前，每个发生器都有自己的4KiB虚拟机页面，当发生器恢复时它将被交换到执行器。这样可以避免复制调用帧，但会增加内存使用量。

## 智能分支（Smart branches）
比较指令直接跟随条件跳转是很常见的。例如：

```
L0:   T2 = IS_EQUAL $a, $b
L1:   JMPZ T2 ->L3
L2:   ECHO "equal"
```

由于这种模式非常普遍，所有比较操作码（如IS_EQUAL）都实现了智能分支机制：它们检查下一条指令是JMPZ还是JMPNZ指令，如果是，则自行执行相应的跳转操作。

智能分支机制只检查下一条指令是否是JMPZ/JMPNZ，但实际上并不检查其操作数是否实际上是比较的结果或其他。在比较和随后的跳跃不相关的情况下，这需要特别小心。例如，代码($a == $b) + ($d ? $e : $f)生成：
```
L0:   T5 = IS_EQUAL $a, $b
L1:   NOP
L2:   JMPZ $d ->L5
L3:   T6 = QM_ASSIGN $e
L4:   JMP ->L6
L5:   T6 = QM_ASSIGN $f
L6:   T7 = ADD T5 T6
L7:   FREE T7
```

请注意，在IS\_EQUAL和JMPZ之间插入了NOP。如果这个NOP不存在，分支将最终使用IS_EQUAL结果，而不是JMPZ操作数。

## 运行时缓存（Runtime cache）
由于操作码数组在多个进程之间共享（无锁），因此它们是不可变的。但是，运行时值可以缓存在单独的“运行时缓存”中，该缓存基本上是一个指针数组。Literals可能有一个关联的运行时缓存条目（或多个），它存储在它们的u2插槽中。

运行时高速缓存条目有两种类型：第一种是普通高速缓存条目，例如INIT\_FCALL使用的条目。INIT\_FCALL查找一次被调用的函数（根据其名称）后，函数指针将被缓存在关联的运行时缓存槽中。

第二种类型是多态高速缓存条目，它们只是两个连续的高速缓存槽，其中第一个存储类条目，第二个存储实际数据。这些用于像FETCH\_OBJ\_R这样的操作，其中某个类的属性表中属性的偏移量被缓存。如果下一次访问发生在同一个类上（很有可能），则将使用缓存的值。否则，将执行更昂贵的查找操作，并将结果缓存到新的类条目中。

## VM中断（VM interrupts）
在PHP 7.0之前，执行超时用于直接从信号处理程序通过longjump处理关闭序列。如你所想，这引起了各种不愉快的事情。由于PHP 7.0超时被延迟，直到控制权返回到虚拟机。如果它在特定的宽限期内没有返回，则该过程被中止。由于PHP 7.1 pcntl信号处理程序使用与执行超时相同的机制。

当一个信号挂起时，VM中断标志被设置，并且这个标志由虚拟机在某些点检查。检查不是在每条指令上执行，而是仅在跳转和调用时执行。因此，在返回到VM时不会立即处理中断，而是在线性控制流的当前部分结束时处理。

## 特殊化（Specialization）
如果查看VM定义文件，会发现操作码处理程序定义如下：

`ZEND_VM_HANDLER(1, ZEND_ADD, CONST|TMPVAR|CV, CONST|TMPVAR|CV)`

1在这里是操作码数，ZEND\_ADD它的名字，而另外两个参数指定指令接受哪些类型的操作。所述生成的虚拟机代码（由生成zend_vm_gen.php然后）将包含为每个可能的操作数类型的组合的专门处理程序。生成有像ZEND\_ADD\_SPEC\_CONST\_CONST\_HANDLER这样的名称。

专用处理程序是通过替换处理程序主体中的某些宏来生成的。比较容易看出的是OP1\_TYPE和OP2\_TYPE，但诸如GET\_OP1\_ZVAL\_PTR（）和FREE\_OP1（）等操作也是专用的。

ADD的处理程序指定它接受CONST|TMPVAR|CV操作数。这里的TMPVAR意味着操作码同时接受TMP和VAR，但要求这些不是单独专用的。请记住，对于大多数用途而言，TMP和VAR之间的唯一区别是后者可以包含引用。对于像ADD这样的操作码（无论如何，引用都在慢速路径（slow-path）上），单独进行特殊化并不值得。一些其他操作码确实可以区分TMP|VAR它们的操作数列表。

除了基于操作数类型的特殊化之外，处理程序还可以专门处理其他因素，例如是否使用其返回值。ASSIGN\_DIM基于以下OP\_DATA操作码的操作数类型进行特殊化：
```
ZEND_VM_HANDLER(147, ZEND_ASSIGN_DIM,
    VAR|CV, CONST|TMPVAR|UNUSED|NEXT|CV, SPEC(OP_DATA=CONST|TMP|VAR|CV))
```

根据此签名，将生成2*4*4=32个不同的ASSIGN_DIM变体。第二个操作数的规范也包含一个条目NEXT。这与特殊化无关，而是指定了UNUSED操作数在此上下文中的含义：它表示这是一个附加操作（$arr[]）。另一个例子：
```
ZEND_VM_HANDLER(23, ZEND_ASSIGN_ADD,
    VAR|UNUSED|THIS|CV, CONST|TMPVAR|UNUSED|NEXT|CV, DIM_OBJ, SPEC(DIM_OBJ))
```

在这里我们已经知道第一个操作数是UNUSED意味着一个访问\$this。这是与对象有关的操作码的一般惯例，例如FETCH\_OBJ\_R UNUSED,'prop'对应于\$this->prop。未使用的第二个操作数也意味着附加操作。这里的第三个参数指定的extended\_value数的含义：它包含区分的标志\$a += 1，\$a[\$b] += 1和\$a->b += 1。最后，SPEC(DIM_OBJ)指示应该为每一个生成一个专门的处理程序。（在这种情况下，将生成的总处理程序数量并不重要，因为VM生成器知道某些组合是不可能的，例如UNUSED op1只与OBJ情况相关，等等）

最后，虚拟机生成器还支持更复杂的特殊化机制。在定义文件的末尾，你会发现一些这种形式的处理程序：
```
ZEND_VM_TYPE_SPEC_HANDLER(
    ZEND_ADD,
    (res_info == MAY_BE_LONG && op1_info == MAY_BE_LONG && op2_info == MAY_BE_LONG),
    ZEND_ADD_LONG_NO_OVERFLOW,
    CONST|TMPVARCV, CONST|TMPVARCV, SPEC(NO_CONST_CONST,COMMUTATIVE)
)
```
这些处理程序不仅基于VM操作数类型，还基于操作数在运行时可能采用的可能类型。确定可能的操作数类型的机制是opcache优化基础结构的一部分，这超出了本文的范围。但是，假设这些信息可用，应该清楚这是int + int -> int一个附加的形式。此外，SPEC注释告诉specilizer，不应该生成两个const操作数的变体，并且操作是可交换的，所以如果我们已经有了CONST + TMPVARCV特殊化，我们也不需要生成TMPVARCV + CONST。

## 快速路径/慢速路径分割（Fast-path / slow-path split）
许多操作码处理程序都是使用快速路径/慢速路径分割来实现的，其中首先处理几个常见情况，然后再回到通用实现。这是关于我们查看一些实际代码的时间了，所以我将在这里粘贴整个SL（左移）实现：


```
ZEND_VM_HANDLER(6, ZEND_SL, CONST|TMPVAR|CV, CONST|TMPVAR|CV)
{
	USE_OPLINE
	zend_free_op free_op1, free_op2;
	zval *op1, *op2;

	op1 = GET_OP1_ZVAL_PTR_UNDEF(BP_VAR_R);
	op2 = GET_OP2_ZVAL_PTR_UNDEF(BP_VAR_R);
	if (EXPECTED(Z_TYPE_INFO_P(op1) == IS_LONG)
			&& EXPECTED(Z_TYPE_INFO_P(op2) == IS_LONG)
			&& EXPECTED((zend_ulong)Z_LVAL_P(op2) < SIZEOF_ZEND_LONG * 8)) {
		ZVAL_LONG(EX_VAR(opline->result.var), Z_LVAL_P(op1) << Z_LVAL_P(op2));
		ZEND_VM_NEXT_OPCODE();
	}

	SAVE_OPLINE();
	if (OP1_TYPE == IS_CV && UNEXPECTED(Z_TYPE_INFO_P(op1) == IS_UNDEF)) {
		op1 = GET_OP1_UNDEF_CV(op1, BP_VAR_R);
	}
	if (OP2_TYPE == IS_CV && UNEXPECTED(Z_TYPE_INFO_P(op2) == IS_UNDEF)) {
		op2 = GET_OP2_UNDEF_CV(op2, BP_VAR_R);
	}
	shift_left_function(EX_VAR(opline->result.var), op1, op2);
	FREE_OP1();
	FREE_OP2();
	ZEND_VM_NEXT_OPCODE_CHECK_EXCEPTION();
}
```

该实现通过GET\_OPn\_ZVAL\_PTR\_UNDEF以BP\_VAR\_R模式获取操作数开始。UNDEF这里的部分意味着在CV情况下不执行未定义变量的检查，而只是按照原样返回UNDEF值。一旦我们有操作数，我们检查两者是否都是整数，并且移位宽度在范围内，在这种情况下，结果可以直接计算出来，并且我们前进到下一个操作码。值得注意的是，这里的类型检查并不关心操作数是否为UNDEF，所以使用GET\_OPN\_ZVAL\_PTR\_UNDEF是合理的。

如果操作数不能满足快速路径，我们回到通用实现，该实现以SAVE\_OPLINE（）开始。这是我们的信号“潜在的投掷操作”。在继续之前，处理未定义变量的情况。在这种情况下，GET\_OPn\_UNDEF\_CV将发出未定义的变量通知并返回NULL值。

接下来，调用通用shift\_left\_function并将其结果写入EX_VAR(opline->result.var)。最后，输入操作数被释放（如果需要），并且我们前进到具有异常检查的下一个操作码（这意味着在前进之前重新装入操作数）。

因此，这里的快速路径保存了未定义变量的两个检查，对通用运算符函数的调用，释放操作数，以及保存和重新加载opline以进行异常处理。大部分性能敏感的操作码都以相似的方式排列。

## VM宏（VM macros）
从前面的代码清单可以看出，虚拟机实现可以自由使用宏。其中一些是普通的C宏，而另一些则是在生成虚拟机时解决的。特别是，这包括许多用于获取和释放指令操作数的宏：

```
OPn_TYPE
OP_DATA_TYPE

GET_OPn_ZVAL_PTR(BP_VAR_*)
GET_OPn_ZVAL_PTR_DEREF(BP_VAR_*)
GET_OPn_ZVAL_PTR_UNDEF(BP_VAR_*)
GET_OPn_ZVAL_PTR_PTR(BP_VAR_*)
GET_OPn_ZVAL_PTR_PTR_UNDEF(BP_VAR_*)
GET_OPn_OBJ_ZVAL_PTR(BP_VAR_*)
GET_OPn_OBJ_ZVAL_PTR_UNDEF(BP_VAR_*)
GET_OPn_OBJ_ZVAL_PTR_DEREF(BP_VAR_*)
GET_OPn_OBJ_ZVAL_PTR_PTR(BP_VAR_*)
GET_OPn_OBJ_ZVAL_PTR_PTR_UNDEF(BP_VAR_*)
GET_OP_DATA_ZVAL_PTR()
GET_OP_DATA_ZVAL_PTR_DEREF()

FREE_OPn()
FREE_OPn_IF_VAR()
FREE_OPn_VAR_PTR()
FREE_UNFETCHED_OPn()
FREE_OP_DATA()
FREE_UNFETCHED_OP_DATA()
```


可以看到，这里有不少变化。该BP\_VAR\_*参数指定的提取模式并支持相同的模式作为FETCH\_ *（与FUNC_ARG除外）的说明。

GET\_OPn\_ZVAL\_PTR()是基本的操作数获取。它会在未定义的CV上发出通知，并且不会取消操作数的取消引用。GET\_OPn\_ZVAL\_PTR\_UNDEF()正如我们已经知道的那样，它是一种不检查未定义的CV的变体。 GET\_OPn\_ZVAL\_PTR\_DEREF()包括zval的DEREF。这是专门的GET操作的一部分，因为解引用仅对于CV和VAR是必需的，但对于CONST和TMP不是必需的。因为这个宏需要区分TMP和VAR，它只能用于TMP|VAR专业化（但不能TMPVAR）。

这些GET\_OPn\_OBJ\_ZVAL\_PTR\*()变体还处理UNUSED操作数的情况。如前所述，通过约定$this访问使用UNUSED操作数，所以GET\_OPn\_OBJ\_ZVAL\_PTR*()宏将返回EX(This)对UNUSED操作的引用。

最后，还有一些PTR\_PTR变体。这里的命名是来自PHP5，其中这实际上使用了双向的zval指针。这些宏用于写操作，因此仅支持CV和VAR类型（其他任何返回NULL）。它们与正常的PTR提取不同，因为它们取消了VAR操作数。

这些FREE\_OP*()宏然后用来释放取出的操作数。要进行操作，它们需要定义一个zend\_free\_op free\_opN变量，GET操作将该 变量存储到该变量中以释放。基线FREE\_OPn()操作将释放TMP和VAR，但不会释放CV和CONST。FREE\_OPn\_IF\_VAR()完全按照它的说法：只有当它是VAR时才释放操作数。

该FREE\_OP*\_VAR\_PTR()变体与PTR\_PTR提取结合使用。它只会释放VAR操作数，并且只有在它们不是INDIRECTed的时候。

FREE\_UNFETCHED\_OP*()在使用GET获取操作数之前必须释放操作数的情况下使用这些变体。如果在操作数提取之前抛出异常，通常会发生这种情况。

除了这些特殊的宏之外，还有一些比较普通的宏。虚拟机定义了三个宏来控制操作码处理程序运行后发生的情况：


```
ZEND_VM_CONTINUE()
ZEND_VM_ENTER()
ZEND_VM_LEAVE()
ZEND_VM_RETURN()
```

CONTINUE将继续正常执行操作码，而ENTER和LEAVE则用于进入/离开嵌套函数调用。这些操作的具体细节取决于VM编译的精确程度（例如，是否使用全局寄存器，如果是，使用哪一个）。从广义上讲，这些将在继续之前同步一些状态。RETURN用于实际退出主VM环路。

ZEND_VM_CONTINUE（）期望事先更新opline。当然，还有更多的宏相关：

|     -     | Continue? | Check exception? | Check interrupt? |
|----------|-----------|------------------|------------------|
|ZEND_VM_NEXT_OPCODE() |   yes   |  no  |  no  |
|ZEND_VM_NEXT_OPCODE_CHECK_EXCEPTION()| yes | yes | no |
|ZEND_VM_SET_NEXT_OPCODE(op) |   no  |  no  |  no  |
|ZEND_VM_SET_OPCODE(op)     |   no  |   no  |  yes |
|ZEND_VM_SET_RELATIVE_OPCODE(op, offset) |  no | no | yes |
|ZEND_VM_JMP(op)|  yes  |       yes        |       yes      |

该表显示宏是否包含隐式ZEND\_VM\_CONTINUE（），是否会检查异常以及是否检查VM中断。

除此之外，还有SAVE\_OPLINE()，LOAD\_OPLINE()和HANDLE\_EXCEPTION()。正如在异常处理一节中所提到的，SAVE\_OPLINE（）在操作码处理程序中的第一个可能的抛出操作之前使用。如有必要，它将VM（可能位于全局寄存器中）使用的opline写回到执行数据中。LOAD\_OPLINE（）是相反的操作，但现在它几乎没有用处，因为它已被有效地转入ZEND_VM_NEXT\_OPCODE_CHECK_EXCEPTION（）和ZEND\_VM\_JMP（）。

HANDLE\_EXCEPTION（）用于在已经知道引发异常后从操作码处理程序返回。它执行LOAD\_OPLINE和CONTINUE的组合，这将有效地分派到HANDLE\_EXCEPTION操作码。

当然，还有更多的宏，但以上应该已经包含了最重要的部分。

译自：[PHP 7 Virtual Machine][1]


  [1]: https://nikic.github.io/2017/04/14/PHP-7-Virtual-machine.html#writes-and-memory-safety
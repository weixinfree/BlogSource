---
title: "基于栈的虚拟机"
date: 2018-11-26T20:50:28+08:00
draft: false
tags: ['java','python','stack-base', 'vm', 'calc']
---

java语言虚拟机的标准实现是基于栈的，jvm 规范定义的方法栈帧里还有一个专门的操作数栈。除了java之外，还有很多语言实现了基于栈的虚拟机，比如：python，Lua的早期版本应该也是基于栈的，后来为了执行效率考虑改为了基于寄存器的虚拟机；此外还有stack-base语言例如forth，red等等。

> 注意：虽然Android App开发使用java语言，但是Android的Dalvik和Art虚拟机，是基于寄存器的虚拟机

基于堆栈的虚拟机的有以下好处：

1. 堆栈机实现起来更简单一些; 
2. 堆栈机的可移植性更好一些，原则上基于栈的虚拟机原则上只需要一个寄存器 PC即可，用来存储下一条指令的位置 
3. 给予栈的虚拟机一般自己码更紧凑一些，可以做到更小的二进制体积 

### JVM 对操作数栈的介绍
> 2.6.2 Operand Stacks

> Each frame (§2.6) contains a last-in-first-out (LIFO) stack known as its operand
stack. The maximum depth of the operand stack of a frame is determined at
compile-time and is supplied along with the code for the method associated with
the frame (§4.7.3).

> Where it is clear by context, we will sometimes refer to the operand stack of the
current frame as simply the operand stack.
The operand stack is empty when the frame that contains it is created.

> The
Java Virtual Machine supplies instructions to load constants or values from local
variables or fields onto the operand stack. Other Java Virtual Machine instructions
take operands from the operand stack, operate on them, and push the result back
onto the operand stack. The operand stack is also used to prepare parameters to be
passed to methods and to receive method results.

> For example, the iadd instruction (§iadd) adds two int values together. It requires
that the int values to be added be the top two values of the operand stack, pushed
there by previous instructions. Both of the int values are popped from the operand
stack. They are added, and their sum is pushed back onto the operand stack.
Subcomputations may be nested on the operand stack, resulting in values that can
be used by the encompassing computation

### Java 字节码例子
```java
public class Demo {

    public static void main(String[] args) {
        System.out.println("hello world!");
    }
}
```

```bash
public class Demo {
  public Demo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #3                  // String hello world!
       5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return
}
```

解释: 
`0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;`

- 0 代表code偏移量, `getstatic` 需要一个字节表示字节码，2个字节表示参数，所以`ldc`的code偏移量为3
- `getstatic` 代表java字节码指令 （获取一个静态变量）
- `#2` 代表`getstatic` 需要的参数（class 常量池index），时机的value是：Field java/lang/System.out:Ljava/io/PrintStream;

所以依据字节码解释main函数是：

1.  pc 偏移量为0，执行 `getstatic` 参数为当前类的class常量池索引`#2` (Field java/lang/System.out:Ljava/io/PrintStream;)，执行结果push到操作数栈中 （top = 0）
2. pc 偏移量为3，执行`ldc`，加载常量，参数为常量池索引位置为 #2 (String hello world!)，将结果push到操作数栈（top = 2）
3. pc 偏移量为5，执行 `invokespecial`，参数为常量池索引位置 #4(java/io/PrintStream.println:(Ljava/lang/String;)V)，并从操作数栈中弹出 1 作为参数，0作为objref，调用 objref.println("hello world!")
4. pc 偏移量为8，执行`return`，方法结束，栈帧弹出


### Python 字节码例子
```python
>>> def hello():
...     print("hello world")
...
...
>>> from dis import dis
>>> dis(hello)
  2           0 LOAD_GLOBAL              0 (print)
              2 LOAD_CONST               1 ('hello world')
              4 CALL_FUNCTION            1
              6 POP_TOP
              8 LOAD_CONST               0 (None)
             10 RETURN_VALUE
>>> def add(l, r):
...     result = l + r
...     return result
...
>>> dis(add)
  2           0 LOAD_FAST                0 (l)
              2 LOAD_FAST                1 (r)
              4 BINARY_ADD
              6 STORE_FAST               2 (result)
  3           8 LOAD_FAST                2 (result)
             10 RETURN_VALUE
>>>
```
可以看到python字节码格式和java字节码非常相似，不过python字节码第一列添加了代码行号; 字节码偏移是两个字节对齐的，而java的字节码都是单字节的

## 基于栈的四则运算器（逆波兰表达式）

有了上面的知识，我们就可以开发一个简单的基于栈的四则运算器了

先看下效果：
```bash
~/Downloads » calc "2 2 + 10 - 6 / 5 *"                            
-5.0
------------------------------------------------------------
~/Downloads » calc "2 2 + 10 - 6 / 5 * sin"                        
0.9589242746631385
------------------------------------------------------------
~/Downloads » calc "2 2 + 10 - 6 / 5 * sin log2"                   
-0.0605112033994035
------------------------------------------------------------
~/Downloads » calc "2 2 + 10 - 6 / 5 * sin log2 hex"               
0x0
------------------------------------------------------------
~/Downloads » calc "2 2 + 10 - 6 / 5 * sin log2 2 pow"             
0.003661605736843982
```

可以看到这个简单的计算器支持四则运算，三角函数，log，指数等函数。借助stack虚拟机的思想，可以只使用很少的代码实现这些功能。

##### 核心代码如下：
```python
def calc(_cmd: str):
    stack: List[float] = []

    cmds = _cmd.split()
    for item in cmds:
        if is_number(item):
            stack.append(float(item))
        else:
            do_op(item, stack)

    return stack.pop()
```
##### 使用正则表达式判断是否是数字：
```python
def is_number(value: str):
    return re.match(r'[0-9]+(.[0-9]+)?', value)
```

##### 针对操作符，可以直接查表：
```python
OP_MAP = {
    '+': binary_op(operator.add),
    '*': binary_op(operator.mul),
    '-': binary_op(operator.sub),
    '/': binary_op(operator.truediv),
    'pow': binary_op(operator.pow),

    'pos': unary_op(operator.pos),
    'neg': unary_op(operator.neg),
    'hex': unary_op(combine(int, hex)),
    'bin': unary_op(combine(int, bin)),

    'sin': unary_op(math.sin),
    'cos': unary_op(math.cos),
    'log2': unary_op(math.log2),
    'log10': unary_op(math.log10),

    'pi': constant(math.pi),
    'e': constant(math.e)
}

def do_op(op: str, stack: List):
    OP_MAP[op](stack)
```

##### unary_op, constant 和 binary_op封装栈的操作的同时给出了非常清晰的语义
```python
def constant(value: float):
    def inner(stack: List[float]):
        stack.append(value)

    return inner

def unary_op(f):
    @wraps(f)
    def inner(stack: List[float]):
        value = stack.pop()
        stack.append(f(value))
    return inner

def binary_op(f):
    @wraps(f)
    def inner(stack: List[float]):
        right = stack.pop()
        left = stack.pop()
        stack.append(f(left, right))
    return inner
```



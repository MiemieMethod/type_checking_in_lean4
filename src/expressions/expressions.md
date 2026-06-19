# 表达式

## 完整语法

下面会更详细地解释表达式。不过为了先把全貌摆出来，包含字符串和自然数字面量扩展在内的表达式完整语法如下：

```
Expr ::= 
  | boundVariable 
  | freeVariable 
  | const 
  | sort 
  | app 
  | lambda 
  | forall 
  | let 
  | proj 
  | literal

BinderInfo ::= Default | Implicit | InstanceImplicit | StrictImplicit

const ::= Name, Level*
sort ::= Level
app ::= Expr Expr
-- 一个 deBruijn 指标
boundVariable ::= Nat
lambda ::= Name, (binderType : Expr), BinderInfo, (body : Expr)
forall ::= Name, (binderType : Expr), BinderInfo, (body : Expr)
let ::= Name, (binderType : Expr), (val : Expr) (body : Expr)
proj ::= Name Nat Expr
literal ::= natLit | stringLit

-- 任意精度自然数/无符号整数
natLit ::= Nat
-- UTF-8 字符串
stringLit ::= String

-- fvarId 可以用唯一名称或 deBruijn 层级实现；
-- 唯一名称更灵活，deBruijn 层级有更好的缓存行为。
freeVariable ::= Name, Expr, BinderInfo, fvarId
```

几点说明：

+ 自然数字面量所用的 `Nat` 应当是任意精度自然数/大整数。

+ 带有绑定子的表达式（lambda、pi、let、自由变量）也完全可以把三个参数（绑定子名称、绑定子类型、绑定子样式）打包成一个参数 `Binder`，其中绑定子为 `Binder ::= Name BinderInfo Expr`。在别处出现的伪代码中，我通常会像它们具有这种性质一样处理，因为这样更易读。

+ 自由变量标识符可以是唯一标识符，也可以是 de Bruijn 层级。

+ Lean 本身使用的表达式类型还有一个 `mdata` 构造子，用来声明附带元数据的表达式。该元数据不影响表达式在内核中的行为，因此这里不包含这个构造子。


## 绑定子信息

用 lambda、pi、let 和自由变量构造子构造的表达式包含绑定子信息，其形式为一个名称、一个绑定子“样式”和绑定子的类型。绑定子的名称与样式只供漂亮打印器使用，不会改变推断、规约或相等性检查的核心过程。不过在漂亮打印器中，绑定子样式可能会根据漂亮打印选项改变输出。例如，用户可能想显示，也可能不想显示隐式参数或实例隐式参数（类型类变量）。

### Sort

`sort` 只是对层级的一层包装，使其可以作为表达式使用。


### 有界变量

有界变量被实现为表示 [de Bruijn 指标](https://en.wikipedia.org/wiki/De_Bruijn_index)的自然数。

### 自由变量

自由变量用于在绑定子当前不可用的情形下传达关于有界变量的信息。通常，这是因为内核已经遍历进入某个绑定表达式的函数体，并选择不携带一份结构化的绑定信息上下文，而是临时把有界变量替换为一个自由变量；之后可以再换入一个新的（也许不同的）有界变量来重建绑定子。这种“不可用”的描述可能听起来有些含糊，但一个或许有帮助的直观解释是：表达式被实现为没有任何父指针的树，因此当我们下降到子节点（尤其是跨越函数边界）时，就会失去对当前表达式中位于我们上方那些元素的视野。

当通过重建绑定子来闭合一个打开表达式时，绑定关系可能已经改变，从而使先前有效的 de Bruijn 指标失效。使用唯一名称或 de Bruijn 层级，可以用一种补偿这些变化的方式重新闭合绑定子，并确保重新绑定变量的新 de Bruijn 指标相对于重建后的望远镜是有效的（见[本节](../kernel_concepts/instantiation_and_abstraction.md#实现自由变量抽象)）。

下文中，我们可能会用某种形式的“自由变量标识符”一词，来指称某个实现所采用方案（唯一 ID 或 de Bruijn 层级）中的对象。

### `Const`

`const` 构造子是表达式引用环境中另一个声明的方式；它必须通过引用做到这一点。

在下面的例子中，`def plusOne` 创建了一个 `Definition` 声明，该声明被检查后进入环境。声明不能直接放入表达式中，因此当 `plusOne_eq_succ` 的类型调用先前的声明 `plusOne` 时，它必须通过名称来调用。于是创建表达式 `Expr.const (plusOne, [])`；当内核发现这个 `const` 表达式时，它可以在环境中按名称查找被引用的声明 `plusOne`：

```
def plusOne : Nat -> Nat := fun x => x + 1

theorem plusOne_eq_succ (n : Nat) : plusOne n = n.succ := rfl 
```

用 `const` 构造子创建的表达式还携带一组层级。通过查找该 `const` 表达式所引用的定义而从环境中取出的任何展开声明或推断声明，都会把这些层级替换进去。例如，对 `const List [Level.param(x)]` 推断类型时，要在当前环境中查找 `List` 的声明，取出它的类型和宇宙参数，然后把 `x` 替换为 `List` 最初声明时使用的宇宙参数。


### Lambda、Pi

`lambda` 和 `pi` 表达式（Lean 本身使用名称 `forallE` 而不是 `pi`）分别指函数抽象和“forall”绑定子（依赖函数类型）。


```
  binderName      body
      |            |
fun (foo : Bar) => 0 
            |         
        binderType    

-- `BinderInfo` 通过包围绑定子所用括号的样式体现。
```

### Let

`let` 正如其名。虽然 `let` 表达式也是绑定子，但它们没有 `BinderInfo`；其绑定子信息被视为 `Default`。

```
  binderName      val
      |            |
let (foo : Bar) := 0; foo
            |          |
        binderType     .... body
```


### App

`app` 表达式表示把一个实参应用到一个函数。App 节点是二叉的（只有两个子节点：一个函数和一个实参），因此 `f x_0 x_1 .. x_N` 会被表示为 `App(App(App(f, x_0), x_1)..., x_N)`，也可以可视化为如下树：

```
                App
                / \
              ...  x_N
              /
            ...
           App
          / \
       App  x_1
       / \
     f  x_0

```

操作表达式时，一个极其常见的内核操作，是折叠和展开一串应用：从上面的树结构得到 `(f, [x_0, x_1, .., x_N])`，或者把 `f, [x_0, x_1, .., x_N]` 折叠回上面的树。


### 投影

`proj` 构造子表示结构投影。只有一个构造子且没有指标的归纳类型可以作为结构。

这个构造子接受一个名称（类型的名称）、一个表示被投影字段的自然数，以及实际被投影的结构。

注意，在内核中，投影指标从 0 开始；而在 Lean 的交互式语言中它从 1 开始。这里的 0 表示跳过参数后的第一个字段。

例如，内核表达式 `proj Prod 0 (@Prod.mk A B a b)` 会投影出 `a`，因为跳过参数 `A` 和 `B` 后，它是第 0 个字段。

虽然 `proj` 所提供的行为也可以通过使用该类型的递归器实现，但 `proj` 能更高效地处理频繁出现的内核操作。

### 字面量

Lean 的内核可选地支持任意精度 Nat 和 String 字面量。必要时，内核可以把自然数字面量 `n` 转换为 `Nat.zero` 或 `Nat.succ m`，也可以把字符串字面量 `s` 转换为 `String.mk List.nil` 或 `String.mk (List.cons (Char.ofNat _) ...)`。

字符串字面量会在测试定义相等时，以及在作为递归器规约中的主前提出现时，被惰性地转换为字符列表。

自然数字面量在与字符串相同的位置得到支持（定义相等以及递归器应用的主前提），但内核还支持对自然数字面量进行加法、乘法、幂、减法、取模、除法，以及布尔相等和“小于等于”比较。

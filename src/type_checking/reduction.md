# 规约

规约的目的，是把表达式推向其[范式](https://en.wikipedia.org/wiki/Normal_form_(abstract_rewriting))，从而判断表达式是否定义相等。例如，为了判断 `(fun x => x) Nat.zero` 与 `Nat.zero` 定义相等，我们需要执行 beta 规约；为了判断 `id List.nil` 与 `List.nil` 定义相等，我们需要执行 delta 规约。

Lean 内核中的规约有两个性质，会引出一些在基础教材处理中有时未被讨论的问题。第一，在某些情况下，规约与推断是交错进行的。除其他事情外，这意味着规约可能需要在打开项上执行，尽管规约过程本身并不创建自由变量。第二，应用于多个实参的 `const` 表达式，在规约期间可能需要同这些实参一起考虑（如 iota 规约中那样），因此在规约开始时，需要一起展开一串应用。

## Beta 规约

Beta 规约处理函数应用的规约。具体而言，规约：

```
(fun x => x) a    ~~>    a
```

Beta 规约的实现必须对表达式去脊（despine），以收集 `app` 表达式中的所有实参；检查这些实参所应用到的表达式是否是 lambda；然后在函数体中，把对应有界变量的所有出现替换为适当的实参：

```
betaReduce e:
  betaReduceAux e.unfoldApps

betaReduceAux f args:
  match f, args with
  | lambda _ body, arg :: rest => betaReduceAux (inst body arg) rest
  | _, _ => foldApps f args
```

对于 beta 规约的实例化（替换）部分，一个重要的性能优化有时被称为“广义 beta 规约”。它会收集那些有对应 lambda 的实参，并一次性替换它们。这个优化意味着，对于 `n` 个连续的、施加了实参的 lambda 表达式，我们只需遍历表达式一次来替换相应实参，而不是遍历 `n` 次。

```
betaReduce e:
  betaReduceAux e.unfoldApps []

betaReduceAux f remArgs argsToApply:
  match f, remArgs with
  | lambda _ body, arg :: rest => betaReduceAux body rest (arg :: argsToApply)
  | _, _ => foldApps (inst f argsToApply) remArgs
```

## Zeta 规约（规约 `let` 表达式）

Zeta 规约是 `let` 表达式规约的一个较华丽名称。具体而言，规约

```
let (x : T) := y; x + 1    ~~>    (y : T) + 1
```

实现可以简单到如下程度：

```
reduce Let _ val body:
  instantiate body val
```

## Delta 规约（定义展开）

Delta 规约指展开定义（以及定理）。Delta 规约会把一个 `const ..` 表达式替换为被引用声明的值，同时把该声明的泛型宇宙参数替换为当前上下文相关的宇宙参数。

如果当前环境包含一个定义 `x`，它以宇宙参数 `u*` 和值 `v` 声明，那么可以对表达式 `Const(x, w*)` 作 delta 规约：先把它替换为 `val`，再把宇宙参数 `u*` 替换为 `w*` 中的那些层级。

```
deltaReduce Const name levels:
  if environment[name] == (d : Declar) && d.val == v then
  substituteLevels (e := v) (ks := d.uparams) (vs := levels)
```

如果为了到达被 delta 规约的 `const` 表达式而不得不移除某些已施加实参，那么这些实参应当重新应用到规约后的定义上。

## 投影规约

`proj` 表达式有一个自然数指标，表示要投影的字段；还有另一个表达式，表示结构本身。构成该结构的实际表达式，应当是一串实参应用到某个引用构造子的 `const` 上。

请记住，对于完全 elaborated 的项，传给构造子的实参会包含任何参数。因此，`Prod` 构造子的一个实例会形如 `Prod.mk A B (a : A) (b : B)`。

表示被投影字段的自然数从 0 开始计数，其中 0 是构造子的第一个*非参数*实参，因为投影不能用于访问结构的参数。

有了这一点就很清楚：一旦我们把构造子的实参去脊为 `(constructor, [arg0, .., argN])`，就可以简单地取位置 `i + num_params` 处的实参来规约投影，其中 `num_params` 正如其名，是该结构类型的参数数量。

```
reduce proj fieldIdx structure:
  let (constructorApp, args) := unfoldApps (whnf structure)
  let num_params := environment[constructorApp].num_params
  args[fieldIdx + numParams]

  -- 沿用 `Prod` 例子，constructorApp 会是 `Const(Prod.mk, [u, v])`
  -- args 会是 `[A, B, a, b]`
```

### 投影的特殊情形：字符串字面量

字符串字面量的内核扩展在投影规约中引入了一个特殊情形，在 iota 规约中也引入了一个特殊情形。

字符串字面量的投影规约：由于投影表达式的结构可能规约为字符串字面量（Lean 的 `String` 类型被定义为一个只有一个字段的结构，该字段是 `List Char`）。

如果该结构规约为 `StringLit (s)`，我们就把它转换为 `String.mk (.. : List Char)`，然后像通常的投影规约一样继续。



## Nat 字面量规约

自然数字面量的内核扩展包括对 `Nat.succ` 的规约，以及加法、减法、乘法、幂、除法、取模、布尔相等和布尔小于等于这些二元操作的规约。

若正在规约的表达式为 `Const(Nat.succ, []) n`，且 `n` 可以规约为自然数字面量 `n'`，则将其规约为 `NatLit(n'+1)`。

若正在规约的表达式为 `Const(Nat.<binop>, []) x y`，且 `x` 和 `y` 可以规约为自然数字面量 `x'` 与 `y'`，则将适当 `<binop>` 的宿主语言版本应用于 `x'` 与 `y'`，并返回所得自然数字面量。


示例：
```
Const(Nat.succ, []) NatLit(100) ~> NatLit(100+1)

Const(Nat.add, []) NatLit(2) NatLit(3) ~> NatLit(2+3)

Const(Nat.add, []) (Const Nat.succ [], NatLit(10)) NatLit(3) ~> NatLit(11+3)
```

## Iota 规约（模式匹配）

Iota 规约是指应用特定于某个归纳声明、并由该归纳声明派生出的规约策略。这里说的是对归纳声明递归器的应用（或者我们稍后会看到的 `Quot` 特殊情形）。

每个递归器都有一组“递归器规则”，每个构造子对应一条递归器规则。与呈现为类型的递归器不同，这些递归器规则是值层级表达式，展示如何消去一个由构造子 `T.c` 创建的类型 `T` 的元素。例如，`Nat.rec` 有一条用于 `Nat.zero` 的递归器规则，还有一条用于 `Nat.succ` 的递归器规则。

对于归纳声明 `T`，`T` 的递归器所要求的元素之一是一个实际的 `(t : T)`，也就是我们正在消去的东西。这个 `(t : T)` 实参称为“主前提”。Iota 规约通过拆开主前提，查看用哪个构造子创建了 `t`，再从环境中取出并应用相应的递归器规则，从而执行模式匹配。

由于递归器的类型签名也要求所需的参数、动机和次前提，因此我们不需要为了在例如 `Nat.zero` 而非 `Nat.succ` 上执行规约而改变递归器的实参。

在实践中，有时需要先做一些初始操作，以暴露创建主前提所用的构造子，因为它可能不是一个直接的构造子应用。例如，`NatLit(n)` 表达式需要被转换为 `Nat.zero`，或 `App Const(Nat.succ, []) ..`。对于结构，我们也可以执行结构 eta 展开，把元素 `(t : T)` 转换为 `T.mk t.1 .. t.N`，从而暴露 `mk` 构造子的应用，使 iota 规约得以继续（如果无法弄清主前提由哪个构造子创建，规约就失败）。

## List.rec 类型

```
forall 
  {α : Type.{u}} 
  {motive : (List.{u} α) -> Sort.{u_1}}, 
  (motive (List.nil.{u} α)) -> 
  (forall (head : α) (tail : List.{u} α), (motive tail) -> (motive (List.cons.{u} α head tail))) -> (forall (t : List.{u} α), motive t)
```

## List.nil 递归器规则

```
fun 
  (α : Type.{u}) 
  (motive : (List.{u} α) -> Sort.{u_1}) 
  (nilCase : motive (List.nil.{u} α)) 
  (consCase : forall (head : α) (tail : List.{u} α), (motive tail) -> (motive (List.cons.{u} α head tail))) => 
  nilCase
```

## List.cons 递归器规则

```
fun 
  (α : Type.{u}) 
  (motive : (List.{u} α) -> Sort.{u_1}) 
  (nilCase : motive (List.nil.{u} α)) 
  (consCase : forall (head : α) (tail : List.{u} α), (motive tail) -> (motive (List.cons.{u} α head tail))) 
  (head : α) 
  (tail : List.{u} α) => 
  consCase head tail (List.rec.{u_1, u} α motive nilCase consCase tail)
```


### 类 k 规约

对于某些归纳类型，即所谓“子单例消去器”，只要我们知道主前提的类型，即使主前提的构造子没有直接暴露，也可以继续执行 iota 规约。例如，当主前提作为自由变量出现时，就可能出现这种情况。这称为类 k 规约；它之所以被允许，是因为子单例消去器的所有元素都是相同的。

要成为子单例消去器，一个归纳声明必须是归纳命题，不能是相互归纳或嵌套归纳；它必须恰好有一个构造子，并且唯一构造子只能接受该类型的参数作为实参（它不能“隐藏”任何未完全由其类型签名捕获的信息）。

例如，类型 `Eq Char 'x'` 的任一元素的值完全由其类型决定，因为该类型的所有元素都是相同的。

如果 iota 规约发现一个主前提是子单例消去器，那么允许把该主前提替换为对该类型构造子的应用，因为这是该自由变量实际可能取到的唯一元素。例如，一个类型为 `Eq Char 'a'` 的自由变量形式主前提，可以被替换为显式构造的 `Eq.refl Char 'a'`。

说到具体机制，如果我们忽略了查找并应用类 k 规约，那么作为子单例消去器的自由变量会无法识别相应的递归器规则，iota 规约会失败，而某些本应成功的转换将不再成功。

### `Quot` 规约；`Quot.ind` 与 `Quot.lift`

`Quot` 引入了两个需要由内核处理的特殊情形：一个用于 `Quot.ind`，一个用于 `Quot.lift`。

`Quot.ind` 与 `Quot.lift` 都处理把函数 `f` 应用于实参 `(a : α)` 的情形，其中 `a` 是某个 `Quot r` 的组成部分，而这个 `Quot r` 由 `Quot.mk r a` 形成。

为了执行规约，我们需要取出作为 `f` 元素的那个实参，以及作为 `Quot` 的那个实参；在后者中可以找到 `(a : α)`。然后把函数 `f` 应用于 `a`。最后，重新应用那些属于某个外层表达式、而与 `Quot.ind` 或 `Quot.lift` 的调用无关的实参。

由于这只是一个规约步骤，我们依赖别处完成的类型检查阶段来保证整个表达式类型良好。

下面重现了 `Quot.ind` 和 `Quot.mk` 的类型签名，并把望远镜元素映射到我们应当在实参中找到的内容。带有 `*` 的元素是规约中我们关心的部分。

```
Quotient primitive Quot.ind.{u} : ∀ {α : Sort u} {r : α → α → Prop} 
  {β : Quot r → Prop}, (∀ (a : α), β (Quot.mk r a)) → ∀ (q : Quot r), β q

  0  |-> {α : Sort u} 
  1  |-> {r : α → α → Prop} 
  2  |-> {β : Quot r → Prop}
  3* |-> (∀ (a : α), β (Quot.mk r a)) 
  4* |-> (q : Quot r)
  ...
```

```
Quotient primitive Quot.lift.{u, v} : {α : Sort u} →
  {r : α → α → Prop} → {β : Sort v} → (f : α → β) → 
  (∀ (a b : α), r a b → f a = f b) → Quot r → β

  0  |-> {α : Sort u}
  1  |-> {r : α → α → Prop} 
  2  |-> {β : Sort v} 
  3* |-> (f : α → β) 
  4  |-> (∀ (a b : α), r a b → f a = f b)
  5* |-> Quot r
  ...
```

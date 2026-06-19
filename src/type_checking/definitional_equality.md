# 定义相等

定义相等被实现为一个函数：它接受两个表达式作为输入；若这两个表达式在 Lean 理论中定义相等，则返回 `true`，否则返回 `false`。

在内核内部，定义相等之所以重要，只是因为它是类型检查的必要组成部分。对于不深入内核的 Lean 用户而言，定义相等仍然是一个重要概念，因为在 Lean 的交互式语言中，定义相等比较容易使用：对于任意定义相等的 `a` 和 `b`，Lean 不需要用户提示或提供额外输入，就能判定两个表达式相等。

实现定义相等过程有两个宏观部分。第一部分是用于检查不同定义相等性的各个具体测试。对于只想从终端用户角度理解定义相等的读者，这大概就是你想知道的内容。

**有意编写类型检查器的读者**还应理解这些具体检查如何同规约和缓存组合起来，使问题可处理；天真地逐个运行检查并在途中规约，可能会得到不可接受的性能结果。关于这一点，我们建议参考已有的内核实现。**这些检查执行的顺序和方式会显著影响性能**；虽然像 Mathlib 这样较大的 Lean 项目可能试图在内核中使用和滥用规约与相等性检查方面保持某种纪律，但代码总会不可避免地依赖某些内核执行路径在 C++ 内核实现中很快或具有某种优势。因此，若用替代策略或更天真的策略检查这些代码库，很可能会造成性能问题。

## 语法相等（也称结构相等或指针相等）

只要类型检查器确保两个对象相等当且仅当它们由相同组件构造而来（相关构造子是 Name、Level 和 Expr 的构造子），那么当两个表达式指向完全相同的实现对象时，它们就是定义相等的。

## Sort 相等

两个 `Sort` 表达式定义相等，当且仅当它们表示的层级根据层级偏序的反对称性相等。

```
defEq (Sort x) (Sort y):
  x ≤ y ∧ y ≤ x
```

## Const 相等

两个 `Const` 表达式定义相等，当且仅当它们的名称相同，并且它们的层级在反对称意义下相等。

```
defEq (Const n xs) (Const m ys):
  n == m ∧ forall (x, y) in (xs, ys), antisymmEq x y

  -- 如果你的 `zip` 不会检查长度，也要断言 xs 和 ys 长度相同。
```

## 有界变量

对于使用基于替换的策略（如局部无名）的实现而言（如果你遵循 C++ 或 lean4lean 实现，这说的就是你），遇到有界变量是一个错误；如果有界变量指向一个实参，那么它应当已经在弱规约期间被替换；或者它应当已经在对 pi 或 lambda 表达式作定义相等检查时，作为强规约的一部分被替换为自由变量。

对于基于闭包的实现，则查找与有界变量对应的元素，并断言它们定义相等。

## 自由变量

两个自由变量定义相等，当且仅当它们具有相同的标识符（唯一 ID 或 de Bruijn 层级）。关于绑定子类型相等的断言，应当已经在构造自由变量的地方执行过（例如 pi/lambda 表达式的定义相等检查），因此现在不必重新检查。

```
defEqFVar (id1, _) (id2, _):
  id1 == id2
```

## App

两个应用表达式定义相等，当且仅当它们的函数部分和实参部分分别定义相等。

```
defEqApp (App f a) (App g b):
  defEq f g && defEq a b
```


## Pi

两个 Pi 表达式定义相等，当且仅当它们的绑定子类型定义相等，并且在替换进适当的自由变量后，它们的函数体类型定义相等。

```
defEq (Pi s a) (Pi t b)
  if defEq s.type t.type
  then
    let thisFvar := fvar s
    defEq (inst a thisFvar) (inst b thisFvar)
  else
    false
```

## Lambda

Lambda 使用与 Pi 相同的测试：

```
defEq (Lambda s a) (Lambda t b)
  if defEq s.type t.type
  then
    let thisFvar := fvar s
    defEq (inst a thisFvar) (inst b thisFvar)
  else
    false
```

## 结构 eta

若两个元素 `x` 和 `y` 都是某个结构类型的实例，并且字段按照下面的过程定义相等，Lean 就承认它们定义相等。该过程比较一边的构造子实参与另一边的投影字段：

```
defEqEtaStruct x y:
  let (yF, yArgs) := unfoldApps y
  if 
    yF is a constructor for an inductive type `T` 
    && `T` can be a struct
    && yArgs.len == T.numParams + T.numFields
    && defEq (infer x) (infer y)
  then
    forall i in 0..t.numFields, defEq Proj(i+T.numParams, x) yArgs[i+T.numParams]

    -- 我们把 `T.numParams` 加到索引上，
    -- 因为只想测试非参数实参。
    -- 参数已经因推断类型定义相等而已知 defEq。
```

更普通的同余情形 `T.mk a .. N` = `T.mk x .. M`，其中 `[a, .., N] = [x, .., M]`，则直接由 `App` 测试处理。

## 类 unit 相等

Lean 在下列条件下承认两个元素 `x: S p_0 .. p_N` 和 `y: T p_0 .. p_M` 定义相等：

+ `S` 是一个归纳类型；
+ `S` 没有指标；
+ `S` 只有一个构造子，且该构造子除 `S` 的参数 `p_0 .. p_N` 外不接受任何实参；
+ 类型 `S p_0 .. p_N` 与 `T p0 .. p_M` 定义相等。

直观地说，这种定义相等是合理的，因为这些类型的元素所能传达的所有信息都已由它们的类型捕获，而我们要求这些类型定义相等。

## Eta 展开

```
defEqEtaExpansion x y : bool :=
  match x, (whnf $ infer y) with
  | Lambda .., Pi binder _ => defEq x (App (Lambda binder (Var 0)) y)
  | _, _ => false
```

右侧创建的 lambda，即 `(fun _ => $0) y`，会平凡地规约为 `y`；但加入 lambda 绑定子让 `x` 和 `y'` 有机会同定义相等过程的其余部分匹配。

## 证明无关相等

Lean 把证明无关相等视为定义相等。例如，Lean 的定义相等过程会把 `2 + 2 = 4` 的任意两个证明都视为定义相等的表达式。

若某个类型 `T` 推断为 `Sort 0`，我们就知道它是一个证明，因为它是 `Prop` 的一个元素（记住，`Prop` 是 `Sort 0`）。

```
defEqByProofIrrelevance p q :
  infer(p) == S ∧ 
  infer(q) == T ∧
  infer(S) == Sort(0) ∧
  infer(T) == Sort(0) ∧
  defEq(S, T)
```

如果 `p` 是类型 `A` 的证明，`q` 是类型 `B` 的证明，那么只要 `A` 与 `B` 定义相等，`p` 与 `q` 就因证明无关性而定义相等。

## 自然数（自然数字面量）

两个自然数字面量定义相等，当且仅当它们可以被规约为 `Nat.zero` 和/或 `NatLit 0`；或者它们可以被规约为 `Nat.succ x`（或 `NatLit x+1`）以及 `Nat.succ y`（或 `NatLit y+1`），其中 `x` 和 `y` 定义相等。

我们预期多数实现会把相同的自然数字面量视为语法相等。也就是说，当 `n` 和 `m` 的值相等时，实现语言会原生地认为 `NatLit n` 与 `Natlit m` 相等。

实践中，这个检查只会在语法相等测试之后执行；在后继情形中，每次对 `defEq` 的递归调用也都应以语法相等检查开始。

```
def isNatZero n:
  match n with
  | Nat.zero | NatLit 0 => true
  | _ => false
  
def isNatSuccOf n:
  match n with
  | .app Nat.succ n' => some n'
  | NatLit n+1 => some (NatLit n) 
  | _ => none

def defEqNatLit a b:
  if isNatZero a && isNatZero b
  then some true
  else 
    match isNatSuccOf a, isNatSuccOf b with
    | some a', some b' => some (defEq a' b') 
    | _, _ => none
```

## 字符串字面量

`StringLit(s), App(String.mk, a)`

字符串字面量 `s` 会被转换为 `Const(String.mk, [])` 对某个 `List Char` 的应用。由于 Lean 的 `Char` 类型用于表示 Unicode 标量值，这些值的整数表示是 32 位无符号整数。

举例来说，字符串字面量 "ok" 使用两个字符，分别对应 32 位无符号整数 `111` 和 `107`，因此会被转换为：

>(String.mk (((List.cons Char) (Char.ofNat.[] NatLit(111))) (((List.cons Char) (Char.ofNat NatLit(107))) (List.nil Char))))

## 惰性 delta 规约与同余

可用的内核实现会在定义相等检查中实现一种“惰性 delta 规约”过程：它使用[可规约性提示](../declarations/declarations.md#可规约性提示)惰性地展开定义，并在看起来有希望时检查同余。这比在检查定义相等之前急切地完全规约两个表达式高效得多。

如果有两个表达式 `a` 和 `b`，其中 `a` 是高度为 10 的定义的一个应用，而 `b` 是高度为 12 的定义的一个应用，那么惰性 delta 过程会采取更高效的路线：展开 `b`，试图让它更接近 `a`；而不是把二者都完全展开，或盲目选择一边展开。

如果惰性 delta 过程发现两个表达式都是某个 `const` 表达式对若干实参的应用，并且这些 `const` 表达式指向同一声明，则会检查这些表达式是否同余（即它们是否是同一个 const 应用于定义相等的实参）。同余失败会被缓存；对于编写自己内核的读者，缓存这些失败是一项性能关键的优化，因为同余检查涉及一次可能代价高昂的 `def_eq_args` 调用。

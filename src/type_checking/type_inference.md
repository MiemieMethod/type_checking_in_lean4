# 类型推断

类型推断是确定给定表达式之类型的过程，也是 Lean 内核的核心功能之一。我们正是通过类型推断确定 `Nat.zero` 是类型 `Nat` 的一个元素，或者 `(fun (x : Char) => var(0))` 是类型 `Char -> Char` 的一个元素。

本节先考察最简单的完整类型推断过程，再考察各过程性能更好、但略微更复杂的版本。

我们还会看到 Lean 内核在类型推断期间作出的若干额外正确性断言。

## 有界变量

如果你遵循 Lean 的实现并使用局部无名（locally nameless）方法，那么在类型推断期间不应遇到有界变量，因为所有打开的绑定子都会用适当的自由变量实例化。

当遇到绑定子时，我们需要遍历进入其函数体，以确定函数体的类型。为了保留关于绑定子类型的信息，有两种主要做法。Lean 本身使用的是：创建一个保留绑定子类型信息的自由变量，使用实例化把相应有界变量替换为这个自由变量，然后进入函数体。这样做很好，因为我们不必为类型上下文额外维护一份状态。

对于基于闭包的实现，通常会有一个单独的类型上下文，用来记录打开的绑定子；此时遇到有界变量，就意味着要索引进入类型上下文，以取得该变量的类型。

## 自由变量

创建自由变量时，它会得到所表示绑定子的类型信息，因此我们可以直接把该类型信息作为推断结果。

```
infer FVar id binder:
  binder.type
```

## 函数应用

```
infer App(f, arg):
  match (whnf $ infer f) with
  | Pi binder body => 
    assert! defEq(binder.type, infer arg)
    instantiate(body, arg)
  | _ => error
```

这里需要的额外断言是：`arg` 的类型与 `binder` 的类型相匹配。例如，在表达式

`(fun (n : Nat) => 2 * n) 10`

中，我们需要断言 `defEq(Nat, infer(10))`。

虽然现有实现倾向于在线执行这一检查，但原则上也可以把这个相等性断言存储起来，在别处处理。

## Lambda

```
infer Lambda(binder, body):
  assert! infersAsSort(binder.type)
  let binderFvar := fvar(binder)
  let bodyType := infer $ instantiate(body, binderFVar)
  Pi binder (abstract bodyType binderFVar)
```

## Pi

```
infer Pi binder body:
  let l := inferSortOf binder
  let r := inferSortOf $ instantiate body (fvar(binder))
  imax(l, r)

inferSortOf e:
  match (whnf (infer e)) with
  | sort level => level
  | _ => error
```

## Sort

任意 `Sort n` 的类型就是 `Sort (n+1)`。

```
infer Sort level:
  Sort (succ level)
```

## Const

`const` 表达式用于按名称引用其他声明；被引用的任何其他声明都必须已经事先声明并经过类型检查。因此，我们已经知道被引用声明的类型，只需在环境中查找它即可。不过，我们确实必须把当前声明的宇宙层级替换进被索引定义的宇宙参数中。

```
infer Const name levels:
  let knownType := environment[name].type
  substituteLevels (e := knownType) (ks := knownType.uparams) (vs := levels)
```

## Let

```
infer Let binder val body:
  assert! inferSortOf binder
  assert! defEq(infer(val), binder.type)
  infer (instantiate body val)
```

## Proj

我们试图推断类似 `Proj (projIdx := 0) (structure := Prod.mk A B (a : A) (b : B))` 这样的东西的类型。

首先推断所给结构的类型；由此可以得到结构名称，并在环境中查找该结构及其构造子类型。

遍历构造子类型的望远镜，把 `Prod.mk` 的参数替换进构造子类型的望远镜中。若查得构造子类型为 `A -> B -> (a : A) -> (b : B) -> Prod A B`，则替换 A 和 B，留下望远镜 `(a : A) -> (b : B) -> Prod A B`。

构造子望远镜剩余的部分表示结构字段，其类型信息位于绑定子中，因此可以检查 `telescope[projIdx]` 并取出绑定子类型。不过还需要处理一件事：由于靠后的结构字段可能依赖靠前的结构字段，我们需要用 `proj thisFieldIdx s` 来实例化望远镜的其余部分（每一阶段的函数体），其中 `s` 是我们正在推断的投影表达式中的原始结构。

```
infer Projection(projIdx, structure):
  let structType := whnf (infer structure)
  let (const structTyName levels) tyArgs := structType.unfoldApps
  let InductiveInfo := env[structTyName]
  -- 这个归纳类型既然声称是结构，就应当只有一个构造子。
  let ConstructorInfo := env[InductiveInfo.constructorNames[0]]

  let mut constructorType := substLevels ConstructorInfo.type (newLevels := levels)

  for tyArg in tyArgs.take constructorType.numParams
    match (whnf constructorType) with
      | pi _ body => inst body tyArg
      | _ => error

  for i in [0:projIdx]
    match (whnf constructorType) with
      | pi _ body => inst body (proj i structure)
      | _ => error

  match (whnf constructorType) with
    | pi binder _=> binder.type
    | _ => error 
```

## 自然数字面量

自然数字面量推断为指向声明 `Nat` 的常量。

```
infer NatLiteral _:
  Const(Nat, [])
```

## 字符串字面量

字符串字面量推断为指向声明 `String` 的常量。

```
infer StringLiteral _:
  Const(String, [])
```

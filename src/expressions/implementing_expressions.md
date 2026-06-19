# 实现表达式

对于有意全部或部分编写自己内核的读者，表达式类型有几个值得注意的要点……

## 存储的数据

表达式需要在内部存储一些数据，或在某处缓存这些数据，以避免代价过高的重复计算。例如，创建表达式 `app x y` 时，需要计算并存储所得 app 表达式的哈希摘要；而这样做应当通过获取 `x` 和 `y` 已缓存的哈希摘要来完成，而不是遍历 `x` 的整棵树递归计算 `x` 的摘要，再对 `y` 做同样的事。

你大概会希望内联存储的数据包括：哈希摘要、表达式中的松散有界变量数量，以及表达式是否含有自由变量。后两者分别有助于优化实例化和抽象。

一个用于 `app` 表达式的“智能构造器”示例如下：

```
def mkApp x y:
  let hash := hash x.storedHash y.storedHash
  let numLooseBvars := max x.numLooseBvars y.numLooseBvars
  let hasFvars := x.hasFvars || y.hasFvars
  .app x y (cachedData := hash numLooseBVars hasFVars)
```

## 不作深拷贝

表达式的实现应当使得用于构造某个父表达式的子表达式不会被深拷贝。换言之，创建表达式 `app x y` 不应递归复制 `x` 和 `y` 的元素，而应取得某种引用：可以是指针、整数索引、对垃圾回收对象的引用、引用计数对象，或其他形式（这些策略都应能提供可接受的性能）。如果你构造表达式的默认策略涉及深拷贝，那么在不消耗巨量内存的情况下，将无法构造任何非平凡环境。

## 松散有界变量数量的示例实现


```
numLooseBVars e:
    match e with
    | Sort | Const | FVar | StringLit | NatLit => 0
    | Var dbjIdx => dbjIdx + 1,
    | App fun arg => max fun.numLooseBvars arg.numLooseBvars
    | Pi binder body | Lambda binder body => 
    |   max binder.numLooseBVars (body.numLooseBVars - 1)
    | Let binder val body =>
    |   max (max binder.numLooseBvars val.numLooseBvars) (body.numLooseBvars - 1)
    | Proj _ _ structure => structure.numLooseBvars
```

对于 `Var` 表达式，松散有界变量的数量是 de Bruijn 指标加一，因为我们计算的是为了使该变量不再松散而需要放在该变量上方的绑定子数量（这里的 `+1` 是因为 de Bruijn 指标从 0 开始）。对于表达式 `Var(0)`，需要在有界变量上方放置一个绑定子，才能使该变量不再松散。对于 `Var(3)`，则需要四个：

```
--  3 2 1 0
fun a b c d => Var(3)
```

当我们创建一个新的绑定子（lambda、pi 或 let）时，可以从函数体中的松散 bvar 数量中减去 1（使用饱和/自然数减法），因为函数体现在位于一个额外的绑定子之下。

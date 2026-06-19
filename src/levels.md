# 宇宙层级

本节将从实现角度描述宇宙层级，并说明读者在对 Lean 声明进行类型检查时需要知道的内容。关于宇宙层级在 Lean 类型论中的作用，更深入的讨论可见 [TPIL4](https://lean-lang.org/theorem_proving_in_lean4/dependent_type_theory.html#types-as-objects)，或 [Mario Carneiro 的博士论文](https://github.com/digama0/lean-type-theory)第 2 节。

宇宙层级的语法如下：

```
Level ::= Zero | Succ Level | Max Level Level | IMax Level Level | Param Name
```

读者应当注意 `Level` 类型的几个性质：宇宙层级上存在一个偏序；其中存在变量（`Param` 构造子）；以及 `Max` 与 `IMax` 之间的区别。

`Max` 只是构造一个表示左右两个参数中较大者的宇宙层级。例如，`Max(1, 2)` 化简为 `2`，而 `Max(u, u+1)` 化简为 `u+1`。`IMax` 构造子也表示左右两个参数中的较大者，*除非*右参数化简为 `Zero`；在这种情况下，整个 `IMax` 解析为 `0`。

`IMax` 的重要之处在于它与类型推断过程的相互作用。例如，它确保 `forall (x y : Sort 3), Nat` 被推断为 `Sort 4`，而 `forall (x y : Sort 3), True` 被推断为 `Prop`。

## 层级上的偏序

Lean 的 `Level` 类型带有一个偏序，也就是说，我们可以对一对层级执行“小于等于”测试。下面这个相当漂亮的实现来自 Gabriel Ebner 的 Lean 3 检查器 [trepplein](https://github.com/gebner/trepplein/tree/master)。虽然需要覆盖的情形不少，但唯一复杂的匹配是依赖 `cases` 的那些；`cases` 通过分别检查把参数 `p` 替换为 `Zero` 时，以及把 `p` 替换为 `Succ p` 时，`x ≤ y` 是否成立，来判断 `x ≤ y`。

```
  leq (x y : Level) (balance : Integer): bool :=
    Zero, _ if balance >= 0 => true
    _, Zero if balance < 0 => false
    Param(i), Param(j) => i == j && balance >= 0
    Param(_), Zero => false
    Zero, Param(_) => balance >= 0
    Succ(l1_), _ => leq l1_ l2 (balance - 1)
    _, Succ(l2_) => leq l1 l2_ (balance + 1)

    -- 向左下降
    Max(a, b), _ => (leq a l2 balance) && (leq b l2 balance)

    -- 向右下降
    (Param(_) | Zero), Max(a, b) => (leq l1 a balance) || (leq l1 b balance)

    -- imax
    IMax(_, p @ Param(_)), _ => cases(p)
    _, IMax(_, p @ Param(_)) => cases(p)
    IMax(a1, b1), IMax(a2, b2) if a1 == a2 && b1 == b2 && balance >= 0 => true
    IMax(a, IMax(b, c)), _ => leq Max(IMax(a, c), IMax(b, c)) l2 balance
    IMax(a, Max(b, c)), _ => leq (simplify Max(IMax(a, b), IMax(a, c))) l2 balance
    _, IMax(a, IMax(b, c)) => leq l1 Max(IMax(a, c), IMax(b, c)) balance
    _, IMax(a, Max(b, c)) => leq l1 (simplify Max(IMax(a, b), IMax(a, c))) balance


  cases l1 l2 p: bool :=
    leq (simplify $ subst l1 p zero) (simplify $ subst l2 p zero)
    ∧
    leq (simplify $ subst l1 p (Succ p)) (simplify $ subst l2 p (Succ p))
```

## 层级相等

`Level` 类型通过反对称性识别相等：两个层级 `l1` 和 `l2` 相等，当且仅当 `l1 ≤ l2` 且 `l2 ≤ l1`。

# 实现说明

注意，导出器不会导出 `Zero`；它被假定为 `Level` 的第 0 个元素。

顺便说一句，`Level` 的实现对性能没有很大影响，因此不必在这里过度优化。

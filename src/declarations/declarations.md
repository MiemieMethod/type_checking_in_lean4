# 声明

声明是最重要的对象，也是我们还需要定义的最后一类领域元素。


```
declarationInfo ::= Name, (universe params : List Level), (type : Expr)

declar ::= 
  Axiom declarationInfo
  | Definition declarationInfo (value : Expr) ReducibilityHint
  | Theorem declarationInfo (value : Expr) 
  | Opaque declarationInfo (value : Expr) 
  | Quot declarationInfo
  | InductiveType 
      declarationInfo
      is_recursive: Bool
      num_params: Nat
      num_indices: Nat
      -- 此类型以及同一 mutual 块中任何其他类型的名称
      allIndNames: Name+
      -- 仅属于“此”类型的构造子名称，
      -- 不包括同一块中 mutual 类型的构造子。
      constructorNames: Name*
      
  | Constructor 
      declarationInfo 
      (inductiveName : Name) 
      (numParams : Nat) 
      (numFields : Nat)

  | Recursor 
        declarationInfo 
        numParams : Nat
        numIndices : Nat
        numMotives : Nat
        numMinors : Nat
        RecRule+
        isK : Bool

RecRule ::= (constructor name : Name), (number of constructor args : Nat), (val : Expr)
```


## 检查声明


对于所有声明，在执行任何针对特定声明种类的额外过程之前，都会先进行下列初步检查：

+ 声明的 `declarationInfo` 中的宇宙参数不得有重复。例如，声明 `def Foo.{u, v, u} ...` 会被禁止。

+ 声明的类型不得含有自由变量；“完成”的声明中的所有变量都必须对应于某个绑定子。

+ 声明的类型必须是一个类型（`infer declarationInfo.type` 必须产生一个 `Sort`）。在 Lean 中，不允许声明 `def Foo : Nat.succ := ..`；`Nat.succ` 是值，不是类型。

### 公理

对公理所做的检查，只有适用于所有声明的那些检查，即确保 `declarationInfo` 合格。如果一个公理具有有效的一组宇宙参数，并且具有一个没有自由变量的有效类型，它就会被接纳进环境。

### Quot

`Quot` 声明包括 `Quot`、`Quot.mk`、`Quot.ind` 和 `Quot.lift`。这些声明具有规定好的类型，并且这些类型在 Lean 理论中已知是可靠的；因此环境中的商类型声明必须与这些类型完全匹配。由于这些类型并不复杂到不可接受，内核实现通常会将它们硬编码。

### 定义、定理、不透明声明

定义、定理和不透明声明的有趣之处在于，它们同时具有类型和值。检查这些声明时，要先对声明的值推断出一个类型，然后断言该推断类型与 `declarationInfo` 中标注的类型定义相等。

在定理的情形中，`declarationInfo` 中的类型是用户声称的类型，因此也是用户声称要证明的命题；而值则是用户为该类型提供的证明。对收到的值推断类型，相当于检查该证明实际上证明了什么；定义相等断言则确保该值所证明的东西，确实是用户打算证明的东西。

#### 可规约性提示

可规约性提示包含关于某个声明应如何展开的信息。`abbreviation` 通常总是会被展开，`opaque` 不会被展开，而 `regular N` 可能会根据 `N` 的值被展开。`regular` 可规约性提示对应于定义的“高度”，也就是该定义用来定义自身所使用的声明数量。若定义 `x` 的值引用了定义 `y`，则 `x` 的高度值会大于 `y`。

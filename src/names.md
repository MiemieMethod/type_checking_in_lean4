# 名称

内核的第一个原始类型是 `Name`，大体上正如其名；它为内核项提供一种寻址事物的方式。

```
Name ::= anonymous | str Name String | num Name Nat
```

`Name` 类型的元素会显示为以点分隔的名称，Lean 用户大概对此很熟悉。例如，`num (str (anonymous) "foo") 7` 会显示为 `foo.7`。

# 实现说明

名称的实现假定字符串为 UTF-8，字符为 Unicode 标量值（关于实现语言字符串类型的这些假设，对于字符串字面量内核扩展也很重要）。

关于名称词法结构的一些信息见[此处](https://github.com/leanprover/lean4/blob/504b6dc93f46785ccddb8c5ff4a8df5be513d887/doc/lexical_structure.md?plain=1#L40)。

导出器不会显式输出匿名名称，而是期望它是导入名称中的第 0 个元素。

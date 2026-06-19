# 导出格式

导出器是一个程序，它使用内核语言发出 Lean 声明，以供外部类型检查器使用。产生导出文件，意味着完全离开 Lean 生态：文件中的数据可以由完全外部的软件检查，而导出器本身并不是可信组件。与其直接检查导出文件本身，以判断声明是否按照开发者意图被导出，不如由外部检查器检查这些导出的声明，再由漂亮打印器把它们显示给用户；漂亮打印器产生的输出远比导出文件本身更可读。读者可以（也被鼓励）为 Lean 导出文件编写自己的外部检查器。

官方导出器是 [lean4export](https://github.com/leanprover/lean4export)。

当前版本的 lean4export 使用 ndjson 格式，其规范见[此处](https://github.com/leanprover/lean4export/blob/master/format_ndjson.md)。

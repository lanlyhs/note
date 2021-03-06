# unicode

当一个 string 类型的值被转换为 []rune 类型值的时候，其中的字符串会被拆分成一个一个的 Unicode 字符。

Go 语言的源码文件必须使用 UTF-8 编码格式进行存储。如果源码文件中出现了非 UTF-8 编码的字符，那么在构建、安装以及运行的时候，go 命令就会报告错误 “illegal UTF-8 encoding”。

## 问题：一个 string 类型的值在底层是怎样被表达的？

是在底层，一个 string 类型的值是由一系列相对应的 Unicode 代码点的 UTF-8 编码值来表达的。

在 Go 语言中，一个 string 类型的值既可以被拆分为一个包含多个字符的序列，也可以被拆分为一个包含多个字节的序列。前者可以由一个以 rune 为元素类型的切片来表示，而后者则可以由一个以 byte 为元素类型的切片代表。

rune 是 Go 语言特有的一个基本数据类型，它的一个值就代表一个字符，即：一个 Unicode 字符。我们已经知道，UTF-8 编码方案会把一个 Unicode 字符编码为一个长度在 [1, 4] 范围内的字节序列。所以，一个 rune 类型的值也可以由一个或多个字节来代表。根据 rune 类型的声明可知，它实际上就是 int32 类型的一个别名类型。也就是说，一个 rune 类型的值会由四个字节宽度的空间来存储。它的存储空间总是能够存下一个 UTF-8 编码值。

一个 string 类型的值会由若干个 Unicode 字符组成，每个 Unicode 字符都可以由一个 rune 类型的值来承载。

这些字符在底层都会被转换为 UTF-8 编码值，而这些 UTF-8 编码值又会以字节序列的形式表达和存储。因此，一个 string 类型的值在底层就是一个能够表达若干个 UTF-8 编码值的字节序列。

## 问题 1：使用带有 range 子句的 for 语句遍历字符串值的时候应该注意什么？

带有 range 子句的 for 语句会先把被遍历的字符串值拆成一个字节序列，然后再试图找出这个字节序列中包含的每一个 UTF-8 编码值，或者说每一个 Unicode 字符。

这样的 for 语句可以为两个迭代变量赋值。如果存在两个迭代变量，那么赋给第一个变量的值，就将会是当前字节序列中的某个 UTF-8 编码值的第一个字节所对应的那个索引值。

而赋给第二个变量的值，则是这个 UTF-8 编码值代表的那个 Unicode 字符，其类型会是 rune。

由此可以看出，这样的 for 语句可以逐一地迭代出字符串值里的每个 Unicode 字符。但是，相邻的 Unicode 字符的索引值并不一定是连续的。这取决于前一个 Unicode 字符是否为单字节字符。

正因为如此，如果我们想得到其中某个 Unicode 字符对应的 UTF-8 编码值的宽度，就可以用下一个字符的索引值减去当前字符的索引值。

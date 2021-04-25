原文：https://www.ardanlabs.com/blog/2020/07/extending-python-with-go.html （译：[@xwjdsh](https://github.com/xwjdsh)）

---

## Python 和 Go：第二部分 - 使用 Go 扩展 Python

### 引言

在 [上一篇文章 ]()中，我们了解了 Go 服务如何使用 gRPC 调用 Python 服务。使用 gRPC 将 Go 和 Python 程序连接在一起可能是一个不错的选择，但随之而来的是复杂性。你需要管理一项服务，部署会变得更加复杂，并且你需要为每项服务进行监视和警报。与一体的应用程序相比，复杂度高了一个数量级。

在本文中，我们将通过在 Go 中编写一个 Python 程序可以直接使用的共享库来降低使用 gRPC 的复杂性。使用这种方法，无需进行任何网络连接，并且取决于数据类型，也无需序列化。在使用 Python 共享库调用函数的几种方法中，我们决定使用 Python 的 [ctypes](https://docs.python.org/3/library/ctypes.html) 包。

*注意：ctypes在底层使用 [libffi](https://github.com/libffi/libffi)。如果您想阅读一些真正令人恐惧的 C 代码 - 前往仓库阅读。*

我还将展示我的工作流程，这是提高工作效率的重要因素之一。我们将首先编写“纯” Go 代码，然后编写代码将其导出到共享库（动态链接库）。然后，我们将切换到 Python，并使用 Python 的交互式提示来调试代码。一切顺利后，我们将使用在互动提示中学到的知识来编写 Python 包。

### 示例：并行检查多个文件的数字签名

假设你有一个包含数据文件的目录，并且需要验证这些文件的完整性。该目录包含一个 `sha1sum.txt` 文件，其中每个文件都有一个 [sha1](https://en.wikipedia.org/wiki/SHA-1) 数字签名。Go 具有并发原语和使用计算机所有核心的能力，因此比 Python 更适合此任务。

```
6659cb84ab403dc85962fc77b9156924bbbaab2c  httpd-00.log
5693325790ee53629d6ed3264760c4463a3615ee  httpd-01.log
fce486edf5251951c7b92a3d5098ea6400bfd63f  httpd-02.log
b5b04eb809e9c737dbb5de76576019e9db1958fd  httpd-03.log
ff0e3f644371d0fbce954dace6f678f9f77c3e08  httpd-04.log
c154b2aa27122c07da77b85165036906fb6cbc3c  httpd-05.log
28fccd72fb6fe88e1665a15df397c1d207de94ef  httpd-06.log
86ed10cd87ac6f9fb62f6c29e82365c614089ae8  httpd-07.log
feaf526473cb2887781f4904bd26f021a91ee9eb  httpd-08.log
330d03af58919dd12b32804d9742b55c7ed16038  httpd-09.log
```

以上显示了数字签名文件的示例。它为目录中包含的所有不同日志文件提供哈希码。此文件可用于验证日志文件是否已正确下载或未被篡改。我们将编写 Go 代码来计算每个日志文件的哈希码，然后将其与数字签名文件中列出的哈希码进行对比。

为了加快此过程，我们将在单独的 goroutine 中计算每个文件的数字签名，将工作分散到计算机上的所有 CPU 中。

### 结构概述和工作计划

在 Python 端，我们将编写一个名为 `check_signatures` 的函数，在 Go 端，我们将编写一个名为 `CheckSignatures` 的函数（做实际计算）。在这两个函数之间，我们将使用 `ctypes` 包（在Python端）并编写一个 `verify` 函数（在Go端）以提供封送处理支持。

![](https://www.ardanlabs.com/images/goinggo/122_figure1.png)

上图显示了从 Python 函数到 Go 函数然后再返回的数据流。

以下是本文剩余部分将要遵循的步骤：

* 编写Go代码（CheckSignature）
* 导出到共享库（verify）
* 在 Python 交互式提示中使用 ctypes 调用 Go 代码
* 编写并打包 Python 代码（check_signatures）
* 留到下一篇文章中介绍（这一篇文章已经足够长了）

### 实现 Go 的 CheckSignatures 函数

我不会在这里分解所有 Go 源代码，如果你想查看所有这些源代码，请在 [这里](https://github.com/ardanlabs/python-go/blob/master/pyext/checksig.go) 查看。

我们来看最为重要的 `CheckSignatures` 函数定义。

```go
// CheckSignatures calculates sha1 signatures for files in rootDir and compare
// them with signatures found at "sha1sum.txt" in the same directory. It'll
// return an error if one of the signatures don't match
func CheckSignatures(rootDir string) error {
```

以上是 `CheckSignatures` 函数的定义，此函数将为每个文件开启一个 goroutine，以检查计算出的 `sha1`签名是否与  `sha1sum.txt` 中对应文件的签名相匹配。如果一个或多个文件不匹配，该函数将返回错误。

### 将 Go 代码导出为共享库

在完成编写和测试 Go 代码后，我们将其导出为共享库。

我们将按照以下步骤将 Go 源代码编译到共享库中，以便 Python 可以调用它：

* 导入 `C` 包（也叫 [cgo](https://golang.org/cmd/cgo/)）
* 在每个我们需要暴露的函数上使用 `//export` 指令
* 一个空的 `main` 函数
* 使用专门的 `-buildmode = c-shared` 标志来编译源代码

*注意：除了 Go 工具链之外，我们还需要 C 编译器（例如 gcc）。每个主要平台都有一个不错的免费 C 编译器：适用于 Linux 的 gcc，适用于 OSX（通过 XCode）的 clang 和适用于 Windows 的 Visual Studio*

```go
01 package main
02 
03 import "C"
04 
05 //export verify
06 func verify(root *C.char) *C.char {
07 	rootDir := C.GoString(root)
08 	if err := CheckSignatures(rootDir); err != nil {
09 		return C.CString(err.Error())
10 	}
11 
12 	return nil
13 }
14 
15 func main() {}
```

以上显示了 `export.go` 文件内容，我们先在第 03 行导入 `C`，然后在第 05 行把 `verify` 函数标记为要在共享库中导出，这很重要。你可以看到在第 06 行，`verify` 函数使用 `C` 包 `char` 类型接受基于 `C` 的字符串指针。为了使 Go 代码可以使用 C 字符串，C 包提供了 `GoString` 函数（第 07 行）和 `CString` 函数（第 09 行）。最后，声明了一个空的 `main` 函数。

要构建共享库，需要运行带有特殊标志的 `go build` 命令。

```bash
$ go build -buildmode=c-shared -o _checksig.so
```

以上显示了生成基于 `C` 的共享库的命令，命名为 `_checksig.so`。

*注意：命名使用 `_` 的原因是为了避免与稍后将显示的 `checksig.py` Python 模块发生名称冲突。如果共享库名为 `checksig.so`，则在 Python 中执行导入 `checksig` 将加载共享库而不是 Python 文件。*

### 准备测试数据

在我们尝试从 Python 调用 `verify` 之前，我们需要一些数据。你会在代码存储库中找到一个名为 `logs` 的目录。该目录包含一些日志文件和 `sha1sum.txt` 文件。

*注意：`http08.log` 的签名是有意的错误。*

在我的机器上，该目录位于 `/tmp/logs`。

### Python 会话

我喜欢 Python 中的交互式命令行，它使我可以一小段一小段的处理代码。在我得到可用的版本后，我将 Python 代码写入文件中。

```bash
$ python
Python 3.8.3 (default, May 17 2020, 18:15:42) 
[GCC 10.1.0] on linux
Type "help", "copyright", "credits" or "license" for more information.

01 >>> import ctypes
02 >>> so = ctypes.cdll.LoadLibrary('./_checksig.so')
03 >>> verify = so.verify
04 >>> verify.argtypes = [ctypes.c_char_p]
05 >>> verify.restype = ctypes.c_void_p
06 >>> free = so.free
07 >>> free.argtypes = [ctypes.c_void_p]
08 >>> ptr = verify('/tmp/logs'.encode('utf-8'))
09 >>> out = ctypes.string_at(ptr)
10 >>> free(ptr)
11 >>> print(out.decode('utf-8'))
12 "/tmp/logs/httpd-08.log" - mismatch
```

以上显示了一个交互式 Python 会话，它将引导你测试我们导出的在 Go 中编写的共享库的使用。在第 01 行，我们导入 `ctypes` 模块，然后在第 02 行，我们将共享库加载到内存中。在 03-05 行，我们从共享库中加载了 `verify` 函数，并设置了输入和输出类型。 06-07行加载了 `free` 函数，因此我们可以释放 Go 分配的内存（请参见下文）。

第 08 行是要验证的实际函数调用。我们需要先将目录名称转换为 Python 的字节，然后再将其传递给函数。返回值是一个 C 字符串，存储在 `ptr` 中。在第 09 行，我们将 C 字符串转换为 Python 字节，在第 10 行，我们释放了 Go 分配的内存。最后，在第 11 行，我们在打印之前将其从 `bytes` 转换为 `str`。

*注意：第 02 行假定共享库 `_checksig.so` 在当前目录中。如果你在其他地方启动了 `Python` 会话，请将第 02 行更改为 `_checksig.so` 的路径。*

只需很少的努力，我们就可以从 Python 调用 Go 代码。

### 在 Go 和 Python 之间共享内存

Python 和 Go 都具有垃圾收集器，垃圾收集器将自动释放未使用的内存。但是，拥有垃圾收集器并不意味着你不会泄漏内存。

*注意：你应该阅读 Bill 的 [Go 中的垃圾回收 ](https://www.ardanlabs.com/blog/2018/12/garbage-collection-in-go-part1-semantics.html)博客文章。它们将使你对一般的垃圾收集器，特别是 Go 的垃圾收集器有一个很好的了解。*

在 Go 和 Python（或 C）之间共享内存时，你需要格外小心。有时不清楚何时分配内存。在 `export.go` 的第 13 行，我们具有以下代码：

```go
str := C.CString(err.Error())
```

`C.String` 的 [文档](https://golang.org/cmd/cgo/) 说：

```go
> // Go string to C string
> // The C string is allocated in the C heap using malloc.
> // **It is the caller's responsibility to arrange for it to be
> // freed**, such as by calling C.free (be sure to include stdlib.h
> // if C.free is needed).
> func C.CString(string) *C.char
```

为了避免在交互式提示中发生内存泄漏，我们加载了 `free` 函数，并使用它释放了 Go 分配的内存。

### 结语

只需很少的代码，你就可以从 Python 使用 Go。与上一部分不同，它没有 RPC 步骤 - 这意味着你不需要在每个函数调用中都序列化和反序列化参数，并且也不需要任何网络。从 Python 到 C 的调用比 gRPC 调用快得多。另一方面，你需要更加谨慎地进行内存管理，并且打包过程更加复杂。

*注意：在我的计算机上，一个简单的基准测试 gRPC 函数调用为 128µs，共享库调用为 3.61µs，这大约快了35倍。*

希望你已经找到了这种编写代码的方式：先写纯 Go 代码，然后导出为共享库，然后在交互式会话中试用它。希望你在下次编写一些代码时可以尝试此工作流程。

在下一部分中，我们将完成工作流程的最后一步，并将 Python 代码打包为一个模块。

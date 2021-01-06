---
title: "Go和Python的源码安全保护"
date: 2021-01-06T22:15:00+08:00
draft: false
tags: ["Go", "Python", "Security"]
---

## Go代码安全

Go 作为编译型语言，程序会被直接编译成二进制机器码运行，在安全性上面已有一定保障，想要进一步进行保护一般情况下只需从源码和编译配置上入手。

### 源码混淆 (gobfuscate)

工具：[gobfuscate](https://github.com/unixpickle/gobfuscate)

#### 使用

```bash
gobfuscate <package> <output_path>
```

#### 限制

不支持 `Go Mod` 。对使用 `Go Mod` 的项目，可以进行混淆但需要进行一系列繁琐操作。

#### 包名混淆

`gobfuscate` 会将导入的包及其路径的名称进行哈希。

`github.com/unixpickle/deleteme` --> `jiikegpkifenppiphdhi/igijfdokiaecdkihheha/jhiofoppieegdaif`

#### 全局命名混淆

`gobfuscate` 会将全局的 `var` 、 `const` 、 `func` 、 `type` 声明的名称进行哈希。

####  字符串常量混淆

`gobfuscate` 会将字符串常量替换成一个匿名函数，并将所有 `const` 声明转换为 `var` 声明。

```go
const str = "hey"
```

```go
var str = func() string {
		mask := []byte{33, 15, 199}
		maskedStr := []byte{73, 106, 190}
		res := make([]byte, 3)
		for i, m := range mask {
			res[i] = m ^ maskedStr[i]
		}
		return string(res)
	}()
```



### 源码混淆 (garble)

工具：[garble](https://github.com/burrowers/garble)

`garble` 是对 Go 编译工具的封装，在编译前会对代码进行混淆，混淆机制大致和 `gobfuscate` 相同，同时还会禁止部分额外信息输入到二进制文件，支持 `Go Mod` 。

#### 使用

```bash
garble -literals -tiny build <build_flags> <package/main_file>
```

#### 限制

只支持 `Go 1.15` 以上

####  例子

混淆前报错：

![Go运行时报错信息1.png](https://i.loli.net/2021/01/06/fNkvGeuJLaXrmMl.png)

混淆后报错：

![Go运行时报错信息2.png](https://i.loli.net/2021/01/06/4cf3wesbEojtuRO.png)



### 禁用符号表和调试信息

虽然 Go 是编译成二进制后运行的，但其**默认编译机制（Release With Debug Info）**会泄漏一些信息。

![Go运行时报错调试信息.png](https://i.loli.net/2021/01/04/6c7AzZXmsI3PY2w.png)

默认情况下， Go 程序在运行出错时，会输出如上报错信息（在哪个线程，哪个文件，哪个函数，哪行出的错）。

这两个信息可以在编译时进行禁用：

```bash
go build -ldflags "-s -w” <package>
```

```bash
-s    disable symbol table
-w    disable DWARF generation
```



### 隐藏环境变量

报错信息中目录信息的来源是编译器运行时所处环境的环境变量。
上图中的程序运行时，环境变量是这样的：

```bash
GOROOT = C:/Users/Administrator/AppData/Local/Go
GOPATH = D:/Project/Go/shack
GOROOT_FINAL = $GOROOT
```

编译时，Go 会从 `$GOPATH` 寻找我们的代码，从 `$GOROOT` 提取标准库。在打包时将 `$GOPATH` 改写为 `$GOROOT_FINAL` 并作为调试信息的一部分写入目标文件。

要隐藏真实的 `$GOPATH` ，需要在另外一个目录里对真实的 `$GOPATH` 创建一个软链接，编译器在寻找时就会把软链接的目录名写到最终文件里，从而达到隐藏目的。

```bash
 ACTUAL_GOPATH = "~/Project"
 export GOPATH = '/tmp'
 export GOROOT_FINAL = $GOPATH
 [ ! -d $GOPATH ] && ln -s "$ACTUAL_GOPATH" "$GOPATH"
 [[ ! $PATH =~ $GOPATH ]] && export PATH=$PATH:$GOPATH/bin
```



## Python代码安全

Python 作为解释型语言，会先将源码转换为**字节码**（.pyc），然后再由解释器对字节码逐行解释成机器码运行。

Python 字节码可以轻松逆向出源码，安全性不高。虽然在运行时可以通过添加 `-B` 参数，不生成 `.pyc` 文件，但仍然可以从内存中把字节码 dump 出来。

### 源码混淆

工具：[Intensio-Obfuscator](https://github.com/Hnfull/Intensio-Obfuscator)

#### 使用

```bash
python3 intensio_obfuscator.py -i <input_path> -o <output_path> -mlen high -ind 4 -rts -ps -rfn -rth
```

#### 编码混淆

`Intensio-Obfuscator` 会将所有变量、类、函数、文件的命名替换成随机字符串，然后将整个运行过程转换成十六进制表示的字符串。

#### 删除非编码字符

`Intensio-Obfuscator` 会删除所有注释和空行。

#### 填充无用代码

`Intensio-Obfuscator` 会在代码段、函数、类之间随机填充无用代码。

#### 例子

https://github.com/Hnfull/Intensio-Obfuscator/blob/master/src/intensio_obfuscator/obfuscation_examples/python/advanced



### 编译成二进制

工具：[Nuitka](https://github.com/Nuitka/Nuitka)

##### 从主文件递归编译整个项目

```bash
python -m nuitka --remove-output --follow-imports <main_file>
```

##### 编译一个包

```bash
python -m nuitka --remove-output --module --no-pyi-file --include-package=<package> <package>
```

编译后的包（ `.pyd` /  `.so` ）的使用与正常包无异，直接可以使用 `import` 导入。

**注意**：编译后的包文件不能改名



### 隐藏Traceback信息

Python 运行报错时打印的 Trackback 信息也会泄露一些信息，可以使用如下方法隐藏：

```python
import sys
sys.stderr = open("/dev/null", 'w')
```



### PyArmor

[PyArmor](https://github.com/dashingsoft/pyarmor) 是一个用于加密和保护 Python 脚本的工具。它能够在运行时刻保护 Python 脚本的二进制代码不被泄露。它的保障机制主要包括：

- 绑定加密后的Python源代码到硬盘、网卡等硬件设备
- 在脚本运行时候动态加密和解密每一个函数（代码块）的二进制代码
- 代码块执行完成之后清空堆栈局部变量



PyArmor 使用分片式技术来保护 Python 脚本。所谓分片保护，就是单独加密每一个函数， 在运行脚本的时候，只有当前调用的函数被解密，其他函数都处在加密状态。而一旦函数执行完成，就又会重新加密。



PyArmor 的核心代码 `_pytransform` 使用 C 来编写，所有的加密和解密算法都在动态链接库中实现。 首先 `_pytransform` 自身会使用 JIT 技术，即动态生成代码的方式来保护自己，加密后的 Python 脚本由动态库  `_pytransform` 来保护，反过来， 加密的 Python 脚本也会校验动态库，确保其没有进行任何修改。Python 代码和 C 代码的交叉保护，使得用户不能通过修改代码段指令来获得没有授权的使用。例如，将指令 `JZ` 修改为 `JNZ` ，从而使得认证失败可以继续执行。



但是需要收费（￥286），试用版只能加密最多 `3kb` 的文件。



### 定制Python解释器

#### 修改入口函数

Python 解释器在运行 Python 脚本时，会使用 `Moduls/main.c` 中的 `Py_Main()` 来作为入口函数。

```c
// Modules/Main.c
int
Py_Main(int argc, char **argv)
{
    if (command) {
        // 处理 python -c <command>
    } else if (module) {
        // 处理 python -m <module>
    }
    else {
        // 处理 python <file>
        ...
        fp = fopen(filename, "r");
        ...
    }
}
```

所以，我们可以在这个过程加入自己的一些验证逻辑，从而达到加密目的。



#### 修改操作码（opcode）

经过解释器编译后的 Python 字节码中使用一系列**操作码**来表示机器操作。

```python
def add(a, b):
    return a + b
```

`add()` 函数会被翻译成如下字节码：

```bash
  9           0 LOAD_FAST                0 (a)
              2 LOAD_FAST                1 (b)
              4 BINARY_ADD
              6 RETURN_VALUE
```

其中，`LOAD_FAST` 、`BINARY_ADD` 等就是**操作码**。

操作码在解析器的 `opcode.h` 中有定义：

```c
...
#define LOAD_FAST   124
#define STORE_FAST  125
#define DELETE_FAST 126
...
```

我们可以修改操作码的值后再进行编译，这样不能直接用未经修改的 Python 解释器运行，也不能直接用现成的逆向工具来反编译出字节码。



## 加壳

使用 [UPX](https://github.com/upx/upx) 或其他壳对二进制文件进行简单加壳，~~增加逆向难度~~ 防君子。

```bash
upx --brute <binary_file>
```



## 总结

通过将源码编译成二进制运行，可以初步保护源码安全，但仍可以使用工具将其**反编译成汇编/类汇编语言**；

一般情况下，编译器编译时**不会主动对字符串做处理**，所以只从字符串就能获取到很多信息；

同时，如果没有做额外混淆的话，程序中会**保留有大量明文信息**（函数名、类名、包名、环境变量等等）。

针对后面两点，我们可以使用上述工具进行混淆处理。但因为第一点的存在，绝对的安全是无法做到的。

从另一个角度来考虑，如果有人能获取到编译后的二进制信息，说明运行环境已经不安全了，这时候要担心的就不是代码安全问题了。
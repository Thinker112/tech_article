# Shebang行

原文：[Shebang 行](https://linuskarlsson.se/blog/shebang-shenanigans/)

你们可能都在shell脚本顶部见过shebang行，第一行都是以`#!/bin/sh`开头。

开头的字符`#!`告诉操作系统这个文件不是一个标准的二进制文件，而是一个需要通过解释器运行的脚本文件。解释器的名称就是`#!`后面的`/bin/sh`。因此你可以看到像 `#!/usr/bin/perl`,  `#!/usr/bin/awk` , `#!/usr/bin/python`

执行一个像`./test.sh`这样的文件，使用shebang行`#!/bin/sh`，类似于调用命令：`/bin/sh ./test.sh`

在学习awk的过程中，我注意到了shebang行 `#!/usr/bin/awk -f`的使用，即带有额外参数的shebang。在python文件中有时也会看到，通常为`#!/usr/bin/python -u` 以启用无缓冲输出。

当试着在awk中添加一些额外的参数，理想情况下将其作为`#!/usr/bin/awk -i inplace -f`运行，以直接修改源文件而不是将结果输出到标准输出。但是并没有起作用。shebang行等同于命令：`awk "-i inplace -f" file.awk`，即用一个参数调用`awk`，所有的参数项都合并到一个字符串中。这肯定不是有效的方法。这让我产生了一个问题：**不同的类 Unix 系统如何处理 shebang 参数？**

## 辅助程序

为了轻松查看参数是如何传递给二进制文件的，用C语言编写了一个小型辅助应用程序。传递进来的每个参数都会打印一行。

```c
#include <stdio.h>

int main(int argc, char **argv)
{
	for (int i = 0; i < argc; ++i) {
		printf("argv[%d]: %s\n", i, argv[i]);
	}

	return 0;
}

```

运行`./args hi there reader` 命令，将会输出：

```bash
$ ./args hi there reader
argv[0]: ./args
argv[1]: hi
argv[2]: there
argv[3]: reader

```

将`./args`文件复制到`/usr/local/bin/args`，然后继续创建以下测试文件，使用`chmod +x file.txt`命令将其变为可执行文件。

```shell
#!/usr/local/bin/args -a -b --something

hello i'm a line that doesn't matter
```

然后在不同的操作系统上执行file.txt文件。

## Linux

```bash
$ ./file.txt
argv[0]: /usr/local/bin/args
argv[1]: -a -b --something
argv[2]: ./file.txt
```

可以看到`argv[1]` 将所有的命令参数作为一个单独的参数了。

## FreeBSD / OpenBSD

```bash
$ ./file.txt
argv[0]: /usr/local/bin/args
argv[1]: -a -b --something
argv[2]: ./file.txt
```

## macOS

macOS会把每个参数都独立地传递给解释器。

```bash
$ ./file.txt
argv[0]: /Users/linus/args
argv[1]: -a
argv[2]: -b
argv[3]: --something
argv[4]: ./file.txt
```



## 总结

|                 | argv[0]               | argv[1]             | argv[2]      | argv[3]       | argv[4]      |
| --------------- | --------------------- | ------------------- | ------------ | ------------- | ------------ |
| Linux           | `/usr/local/bin/args` | `-a -b --something` | `./file.txt` |               |              |
| FreeBSD/OpenBSD | `/usr/local/bin/args` | `-a -b --something` | `./file.txt` |               |              |
| macOS           | `/usr/local/bin/args` | `-a`                | `-b`         | `--something` | `./file.txt` |
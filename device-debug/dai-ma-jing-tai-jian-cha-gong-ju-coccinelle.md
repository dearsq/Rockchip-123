# Coccinelle 代码静态检查工具
## 介绍
Coccinelle是一个程序匹配和转换引擎，它使用 SmPL 语言（语义包语言）来为 C 代码提供匹配和转换代码的功能。

它的主要功能是完成 Linux 的 “Collateral Evolutions” 功能，即 API 更新功能。另外还可以用它来查找和修复系统代码中的错误。

举个例子：foo(int) 函数突然变成 foo(int, char *) 函数，多出了一个输入参数（可以把第二个参数置为 null）。所有调用 foo() 函数的代码都需要更新了，这可能是个悲摧的体力活。但是使用 Coccinelle 的话，这项工作瞬间变得轻松，脚本会帮你找到调用 foo(parameter1) 的代码，然后替换成 foo(parameter1, NULL)。做完这些后，所有调用这个函数的代码都可以运行一遍，验证下第二个参数为 NULL 是否能正常工作。


## 实例
检查 video/rockchip/bmp_helper 模块：

```
sudo apt-get install coccinelle
cd kernel
make coccicheck MODE=report M=drivers/video/rockchip/bmp_helper.c 'SPFLAGS=--timeout 1' J=1
```

## 参考资料
官方网站：http://coccinelle.lip6.fr/
官方文档：Documentation/coccinelle.txt
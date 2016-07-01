---
title: fix-windows-shell-error-on-linux
date: 2016-07-01 15:08:08
tags: [linux]
---

在windows上编写的shell代码文件，丢到linux上运行时候经常会报错`syntax error near unexpected token`或者`command not found`等等，开始还天真的以为自己手误写错代码，其实怎么都检查不出错误，原来是因为编码格式在不同系统之间的兼容性问题以及换行符问题。

一般来讲我们可以在IDE中设置以unix格式保存文件，对于已经上传到linux上的文件也可以通过`vim`修改fileformat格式来修复此类问题。

```bash
# vim打开文件
vim test.sh
# 查看当前文件格式
:set fileformat
# 或者简写为
:set ff
# 此时可以看到当前文件格式
fileformat=dos
# 修改文件格式为unix
:set ff=unix
# 保存退出
:wq
```

修改之后再执行一般就没有问题了，还有问题那你真得检查下自己的代码了:D

---
title: idea配置查看代码汇编指令插件
date: 2020-02-05 23:22:59
tags:
---
**一、工具**
``` bash
hsdis-amd64.dll
hsdis-amd64.lib
```

**二.开始配置：**
1. 将上述两个文件放在你的 jre/bin 路径下的路径里。
2.去idea 去配置启动参数：-server -Xcomp -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:CompileCommand=compileonly,*ClassName.method（*后面配置成自己的 “类名.方法名”)
3.设置jre路径为配置过的jre路径
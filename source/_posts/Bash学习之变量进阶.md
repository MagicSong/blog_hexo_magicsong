---
title: Bash学习之变量进阶
date: 2018-03-20 09:43:36
categories:
    - linux
tags:
    - bash
    - variable
    - shell
    - 读书笔记
---

> 写脚本时候常常需要用到变量，但是Linux中的变量和一般编程语言中的变量差别较大，这里罗列一些比较细节的用法。

# 变量的特殊用法
1. `$?` 表示上一个语句的执行结果
2. `$#` 脚本或者函数输入的参数数量
3. `$0` 表示脚本名称
4. `$1-n` 表示具体输入的第几个变量
5. `$@` 表示所有变量
5. `${!name}` 间接引用，表示将`$name`这个变量的值作为一个变量的名称，如下：
```bash
$ export xyzzy=plugh ; export plugh=cave
$ echo ${xyzzy}  # normal, xyzzy to plugh
plugh
$ echo ${!xyzzy} # indirection, xyzzy to plugh to cave
cave
```
这个特殊用法还有一个用法，```${!#}```，我们知道`$#`表示变量个数，那么上述用法其实就是参数中的最后一个元素。
6. **shift**命令，shift可以将参数依次轮询，即通过`$1`就可以访问所有元素。注意这个不会影响`$0`。shifit可以接受参数，指定跳跃多少个参数。但是如果跳跃的值大于了当前的`$#`，那么参数不会受影响，而shift则会返回0。下面这个案列展示了这个效果：
```bash
#!/bin/bash
# shift-past.sh
shift 3 # Shift 3 positions.
n=3; shift $n
# Has the same effect.
echo "$1"
exit 0
# ======================== #
until [ -z "$1" ]
do
    echo -n "$1 "
    shift 20 # If less than 20 pos params,
done #+ then loop never ends!
# When in doubt, add a sanity check. . . .
shift 20 || break
```
7. **let**指令用于变量数值的加减。
8. 字符串在加减中默认为0


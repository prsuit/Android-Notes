---
title: AS常用快捷键汇总Mac
tags: [工具系列]
categories: [工具系列]
date: 2021-10-27 17:12:17
---

### mac上按键符号

#### ⌥ : `option / alt` 

#### ⇧ : `shift`

#### ⌃ : `control`

#### ⌘ : `command`

### （一）查找/查看相关

搜索任意内容

```
双击 sft
```

<!--more-->

当前文件查找/替换

```
cmd + F / cmd + R
cmd + G  跳到下一个高亮的变量
sft + cmd + G  跳到上一个高亮的变量
```

全局查找/替换

```
sft + cmd + F/R
```

全局搜索类

```
cmd + O
```

全局搜索文件

```
sft + cmd + O
```

全局搜索类/方法/参数

```
opt + cmd + O
```

打开最近访问的文件列表

```
cmd + E
```

类/方法在全局项目中引用情况

```
opt + fn + F7 / cmd + 鼠标点击
```

类/方法在当前文件中引用情况

```
cmd + fn + F7
```

方法被调用层级结构

```
ctr + opt + H
```

查看接口的实现

```
opt + cmd + B
```

跳转至超类的方法

```
cmd + U
```

跳转至第几行

```
cmd + L
```

返回到上次编辑位置

```
cmd + [ / ]
opt + cmd + ← / →
```

当前编辑的文件中结构快速导航

```
cmd + fn + F12
```

列出函数方法一系列的有效参数

```
cmd + P
```

跳转至错误或警告

```
fn + F2
```

查看类／方法的注释文档

```
fn + F1
```

### （二）控制操作相关

Surround with快速调出if,for,try…catch,while等环绕代码

```
opt + cmd + T
```

快速生成模版代码块，如if,while,return

```
cmd + J
```

快速生成getter／setter方法，构造方法，toString()方法等

```
cmd + N
```

行尾自动添加分号，if后面自动加“(){ }”

```
sft + cmd + enter
```

引入重写父类的方法

```
ctr + O
```

引入接口或抽象类方法的实现

```
ctr + I
```

将最近使用的剪贴板内容选择插入到文本

```
sft + cmd + V
```

单行注释与取消注释，注释效果 //…

```
cmd + /
```

多行注释与取消注释，注释效果 /*…*/

```
opt + cmd + /
```

上下移动代码

```
opt + sft + up/down
```

上下代码行换位

```
sft + cmd + up/down
```

切换大小写

```
sft + cmd + U
```

切换文件

```
ctr + tab
```

选择区域

```
opt + up/down
```

局部代码块展开/收缩

```
cmd + '+' / '-'
```

全部代码块展开/收缩

```
sft + cmd + '+' / '-'
```

撤销/取消撤销

```
cmd + Z / sft + cmd + Z
```

删除行

```
cmd + X/delete
```

复制行

```
cmd + D
```

合并行

```
sft + ctr + J
```

列编辑

```
alt + 鼠标框选
```

格式化代码

```
opt + cmd + L
```

清除无效包引用

```
opt + ctr + O
```

打开设置

```
cmd + ,
```

隐藏窗口

```
sft + esc
```

### （三）代码重构相关

类名/方法名/变量名 重命名操作

```
sft + fn + F6
```

方法重构，方法抽离

```
opt + cmd + M
```

抽离成方法参数

```
opt + cmd + P
```

抽离为局部变量

```
opt + cmd + V
```

抽离为成员变量

```
opt + cmd + F
```

### （四）编译运行调试

编译源码

```
cmd + fn + F9
```

运行

```
ctr + R
```

调试

```
ctr + D
```

Step Into（进入到代码）

```
fn + F7
```

Step Over（跳到下一步）

```
fn + F8
```

直接运行

```
opt + cmd + R
```

退出调试

```
cmd + fn + F2
```

### （五）版本控制

打开git操作列表

```
ctr + V
```

提交修改

```
cmd + K
```

推到服务器

```
sft + cmd + K
```


---
title: "linux terminal 目录的背景色"
subtitle: ""
date: 2023-07-12T16:45:30+08:00
draft: false
author: ""
authorLink: ""
description: "修改LS_COLORS实现对目录背景色的修改"
keywords: ""
license: ""
comment: false
weight: 0

tags: ["linux"]
categories: ["笔记"]

hiddenFromHomePage: false
hiddenFromSearch: false

summary: ""
resources:
- name: featured-image
  src: 109458640_p0.png
- name: featured-image-preview
  src: preview.png

lightgallery: true

---

<!--more-->
## 问题
{{<image src="example.png" caption="背景色">}}
平时用terminal的时候，偶尔会遇到这种有背景色的目录，这是因为该路径除了所有者外，还有其他人有权限。在有背景色的情况下，很容易与主题色发生冲突，导致看不清路径名称。修改方案除了调整路径的权限外，还有修改LS_COLORS环境变量。

```bash
$ echo $LS_COLORS
di=1;36:ln=35:so=32:pi=33:ex=31:bd=34;46:cd=34;43:su=30;41:sg=30;46:tw=30;42:ow=30;43
```
LS_COLORS环境变量由多个键值对用冒号连接而成，每个键值对的结构为`key=val1;val2`，不同值之间用分号连接。其中key为文件的类型缩写，value为对应的颜色编码

## key值表
| key | dir_colors name | description |
|--|--|--|
|no|	NORMAL, NORM|	Global default, although everything should be something|
|fi|	FILE|	Normal file|
|di|	DIR|	Directory|
|ln|	SYMLINK, LINK, LNK|	Symbolic link. If you set this to 'target' instead of a numerical value, the colour is as for the file pointed to.|
|pi|	FIFO, PIPE|	Named pipe|
|do|	DOOR|	Door|
|bd|	BLOCK, BLK|	Block device|
|cd|	CHAR, CHR|	Character device|
|or|	ORPHAN|	Symbolic link pointing to a non-existent file|
|so|	SOCK|	Socket|
|su|	SETUID|	File that is setuid (u+s)|
|sg|	SETGID|	File that is setgid (g+s)|
|tw|	STICKY_OTHER_WRITABLE|	Directory that is sticky and other-writable (+t,o+w)|
|ow|	OTHER_WRITABLE	Directory| that is other-writable (o+w) and not sticky|
|st|	STICKY|	Directory with the sticky bit set (+t) and not other-writable|
|ex|	EXEC|	Executable file (i.e. has 'x' set in permissions)|
|mi|	MISSING|	Non-existent file pointed to by a symbolic link (visible when you type ls -l)|
|lc|	LEFTCODE, LEFT|	Opening terminal code|
|rc|	RIGHTCODE, RIGHT|	Closing terminal code|
|ec|	ENDCODE, END|	Non-filename text|
|*.extension|	 	|Every file using this extension e.g. *.jpg|

## value表
### effects
|code|property|
|---|---|
|00|	Default colour|
|01|	Bold|
|04|	Underlined|
|05|	Flashing text|
|07|	Reversetd|
|08|	Concealed|

### colors
|code|property|
|---|---|
|30|	Black|
|31|	Red|
|32|	Green|
|33|	Orange|
|34|	Blue|
|35|	Purple|
|36|	Cyan|
|37|	Grey|

### backgrounds
|code|property|
|---|---|
|40|	Black background|
|41|	Red background|
|42|	Green background|
|43|	Orange background|
|44|	Blue background|
|45|	Purple background|
|46|	Cyan background|
|47|	Grey background|

### extra colors
|code|property|
|---|---|
|90	|Dark grey|
|91	|Light red|
|92	|Light green|
|93	|Yellow|
|94	|Light blue|
|95	|Light purple|
|96	|Turquoise|
|97	|White|
|100|	Dark grey background|
|101|	Light red background|
|102|	Light green background|
|103|	Yellow background|
|104|	Light blue background|
|105|	Light purple background|
|106|	Turquoise background|
|107|	White background|

## 修改
```bash
export LS_COLORS=${LS_COLORS}'key=val1;val2':
```
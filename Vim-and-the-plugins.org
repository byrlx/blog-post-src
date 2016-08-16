---
layout: post
title: Vim and the plugins
category: Life
tag: [Tech]
description: [The vim plugins I use]
---

学Linux之初，就知道有vim这么个东西，但一直没有深入使用，读研之后，开始写代码，
开始使用vim，其中多少也有些装X的成分。那时候对vim了解不深，用的也都是些基本的功能。
工作之后，刚去公司，就一直用vim学习代码，也开始学习些更复杂的操作，期间也试着
用了一段时间的UE以及公司买的sourceinlight。不过没多久就又回到vim里了。

就跟练武一样，时间久了，”武器“就越用越顺手，遂决定将其作为”首席武器”。于是开始
去网上搜高级操作，搜各种实用插件。偶尔也会去看一些“vim”和“emacs”巅峰对决的帖子，
想着自己正使用着代码界的”两大神器“之一，就会又膨胀了一些。

写此文，主要记录平常使用的vim插件及使用心得。插件越来越多，总免不了会有一些操作
或快捷键上的冲突。记下来，将来用的时候省得再去翻文档。

#### 1. [taglist.vim](http://www.vim.org/scripts/script.php?script_id=273)

该插件主要用于代码浏览，目前来讲我最常用的一个命令就是“TlistToggle”，我还把F2
键赋予了这个命令.
	
	nmpa <F2> :TlistToggle <CR>

当在vim输入“:TlistToggle”命令后，默认会在vim左边开启一个“窗口”，该窗口中会按照
”varible/class/fuction“等关键字分成几段文字，每一段文字都显示了目前在该代码文件
中定义的代码”关键字“, 通过将光标移到相应的关键字上，然后按回车键，会自动跳到代码
文件相应的定义处。同时，如果在代码文件中不停的跳转，该“窗口”的光标也会跟着自动跳跃
到相关的函数名处。

![taglist.vim]({{root_url}}/public/img/taglist.png "taglist.vim")

默认情况，进入vim后左边的“窗口”不会启动，可以通过修改vimrc，加入*"let Tlist_Auto_Open=1"*
将其设为进入vim自动启动。

默认情况，打开taglist之后，vim会开启一个单独的buffer，所以如果退出了代码文件的buffer，
taglist的buffer默认还会存在，如果想让所有源文件都退出后vim自动退出（即taglist的buffer
自动退出），可以在vimrc中设定*"let Tlist_Exit_OnlyWindow=1"*。

可以通过*"let Tlist_WinWidth = x(let Tlist_WinHeight = x)"* 设定
taglist 窗口的宽度（高度，适用于新窗口以水平方式打开)。

#### 2. [conque_term.vim](http://www.vim.org/scripts/script.php?script_id=2771)

该插件可以让你在vim里打开一个窗口执行shell命令。对一些简单的shell命令，用起来还
蛮方便的。

![conque.vim]({{root_url}}/public/img/conque.png "conque.vim")

常用命令“ :ConqueTermSplit bash”。

#### 3. [ack.vim](http://www.vim.org/scripts/script.php?script_id=2572)

要使用这个插件，必须先安装“ack”这个工具，笔者在[这篇文章](http://byrlx.github.io/2013/10/31/Welcome-ack-to-my-toolbox.html)中有对这个tool作简单介绍。

安装以后，可以通过“:Ack xx”在命令模式下使用ack命令，命令结果默认会输出到vim底端新开启的“窗口”中，
通过上下选择不同的结果，可以跳转到相应文件的相应行。

![ack.vim]({{root_url}}/public/img/ack.png "ack.vim")

#### 4. [matrix.vim](http://www.vim.org/scripts/script.php?script_id=1189)

一个很有趣的插件，让vim窗口变成《黑客帝国》。

![matrix.vim]({{root_url}}/public/img/matrix.png "matrix.vim")

#### 5. [minibufexpl.vim](http://www.vim.org/scripts/script.php?script_id=159)

很实用的插件，如果你打开多个文件，会自动在最上方开一个类似“tab”的窗口，显示每个打开的文件名称，可以使用TAB键在文件名之间移动，按下回车即可以切换到该文件的buffer。

![minibuf.vim]({{root_url}}/public/img/minibufexpl.png "minibufexpl.vim")

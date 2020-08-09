title: VIMscript学习笔记
toc: true
original: true
date: 2015-12-26 09:06:53
tags:
- vim
- Linux
categories:
- 学习笔记
---
之前的vim一直在用这个项目：[kvim](https://github.com/wklken/k-vim)，但是用习惯了也想有一份自己的vim配置，为了提高效率，于是就学起了vimscript（主要是为了`~/.vimrc`）。

	vi 之大道如我心之禅，
	vi 之漫路即为禅修，
	vi 之命令禅印于心，
	未得此道者视之怪诞，
	与之为伴者洞其真谛，
	长修此道者巨变人生。
	作：reddy@lion.austin.com
	译：yangyangwithgnu@yeah.net

## 学习笔记

### 打印信息
打印信息有`echo`和`echom`两个，但是使用`echom`会把打印的消息存到历史记录，用`message`可查看。

### 注释
在编写vimscript脚本(`~/.vimrc`)的时候，可用`"`来注释，但是不一定有效，例如

	echo "Hello World!" "hellowrld

### 设置选项

#### 布尔选项

##### 打开关闭布尔选项
可用`set`来打开布尔选项，例如`set number`, `set nonumber`，所有布尔选项都可以用`set <name>`打开，`set no<name>`关闭。

##### 切换布尔选项开关
`set <name>!`可切换开关状态，例如`set number!`，若当前显示（隐藏）行号可关闭（打开）。

##### 查看当前值
`set <name>?`可显示当前值，例如`set number?`，若行号已打开则输出`number`，否则输出`nonumber`。

#### 键值选项
有些选项不只有`on`和`off`两种选项，需要一个值，可通过`set <name>=<value>`，例如`set numberwidth=2`。

同样可用`set <name>?`来查看值。

### 基本映射
用`map KEY CMD`可以映射按键到指定功能，例如`map - dd`，这时在`normal`模式下按下`-`即可删除一行。

需要注意`map`在`normal`和`visual`模式下有用，在`insert`模式下无用。

`<keyname>`可告诉vim一些特殊的按键，比如`<space>`代表空格键。例如`map <c-d> dd`表示按下`ctrl+d`删除一行。

### 模式映射
前面提到`map`在`normal`和`visual`模式下有用，在`insert`模式下无用。

这时可以使用`nmap`, `vmap`, `imap`分别指定仅在`normal`,`visual`,`insert`模式下生效。

这里举个例子，`imap <c-d> <esc>ddi`可在`insert`模式下，按下`ctrl+d`即可删除一行。

### 删除映射
用`*unmap KEY`能删除映射，例如

	nmap - dd
	nunmap -

### 递归
运行命令

	nmap dd 0<esc>jddk

这时会进入递归状态永不终止，导致出现一些奇怪的问题。

### 非递归映射
`*map`系列存在递归的危险，还有若重定义一个按键不同行为冲突会导致前面映射无效。

所以`vim`提供了一组非递归映射指令`*noremap`，在创建映射的时候不会进行递归。所以尽量用`*noremap`指令，能省很多麻烦。

### Leaders
`vim`强大之处在于可随意映射键位，为了解决重定义键位带来的冲突，这里可用前缀`<leader>`来触发。

首先定义`<leader>`，我习惯用`,`做`<leader>`

	let mapleader = ','

在键盘映射的时候，可以这样

	nnoremap <leader>d dd

这样可以通过按`,d`来删除一行。

### Local Leader
`vim`还可以设置`<localleader>`来针对于某些文件设置映射，使用方法和`<leader>`一样。

	let maplocalleader = ','

### 编辑/加载vimrc映射

	:nnoremap <leader>ev :vsplit $MYVIMRC<cr>
	:nnoremap <leader>rv :source $MYVIMRC<cr>

这样可以在编辑文件的时候通过按`<leader>ev`，`<leader>rv`来打开`.vimrc`文件进行编辑/加载。
![编辑/加载vimrc](http://7xibui.com1.z0.glb.clouddn.com/%40%2F2015%2F12%2Fscreenshot-window-2015-12-26-100743.png)

### Abbreviations（缩写）
`vim`还有灵活的缩写功能，这里只介绍`insert`模式下的缩写功能。

	iabbrev blog http://www.netcan666.com

在`insert`下输入`blog`就会替换成`http://www.netcan666.com`了。

那为啥不用映射呢？

	inoremap blog http://www.netcan666.com

因为用映射的话，但你输入`myblog`就会变成`myhttp://www.netcan666.com`，而`abbrev`是不会替换的。 

### 熟悉自定义映射
当你定义了一个快捷键的时候，例如

	inoremap jk <esc>

试图在`insert`模式下快速按下`jk`来进入`normal`模式，可是以前都习惯了直接按`<esc>`进入，这时可以这样：

	inoremap <esc> <nop>

`<nop>`即`no operation`，这样在`insert`模式下`<esc>`即无效了。可强迫自己使用自定义快捷键`jk`。

### 本地缓冲区的映射和选项设置

#### 映射
当设置一个映射后，开一个`buffer`你会发现会继承之前定义的映射，比如说

	nnoremap <buffer> <leader>x dd

当你在这个`buffer`下设置之后，切换到另一个`buffer`，`<leader>x`将不生效。

那么这里的`<leader>`是一个本地缓冲区`<leader>`，一般使用`<localleader>`而不是`<leader>`

这样在你编写一个其他人可能会用到的插件时，用`<localleader>`可防止覆盖他人的`<leader>`全局映射。

#### 特性

	:nnoremap <buffer> Q x
	:nnoremap          Q dd

按下`Q`将会使用第一个，因为第一个更具体，然而切换`buffer`之后将使用第二个，因为第一个未定义。

#### 缩写
	iabbrev <buffer> --- &mdash;

这里的`---`缩写将在当前`<buffer>`中生效。

#### 选项设置
同样的，在当前`buffer`设置选项可以用`setlocal OPTION`。

### 自动命令
> 自动命令可以让Vim自动执行某些指定的命令，这些指定的命令会在某些事件发生的时候执行。

比如

	autocmd BufNewFile * :write

将在你创建文件的时候就保存。（PS: 自动命令需要关闭`vim`才能删除）

### 自动命令结构

	:autocmd 事件1,事件2 * :write
				^       ^	^
				|       |	|
				|       |	要执行的命令
				|       |
				|       用于事件过滤的“模式（pattern）”
				|
				要监听的“事件”

`:help autocmd-events`可参看所有绑定的事件。

例如

	:autocmd BufNewFile,BufRead *.html setlocal nowrap

这样打开或者创建`html`文件将设置`nowrap`选项。

### FileType事件
这个可以为不同文件类型创建不同的映射，例如

	:autocmd FileType javascript nnoremap <buffer> <localleader>c I//<esc>
	:autocmd FileType python     nnoremap <buffer> <localleader>c I#<esc>

在`buffer`打开`*.js`文件，可按`<leader>c`注释`//`一行，而在`buffer`新开`*.py`文件，可用`#`注释一行。

### 自动命令缩写

	autocmd FileType python     :iabbrev <buffer> iff if:<left>
	autocmd FileType javascript :iabbrev <buffer> iff if ()<left>

将在不同文件类型采用不同缩写。

同样可以禁用之前的写法，例如

	iabbrev <buffer> if "233333333333333"

这样就不得不逼自己使用`iff`缩写了。

### 自动命令组
> 自动命令组主要就是可以为一组命令命名，多组相同的名称可以组合。

例如

	augroup testgroup
	    autocmd BufWrite * :echom "Foo"
	    autocmd BufWrite * :echom "Bar"
	augroup END
	augroup testgroup
	    autocmd BufWrite * :echom "Baz"
	augroup END

将结合这三个命令。当你保存文件的时候，就会输出

	Foo
	Bar
	Baz

这里有个重要的用法，就是当你创建的自动命令触发的事件一致的时候，`vim`并不会覆盖之前的事件，也就是说当你触发事件的时候，会执行所有定义过的自动命令。

例如

		autocmd BufWrite * :sleep 200m
		autocmd BufWrite * :sleep 200m
		autocmd BufWrite * :sleep 200m

保存文件会依次执行三次`sleep 200m`。

用命令组`autocmd!`可以删除之前定义过的事件，

	augroup testgroup
	    autocmd BufWrite * :echom "Foo"
	    autocmd BufWrite * :echom "Bar"
	augroup END
	augroup testgroup
	"" 用autocmd!即可清除前面同名组的命令
		autocmd!
	    autocmd BufWrite * :echom "Baz"
	augroup END

保存文件的时候将只输出`Baz`。

### Operator-Pending映射
这部分比较难了。

> 一个Operator（操作）就是一个命令，你可以在这个命令的后面输入一个Movement（移动）命令，然后Vim开始对文本执行前面的操作命令，这个操作命令会从你当前所在的位置开始执行，一直到这个移动命令会把你带到的位置结束。

常用到的Operator有d，y和c。例如：

按键 | 操作  |     移动
:-:|:-:|:-:
dw   |  删除  |     到下一个单词
ci( |   修改   |    在括号内
yt,  |  复制   |    到逗号

### Movement映射
> Vim允许你创建任何新的movements，这些movements可以跟所有命令一起工作。

例如

	onoremap p i(

这样就可以用`dp`代替`di(`了，也就是删除括号内的文字。

#### 从当前光标位置开始
> 当你想搞清楚怎么定义一个新的operator-pending movement的时候，你可以从下面几个步骤来思考：

1. 在光标所在的位置开始。
2. 进入可视模式(charwise)。
3. ... 把映射的按键放到这里 ...
4. 所有你想包含在movement中的文字都会被选中。

你所要做的工作就是在第三步中填上合适的按键。

例如

	onoremap p :<c-u> normal! vllll<cr>

执行`dp`后，就会删除当前光标后四个字符。

这里的`normal`可以看做模拟按键操作（加个`!`是为了不使用映射），即按了四下`llll`。

`<c-u>`原文并没指出具体作用，Google了一下好像可以清除命令行，不用理会这个。

#### 改变开始位置
只要移动到指定位置，在`visual`模式下选中，即可操作选中的文本。

> 下面两条规则可以让你可以很直观的以多种方式创建operator-pending映射：

1. 如果你的operator-pending映射以在可视模式下选中文本结束，Vim会操作这些文本。
2. 否则，Vim会操作从光标的原始位置到一个新位置之间的文本。

#### 实战
这里我将用到前面介绍的`Movement`映射，来快速修改电子邮件地址。

	onoremap in@ :<c-u>execute "normal! /[0-9A-Za-z_-]\\+@\r:nohlsearch\rvt@"<cr>

比如文本中有`123456789@xx.xx`，当你按下`cin@`，就会删除`123456789`部分，并进入`insert`模式进行修改。

这里又有个新命令`execute`，为什么不直接用`normal`呢？

因为`normal`不能模拟特殊按键，例如`:normal! /a<cr>`，并没有反应，因为他不能识别`<cr>`。。。

> 当execute碰到任何你想让它执行的字符串的时候。它会先替换这个字符串中的所有特殊字符。在这个示例中，\r是一个转义字符，它表示的是"回车符（carriage return）"。两个反斜线也是一个转义字符，它会将一个反斜线当作一般字符放到字符串中。

所以按照这个说法来理解，就可以搞清楚这个映射怎么执行了。

	normal! /[0-9A-Za-z_-]\\+@\r:nohlsearch\rvt@
	                      ^    ^            ^
						  |    |            |
	normal! /[0-9A-Za-z_-]\+@<cr>:nohlsearch<cr>vt@

接着分析`normal!`部分，和键盘输入一样：

	/[0-9A-Za-z_-]\+@
	:nohlsearch
	vt@

1. 第一部分正则是匹配电子邮件`@`前面部分，并跳转到匹配的第一个字符
2. 然后紧接着清除匹配的字符高亮
3. 进入`visual`模式并选中`@`前面的部分

前面说了，如果你的operator-pending映射以在可视模式下选中文本结束，Vim会操作这些文本。

## 参考资料
* [笨方法学Vimscript](http://learnvimscriptthehardway.onefloweroneworld.com/)
* [所需即所获：像 IDE 一样使用 vim](https://github.com/yangyangwithgnu/use_vim_as_ide)

title: 试着用yacc和lex编写一个计算器
toc: false
original: true
tags:
  - C/C++
  - Yacc/Lex
categories:
  - 编程
date: 2015-04-09 17:11:49
updated: 2015-04-09 17:11:49
---
最近在看一本[日]前桥和弥写的《自制编程语言》，很感兴趣，里面第二章的案例就是试做一个计算器。

编程语言分为编译型语言和解释型语言两种。

编译语言比较有代表性的是C/C++，最终输出机器码可执行文件。

想要输出机器码的话，必须要掌握机器码才行，即使学习了机器码并写出了编译器，也无法输出其他型号cpu运行的文件。

机器码的编程语言优点是运行速度非常快，但是学习起来非常有难度= =所以这本书选择了开发解释性语言= =可以用编程语言扩展应用程序。。。

解释性语言的解释一词来自英语的interpreter，是“能进行翻译的意思”，程序员编写的代码通过解释器一边解析一边运行，但除了DOS的批处理脚本或Unix的SHELL脚本这么接近该定义，大多数解释型语言都会将源代码临时转换为某种中间形态。
<!--more-->

这有段代码，

	if(a == 10) {
		printf("hoge\n");
	} else {
		printf("piyo\n");
	}
大多数编程语言，都会将上面代码转换成一种叫语法分析树的东西。如上代码做成分析树如下图所示。
![http://7xibui.com1.z0.glb.clouddn.com/2015/04/tree.png](http://7xibui.com1.z0.glb.clouddn.com/2015/04/tree.png)

解释器会将源码或者分析树解析为字节码这种中间形态，并且一边解析一边运行，但是解释器并不会将源码翻译为机器码。

下面开始介绍计算器实例。

首先介绍yacc和lex这两个工具，一般编程语言的语法处理，都会有一下的过程。

1. 词法分析
将源代码分割为若干个记号（token）的处理
2. 语法分析
将从记号构建分析树(parse tree)的处理，分析树也叫语法树或者抽象语法树。
3. 语义分析
经过语法分析生成的分析树，并不包含数据类型等语义信息，因此在语义分析阶段，会检查程序中是否含有语法正确但是存在逻辑问题的错误。
一般来说执行语义分析时主要会做数据类型的解析以及错误的检查。
4. 生成代码
如果是C语言等生成机器码的编译器或Java这样生成字节码的编译器，在分析树构建完毕后会进入代码生成阶段。

比如说有如下代码：

	if(a == 10) {
		printf("hoge\n");
	} else {
		printf("piyo\n");
	}

执行词法分析后，将被分割成如下所示的记号（每一个块就是一个记号）。

If` `(` `a` `==` `10` `)` `{` `printf` `(` `“ hoge\n”` `)` `;` `}` `else` `{` `printf` `(` `“piyo\n”` `)` `;` `}`

语法分析树如图：
![http://7xibui.com1.z0.glb.clouddn.com/2015/04/tree.png](http://7xibui.com1.z0.glb.clouddn.com/2015/04/tree.png)

执行词法分析的程序称为词法分析器（lexical analyzer)，lex的工作原理就是根据词法规则自动生成的词法分析器。

执行语法分析的程序称为解析器(parser)，yacc就是能根据语法规则自动生成解析器程序。

yacc和lex一起使用可以将一个特殊格式的定义文件输出为C语言代码。

书中第一个例子是制作一个简单的计算器，因为一开始就讲“用yacc/lex制作编程语言”肯定会吃不消的。。我们先来看看有什么功能吧。。
![http://7xibui.com1.z0.glb.clouddn.com/2015/04/blob.png](http://7xibui.com1.z0.glb.clouddn.com/2015/04/blob.png)

看得出来mycalc可按照优先级输出结果。虽然使用了整数，但是内部是用double实现的运算。


lex是自动生成词法分析器的工具，通过扩展名为.l的文件，输出词法分析器的C语言代码。

词法分析器是将输入的字符串分割为记号的程序，因此必须首先定义mycalc所用到的符号。

1. 运算符。在这个例子中可以使用四则运算，即+、-、*、/
2. 整数，比如233
3. 实数，比如2.33
4. 换行符。输入换行符进行运算，所以这也是个记号。

在lex中使用正则表达式定义记号。下面是mycalc.l文件。

```
%{
    #include <stdio.h>
    #include "mycalc.h"
    int yywrap(void) {
        return 1;
    }
%}
%%
"+" return ADD;
"-" return SUB;
"*" return MUL;
"/" return DIV;
"\n"    return CR;
(([1-9][0-9]*)|0|([0-9]+\.+[0-9]+)) {
    double temp;
    sscanf(yytext, "%lf", &temp);
    yylval.double_value = temp;
    return DOUBLE_LITERAL;
    }
[ \t] ;
. {
    fprintf(stderr, "Lexical error!\n");
    exit(1);
    }

%%
```

代码第8行为%%，此行之前部分叫做定义区域。在定义区域内，可以定义初始状态或者为正则表达式命名。

记号|含义
-|-
ADD|加法运算符+
SUB|减法运算符-
MUL|乘法运算符*
DIV|除法运算符/
CR|回车符
DOUBLE_LITERAL|double类型的字面常量


第4-6行定义了yywrap()函数，没有这个函数的话需要连接lex库文件。

第9-24行是规则区域，使用正则表达式描述记号。

在规则区域中遵循这样的书写方式：一个正则表达式后面紧跟若干个空格，后接C代码。如果输入匹配正则表达式，则执行后面的C代码。这里的C代码部分称为动作(action)。

第9-13行中会找到四则运算符以及换行符，通过return返回其特征符，特征符是在y.tab.h中用#define定义，用来区分记号种类的代号。

对于+或者-这样的记号只需要关注其记号种类即可，如果是DOUBLE_LIBTERAL记号，则记号的种类和记号的值都必须传递给解析器。

第14行匹配一个数值，记号的原始字符（比如输入123.45这个记号的原始字符就是1"123.45"）会在相应动作中的被名为yytext的全局变量引用。

动作解析的值会存放在名为yyval的全局变量中，这个全局变量本质是一个联合体(union)，可以存放各种记号的值（在这个计算器程序只有double的值）。联合体的定义部分写在yacc的定义文件mycalc.y中。

到26行，出现一次%%。表示规则区域的结束，这之后的代码称为用户代码区域。用户代码区域可以随意编写任意C代码，与定义区域不同的是，用户区域无需使用％{％}包裹。


yacc是自动生成语法分析器的工具，输入扩展名为.y的文件，就会输出语法分析器的C语言代码。mycalc.y代码如下。

```
%{
#include <stdio.h>
#include <stdlib.h>
#define YYDEBUG 1
%}
%union {
    int int_value;
    double double_value;
}
%token <double_value> DOUBLE_LITERAL
%token ADD SUB MUL DIV CR
%type <double_value> expression term primary_expression
%%
line_list
    : line
    | line_list line;
line
    : expression CR
    {
        printf("\e[32m[Netcan-Soft]>>\e[0m \e[36m%lf\e[0m\n", $1);
    };
expression
    : term
    | expression ADD term
    {
        $$ = $1 + $3;
    }
    | expression SUB term
    {
        $$ = $1 - $3;
    };
term
    : primary_expression
    | term MUL primary_expression
    {
        $$ = $1 * $3;
    }
    | term DIV primary_expression
    {
        $$ = $1 / $3;
    };

primary_expression
    : DOUBLE_LITERAL;
%%
int yyerror(char const *str) {
    extern char *yytext;
    fprintf(stderr, "\e[31m解析错误: %s\e[0m\n", yytext);
    return 0;
}

int main() {
    extern int yyparse(void);
    extern FILE * yyin;
    yyin = stdin;
    if(yyparse()) {
        fprintf(stderr, "\e[31m错误！\e[0m\n");
        exit(1);
    }
}
```

第1-5行与lex相同，使用％{％}包裹一些C代码。

第6-9行声明了记号以及非终结符的类型。前面提过记号不仅要包含类型，还需要包含值。记号可能有很多类型，这些类型都声明在集合（共用体）中。

非终结符是由多个记号共同构成的，即代码中的line_list、line、expression、term这些部分。为了分割非终结符，非终结符最后都会以一个特殊记号结尾，这种记号称为终结符。

10-11行是记号的声明。ADD、SUB、MUL、DIV、CR等记号只需要包含记号的类型即可，而值为DOUBLE_LITERAL的记号，其类型被指定为<double_value>，这里的double_value来自上面代码中%union集合中的一个成员名。

12行声明了非终结符的类型。

与lex一样，13行的%%为分界，之后的为规则区域。yacc的规则区域是由语法规则以及C语言编写相应的动作两部分组成。

可将语法规则简化为下面的格式。

	A
		: B C
		| D
		;

即A的定义是B和C的组合或者D。

14-16行的书写方式，为了表示该语法规则在程序中可能出现一次以上。例如在mycalc中，输入一行后回车进行计算，之后还可以继续输入。

	term
		: primary_expression
		| term MUL primary_expression
		{
			$$ = $1 * $3;
		}

这些表达式中，`$1`、`$3`的意思是分别保存了term与primary_expression的值。即yacc输出解析器的代码时，栈中相应位置的元素将会被转换为一个能表述元素特征的数组引用。`$$`这里这代表term的值。

如果没有书写动作，例如43-44行，yacc会自动补全一个`{ $$ = $1 }`的动作，`$$`与`$1`的数据类型，分别与其对应的记号或者非终结符的类型一致。例如，DOUBLE_LITERAL对应的记号被定义为：

	%token <double_value> DOUBLE_LITERAL
		expression、term、primary_expression的类型则为：

	%type <double_value> expression term primary_expression

这里的类型被指定为`<double_value>`，其实是使用了%union部分声明的联合体double_value成员。


生成可执行文件步骤如下，

	# lex mycalc.l
	# yacc -dv mycalc.y
	# gcc y.tab.c lex.yy.c -o mycalc


---
title: 利用lex和yacc做代码检查(上)
date: 2022-01-13 09:15:34
tags:
---
# 利用lex和yacc做代码检查(上)
这篇文章我们将使用lex和yacc来对公司代码进行扫描检查。背景：公司的业务代码是用伪代码和c++来实现的，再由开发工具翻译成c++语言，本次我们要代码检查的就是这些伪代码以及c++代码组成的业务代码，然后找出其中的类型不一致等问题。
![](Images\compiler_flow.png)
## 预备知识
以下的预备知识，对于理解lex和yacc的程序是必须的，因此如果不清楚，须先自学
1. 编译原理, 要求理c/c++解编译器的整体流程、理解词法分析、语法分析
概述(转): https://blog.csdn.net/cprimesplus/article/details/105724168
2. 正则表达式，主要在lex语法中用到，要求至少能看懂
入门：https://liji53.github.io/2021/12/03/regxStudy/
3. lex和yacc的语法，2者语法结构类似，yacc比lex要复杂一些，先理解清楚lex，再去理解yacc会容易很多
lex入门(转)：https://www.cnblogs.com/wp5719/p/5528896.html
lex和yacc小结(转)：https://www.daimajiaoliu.com/daima/4717f8908900400
4. c语言的语法文件，要求能看懂。 **这两个文件是本次语法分析的基础文件，后续代码都在此基础上添加的**
c语言lex的语法文件：http://www.quut.com/c/ANSI-C-grammar-l-1998.html
c语言yacc的语法文件：http://www.quut.com/c/ANSI-C-grammar-y-1998.html
   
## 本次词法、语法分析的目标
公司的业务代码长这个样子，这种[xxx]的写法就是伪代码，其本质是宏。c++部分的语法分析由于我只找到c的语法分析，而且用c的语法分析对接下来的语法解析足够了，因此直接使用最新的The ANSI C grammar(上文已经提到)， 
这个文件后续作为待测试文件记test.cpp
```c++
// 1）函数调用基本写法：[函数名][入参][出参]
[function_name][parameter1=1, parameter2="test"][output1=@id]
// 2）函数调用多行写法
[function_name2][
    parameter1=1, 
    parameter2="test"][]
// 3）宏调用写法,<A-Z> 可无
<A>[marco]
[marco][variable][]
// c++语法
for(int i = 0; i < 10; i++){
    variable1 = variable2;
} 
```
## 先熟悉流程
这一步我们通过生成c语言的语法分析器，来熟悉lex和yacc的流程。
下载http://www.quut.com/c/ANSI-C-grammar-l-1998.html, 命名c.l
下载http://www.quut.com/c/ANSI-C-grammar-y-1998.html, 命名c.y
#### 1. 编译词法文件, 会生成lex.yy.c
```shell
# 编译词法文件
lex c.l
```
#### 2. 编译语法文件, 会生成y.tab.c, y.tab.h
```shell
# 直接编译会出现yacc: 1 shift/reduce conflict. 错误，先编辑c.y文件
# 2.1 在c.y的序幕部分加以下内容
%nonassoc LOWER_THAN_ELSE
%nonassoc ELSE
# 2.2 在c.y的规则部分selection_statement修改成以下内容
selection_statement
  : IF '(' expression ')' statement %prec LOWER_THAN_ELSE
  | IF '(' expression ')' statement ELSE statement
  | SWITCH '(' expression ')' statement
  ;
# 2.3 在c.y的最后，增加main函数
int main(void) {
    yyparse();
    return 0;
}
# 2.4 编译语法文件，其中-d用来生成头文件
yacc -d c.y
```
#### 3. 生成语法分析器, 生成a.out
```shell
gcc y.tab.c lex.yy.c
```
#### 4. 测试词法分析器
```shell
# 4.1 写一个简单的c文件,带上错误
echo "int main(){a=b}" > test.cpp  
# 4.2 测试下分析器能不能检查出错误
./a.out < test.cpp
# 4.3 程序输出结果如下，perfect
int main(){a=b}
              ^
   syntax error
```
## 词法分析lex
在这里词法分析主要对上面示例中的1,2,3种写法进行解析，这里需要判断是否是伪代码，并返回伪代码的标记。由于c.l源文件内容较多，这里只贴修改部分代码
#### 1. 在定义部分，增加2个伪代码的状态标志 
```c++
%{
#include <stdio.h> 
#include "y.tab.h"
void count(void);
int marco_flag = 0;  //伪代码状态，与MARCO_HEAD类似,表示除第一个[xxx]以外的状态
%}
/* 状态（或条件）定义- 用来标志伪代码头,即第一个[xxx]部分的状态 */
%s MARCO_HEAD
%%
```
#### 2. 在规则部分，增加伪代码的规则解析
MARCO_SYMBOL_BEGIN、MARCO_NAME等需要在c.y中定义。
```c++
"<"[A-Z]">["                    {count(); BEGIN MARCO_HEAD; return(MARCO_SYMBOL_BEGIN); }
[ \t]*"["                       {count(); BEGIN MARCO_HEAD; return(MARCO_SYMBOL_BEGIN); }
<MARCO_HEAD>{L}({L}|{D})*       {count(); return(MARCO_NAME); }
<MARCO_HEAD>"]["                {count(); BEGIN INITIAL; marco_flag = 1; return(MARCO_SYMBOL_SPLIT); } 
<MARCO_HEAD>"]"                 {/*必须 写在<MARCO_HEAD>"][" 之后*/   count(); BEGIN INITIAL; return(MARCO_SYMBOL_END); }   
<MARCO_HEAD>[ \t\v\n\f]	        {count(); }
<MARCO_HEAD>.                   {/* ECHO是一个宏，相当于 fprintf(yyout, "%s", yytext)*/ ECHO; }    
"]"                             {
                                    count();
                                    if(marco_flag){
                                        if(input() == '['){
                                            return (MARCO_SYMBOL_SPLIT);
                                        }
                                        else{
                                            marco_flag = 0;     
                                            return (MARCO_SYMBOL_END);
                                        }
                                    }
                                    else{ 
                                        return(']');
                                    } 
                                }
```
这部分代码至少要放在下面代码之前，否则会因为规则的先后匹配错误
```c++
("["|"<:")		{ count(); return('['); }
("]"|":>")		{ count(); return(']'); }
```
## 测试lex正确性
最后我们测试下词法分析器的正确性，待测试文件即上文提到的test.cpp
#### 1. 在c.y中定义部分添加宏定义
后面只用到了c.y生成的头文件
```c++
%token MARCO_SYMBOL_BEGIN MARCO_SYMBOL_SPLIT MARCO_SYMBOL_END MARCO_NAME
```
#### 2. 在c.l中添加main函数
```c
void writeout(int c){
  switch(c){
    case MARCO_SYMBOL_BEGIN: fprintf(yyout, "(MARCO_SYMBOL_BEGIN, \"%s\") ", yytext);break;
    case MARCO_SYMBOL_SPLIT: fprintf(yyout, "(MARCO_SYMBOL_SPLIT, \"%s\") ", yytext);break;
    case MARCO_SYMBOL_END: fprintf(yyout, "(MARCO_SYMBOL_END, \"%s\") ", yytext);break;
    case MARCO_NAME: fprintf(yyout, "(MARCO_NAME, \"%s\") ", yytext);break;
    case IDENTIFIER: fprintf(yyout, "(IDENTIFIER, \"%s\") ", yytext);break;
    case CONSTANT: fprintf(yyout, "(CONSTANT, \"%s\") ", yytext);break;
    case STRING_LITERAL: fprintf(yyout, "(STRING_LITERAL, \"%s\") ", yytext);break;
    case SIZEOF: fprintf(yyout, "(SIZEOF, \"%s\") ", yytext);break;
    default:break;
  }
}
int main (int argc, char ** argv){
	int c;
	if (argc>=2){
	  if ((yyin = fopen(argv[1], "r")) == NULL){
	    printf("Can't open file %s\n", argv[1]);
	    return -1;
	  }
	  while (c = yylex()){		
        writeout(c);
	  }
	  fclose(yyin);
	}
	return 0;
}
```
#### 3. 编译词法分析器
```shell
lex lex.l && yacc -d yacc.y && gcc lex.yy.c
```
#### 4. 测试test.cpp
```shell
[liji@null test_compile]$ ./a.out test.cpp

[(MARCO_SYMBOL_BEGIN, "[") function_name(MARCO_NAME, "function_name") ][(MARCO_SYMBOL_SPLIT, "][") parameter1(IDENTIFIER, "parameter1") =1(CONSTANT, "1") , parameter2(IDENTIFIER, "parameter2") ="test"(STRING_LITERAL, ""test"") ](MARCO_SYMBOL_SPLIT, "]") output1(IDENTIFIER, "output1") =id(IDENTIFIER, "id") ](MARCO_SYMBOL_END, "]") 

[(MARCO_SYMBOL_BEGIN, "[") function_name2(MARCO_NAME, "function_name2") ][(MARCO_SYMBOL_SPLIT, "][") 
    parameter1(IDENTIFIER, "parameter1") =1(CONSTANT, "1") , 
    parameter2(IDENTIFIER, "parameter2") ="test"(STRING_LITERAL, ""test"") ](MARCO_SYMBOL_SPLIT, "]") ](MARCO_SYMBOL_END, "]") 

<A>[(MARCO_SYMBOL_BEGIN, "<A>[") marco(MARCO_NAME, "marco") ](MARCO_SYMBOL_END, "]") 
[(MARCO_SYMBOL_BEGIN, "[") marco(MARCO_NAME, "marco") ][(MARCO_SYMBOL_SPLIT, "][") xxx(IDENTIFIER, "xxx") ](MARCO_SYMBOL_SPLIT, "]") ](MARCO_SYMBOL_END, "]") 

for(int i(IDENTIFIER, "i")  = 0(CONSTANT, "0") ; i(IDENTIFIER, "i")  < 10(CONSTANT, "10") ; i(IDENTIFIER, "i") ++){
    variable1(IDENTIFIER, "variable1")  = variable2(IDENTIFIER, "variable2") ;
```
从上文的第7行， ](MARCO_SYMBOL_SPLIT, "]")这个内容其实是有问题的。 因为在规则部分，处理"]"的时候，用到了input()来判断下个字符是不是"["， 因此yytext的值是"]"，其实应该是"][" 
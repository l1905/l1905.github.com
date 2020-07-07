---
title: 入门使用ANTLR词法语法分析工具
description: 入门使用ANTLR词法语法分析工具
categories:
 - golang
 - ANTLR
tags:
 - golang
 - ANTLR
---

## 入门使用ANTLR词法语法分析工具

## 背景

看到[B站开源](https://github.com/alyron/gengine)了golang版的规则引擎， 可以动态加载DSL程序代码， 非常酷。

之前一直感兴趣“代码生成代码”，但不得入门， 基于此项目比较简单， 重点研究下此项目的底层实现ANTLR的逻辑

### ANTLR:

能够根据【用户定义的语法文件】自动生成词法分析器和语法分析器， 并将输入文本处理为(可视化）语法分析树。 用户进而可以对此语法分析树做响应的逻辑操作。

ANTLR自动生成的编译器前端搞笑，准确，能够将开发者从繁杂的编译理论中解放出来，集中精力处理自己的业务逻辑， ANTLR 4引入的自动语法分析树创建与遍历机制， 极大的提高了语法识别程序的开发效率。

ANTLR的作者寄语：

>  为什么不花5天时间编程， 来使你25年的生活自动化呢？ 


## 安装
 
 ANTLR是基于java编写， 只要有java环境， 直接下载就能运行

 ```
$ wget http://www.antlr.org/download/antlr-4.7-complete.jar
$ alias antlr='java -jar $PWD/antlr-4.7-complete.jar'
 ```

## 使用流程

1.  自定义语法文件
2. 通过ANTLR通过语法文件生成对应语言的词法分析器和语法分析器
3. 步骤2生成遍历语法树的visitor和listener接口， 自定义自己的业务逻辑
4. 根据自定义语法，生成自己的规则内容
5. 通过语法分析器和词法分析器 处理 步骤4生成的规则内容， 然后调用自己步骤3定义的业务逻辑， 完成处理


下面我们 定义一个加减乘除法来，帮我们理解此过程

## 定义语法规则

calc.g4文件

```
grammar Calc; // 定义规则名称

// Tokens 元素
MUL: '*';
DIV: '/';
ADD: '+';
SUB: '-';
NUMBER: [0-9]+;
WHITESPACE: [ \r\n\t]+ -> skip;

// Rules // 入口规则
start : expr EOF; // 这里依赖expr规则

expr  // 具体的expr规则， 这里是个递归，可以自己调用自己，这里有先后顺序， 
   : expr op=('*'|'/') expr # MulDiv // 乘法
   | expr op=('+'|'-') expr # AddSub // 加法
   | NUMBER                 # Number // 数字
   ;

```

## 生成对应golang客户端调用语法


这里可以指定对应的语言， 我这里选择golang

```
 antlr -visitor -Dlanguage=Go -o parser Calc.g4
 
```
生成代码目录如下：

```
parser
├── Calc.tokens
├── CalcLexer.tokens。   // 生成的tokens
├── calc_base_listener.go // 基础listenr模式
├── calc_base_visitor.go // 基础visitor模式
├── calc_lexer.go // 此法分析文件
├── calc_listener.go // listen模式
├── calc_parser.go // 语法分析
└── calc_visitor.go // visitor模式

```

## 生成自己的listener模式和visitor模式业务逻辑处理代码


为了将文法与分析程序分离，关键是通过分析器创建一颗分析树，随后遍历这颗树，并在遍历时触发相关的处理代码。这可以通过Antlr提供的树遍历机制来实现


listener(观察者模式，通过结点监听，触发处理方法)， 这里即编译器方法会自动回调我这里的方法， 不需要用户定义需要

* 程序员不需要显示定义遍历语法树的顺序，实现简单
* 缺点，不能显示控制遍历语法树的顺序
* 动作代码与文法产生式解耦，利于文法产生式的重用
* 没有返回值，需要使用map、栈等结构在节点间传值

```
package logic

import (
	"fmt"
	"lt-rule/demo/parser"
	"strconv"
)

type CalcListener struct {
	*parser.BaseCalcListener

	stack []int
}

func (l *CalcListener) push(i int) {
	l.stack = append(l.stack, i)
}

func (l *CalcListener) Pop() int {
	if len(l.stack) < 1 {
		panic("Stack is empty unable to Pop")
	}

	// Get the last value from the Stack.
	result := l.stack[len(l.stack)-1]

	// Remove the last element from the Stack.
	l.stack = l.stack[:len(l.stack)-1]

	return result
}

func (l *CalcListener) ExitMulDiv(c *parser.MulDivContext) {
	fmt.Println("ExitMulDiv")
	right, left := l.Pop(), l.Pop()
	switch c.GetOp().GetTokenType() {
	case parser.CalcParserMUL:
		l.push(left * right)
	case parser.CalcParserDIV:
		l.push(left / right)
	default:
		panic(fmt.Sprintf("unexpected op: %s", c.GetOp().GetText()))
	}
}

func (l *CalcListener) ExitAddSub(c *parser.AddSubContext) {
	fmt.Println("ExitAddSub=====")
	right, left := l.Pop(), l.Pop()

	switch c.GetOp().GetTokenType() {
	case parser.CalcParserADD:
		l.push(left + right)
	case parser.CalcParserSUB:
		l.push(left - right)
	default:
		panic(fmt.Sprintf("unexpected op: %s", c.GetOp().GetText()))
	}
}

func (l *CalcListener) ExitNumber(c *parser.NumberContext) {
	fmt.Println("ExitNumber")
	i, err := strconv.Atoi(c.GetText())
	if err != nil {
		panic(err.Error())
	}

	l.push(i)
}

```


visitor模式(访问者模式，主动遍历)

* 程序员可以显示定义遍历语法树的顺序
* 不需要与antlr遍历类ParseTreeWalker一起使用，直接对tree操作
* 动作代码与文法产生式解耦，利于文法产生式的重用

```
package logic

import (
	"fmt"
	"github.com/antlr/antlr4/runtime/Go/antlr"
	"lt-rule/demo/parser"
	"strconv"
)

type CalVisitor struct {
	parser.BaseCalcVisitor
	Stack []int
}

func (l *CalVisitor) push(i int) {
	l.Stack = append(l.Stack, i)
}

func (l *CalVisitor) Pop() int {
	if len(l.Stack) < 1 {
		panic("Stack is empty unable to Pop")
	}
	// Get the last value from the Stack.
	result := l.Stack[len(l.Stack)-1]

	// Remove the last element from the Stack.
	l.Stack = l.Stack[:len(l.Stack)-1]

	return result
}

func (v *CalVisitor) visitRule(node antlr.RuleNode) interface{} {
	node.Accept(v)
	return nil
}

func (v *CalVisitor) VisitStart(ctx *parser.StartContext) interface{} {
	fmt.Println("VisitStart")
	return v.visitRule(ctx.Expr())
}

func (v *CalVisitor) VisitNumber(ctx *parser.NumberContext) interface{} {
	fmt.Println("VisitNumber")
	i, err := strconv.Atoi(ctx.NUMBER().GetText())
	if err != nil {
		panic(err.Error())
	}

	v.push(i)
	return nil
}

func (v *CalVisitor) VisitMulDiv(ctx *parser.MulDivContext) interface{} {
	fmt.Println("VisitMulDiv")
	//push expression result to Stack
	v.visitRule(ctx.Expr(0))
	v.visitRule(ctx.Expr(1))

	//push result to Stack
	var t antlr.Token = ctx.GetOp()
	right := v.Pop()
	left := v.Pop()
	switch t.GetTokenType() {
	case parser.CalcParserMUL:
		v.push(left * right)
	case parser.CalcParserDIV:
		v.push(left / right)
	default:
		panic("should not happen")

	}

	return nil
}

func (v *CalVisitor) VisitAddSub(ctx *parser.AddSubContext) interface{} {
	fmt.Println("VisitAddSub======")
	//push expression result to Stack
	v.visitRule(ctx.Expr(0))
	v.visitRule(ctx.Expr(1))

	//push result to Stack
	var t antlr.Token = ctx.GetOp()
	right := v.Pop()
	left := v.Pop()
	switch t.GetTokenType() {
	case parser.CalcParserADD:
		v.push(left + right)
	case parser.CalcParserSUB:
		v.push(left - right)
	default:
		panic("should not happen")
	}
	fmt.Println("VisitAddSub======exit")
	return nil

}

// 更偏向于中间递归 ( *, (2, 3) 这种
```


## 做测试

测试支持解析 `1 +     2 * 3+1+2344`


```
package main

import (
	"fmt"
	"github.com/antlr/antlr4/runtime/Go/antlr"
	"lt-rule/demo/logic"
	"lt-rule/demo/parser"
)

func main() {
	runVisitor()
}

func runListener()  {
	// Setup the input
	is := antlr.NewInputStream("1 +     2 * 3+1+2344")

	// Create the Lexer
	lexer := parser.NewCalcLexer(is)
	stream := antlr.NewCommonTokenStream(lexer, antlr.TokenDefaultChannel)

	// Create the Parser
	p := parser.NewCalcParser(stream)


	listen := logic.CalcListener{}
	// Finally parse the expression
	antlr.ParseTreeWalkerDefault.Walk(&listen, p.Start())
	fmt.Println(listen.Pop())
}

func runVisitor() {
	// Setup the input
	is := antlr.NewInputStream("1 +     2 * 3+1+1")
	// Create the Lexer
	lexer := parser.NewCalcLexer(is)
	stream := antlr.NewCommonTokenStream(lexer, antlr.TokenDefaultChannel)

	// Create the Parser
	p := parser.NewCalcParser(stream)

	visitor := logic.CalVisitor{}
	// Finally parse the expression
	p.Start().Accept(&visitor)

	fmt.Println(visitor.Pop())
}
```

## 增加括号

这里我们完善规则，增加括号， 比如这里即会支持 `2*(1+2+3)`表达式。 
具体代码地址: https://github.com/l1905/antlr-demo


## 后续

这里只是入门相关知识，后续需要完善使用, 构造复杂的语法规则， 应用到具体业务中. 具体需要。

1. 强化使用
2. gui展示表达式

## 参考资料

1. https://github.com/dohkoos/antlr4-short-course
2. [antlr权威指南(中文翻译)]()
3. [ANTLR4笔记-01](https://abcdabcd987.com/notes-on-antlr4/) [ANTLR4笔记-02](https://abcdabcd987.com/using-antlr4/)
4. [计算机源码golang例子](https://github.com/thesues/antlr-calc-golang-example) 比较详细使用
5. [ANTLR电子书](https://wizardforcel.gitbooks.io/antlr4-short-course/basic-concept.html)
6. https://www.cnblogs.com/clonen/p/9083359.html




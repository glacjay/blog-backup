---
categories:
  - Compiler
comments: true
layout: post
published: true
status: publish
tags:
  - bison
  - compiler
title: "MiniJava 语法中的 non-LALR(1) 的部分及用 Bison 的解决方案"
type: post
---

我正在看[虎书](http://www.cs.princeton.edu/~appel/modern/java/)，在这本书中所给出的那个 [MiniJava](http://www.cambridge.org/us/features/052182060X/) 语言的语法并不是 LALR(1) 语法，因此在某些情况下，所生成的语法分析器会对正确的输入给出语法解析错误。不过 Bison 提供了一个简单而又强大的[解决方案](http://www.gnu.org/software/bison/manual/html_mono/bison.html#Generalized-LR-Parsing)，可以轻易的解决掉这个问题。

MiniJava 语法中的 non-LALR(1) 部分其实是一个 [shift/reduce 冲突](http://www.gnu.org/software/bison/manual/html_mono/bison.html#Shift_002fReduce)。具体来说就是，在 [MethodDeclaration](http://www.cambridge.org/us/features/052182060X/grammar.html#prod7) 中的 ([VarDeclaration](http://www.cambridge.org/us/features/052182060X/grammar.html#prod6))* 和 ([Statement](http://www.cambridge.org/us/features/052182060X/grammar.html#prod5))* 之间的状态下，当 Lookahead Token 是一个 IDENTIFIER 时，如果选择 shift ，那么下一步的动作就是进一步将这个 IDENTIFIER reduce 为一个 Type ，即将这个 IDENTIFIER 看作一个新的变量定义的开始；而如果选择 reduce 的话，实际上就是按照 (Statement)* 中的 epsilon 规则来 reduce ，即结束变量定义部分的解析，转而进入语句定义部分的解析。我们可以看到，仅仅依靠目前的信息，即 IDENTIFIER 这个 Lookahead Token ，是不足以决定接下来的动作的，因此 Bison 的默认策略，即 shift ，就会在某些情况下（这种情况其实很常见，就是当第一条语句是赋值语句时）产生错误的解析步骤。

这个问题其实很好解决，只要 Bison 能多向前看一个 Token 就可以了，可是 Bison 是 LALR(1) 语法分析器生成器，而不是 LALR(2) 。不过 Bison 提供了另外一条解决问题的途径，就是 GLR - Generalized LR Parsing 。简单来说，就是当 Bison 遇到一个冲突时，不管是 shift/reduce 冲突，还是 [reduce/reduce 冲突](http://www.gnu.org/software/bison/manual/html_mono/bison.html#Reduce_002fReduce)，就会将分析路径分为两条，分别跟进两种情况。当其中一条分析路径遇到语法错误，进行不下去时，就会自动消失。如果我们的文法是没有二义性的 LR 文法的话，最后就肯定可以得到正确的分析结果了。而我们的 MiniJava 的文法就是这种情况。说来好像复杂，但用起来其实很简单，只要在我们的 Bison 文法文件中加上 %glr-parser 这个选项就可以了。

不过奇怪的是，加上这个选项之后，我们就不用再自己写 YYSTYPE 和 YYLTYPE 的定义了，搞不懂，莫非是个 Bug ？

另外，本来我以为只要将变量定义语句也算到语句中的一种的话，这个问题也可以得到解决，不过后来发现我想错了，这样还是无法处理函数定义中的第一条语句是赋值语句的情况。未经证实，嗯，因为要改的地方还不少。

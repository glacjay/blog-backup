---
categories:
  - Compiler
comments: true
layout: post
published: true
status: publish
tags:
  - compiler
  - linux
  - ruby
title: "[翻译] 用 Ruby 写编译器之六：匿名函数 lambda"
type: post
---

原文链接：[http://www.hokstad.com/writing-a-compiler-in-ruby-bottom-up-step-6.html](http://www.hokstad.com/writing-a-compiler-in-ruby-bottom-up-step-6.html)

既然上次已经提到说，我们其实是在从 Lisp、Scheme 还有类似的其他语言中借鉴各种要实现的功能（我没想过要把这个项目做成原创的⋯⋯或者至少也要等到以后再说吧），那么现在也**是时候**实现一些更加强大的功能了。

## 那就来做延迟求值以及匿名函数吧

Lambda ，又名匿名函数，可以像普通的数值或者字符串类型那样被当作函数参数来到处传递，也可以在需要的时候才调用（当然不调也可以）。同时，外层函数（也就是定义匿名函数的函数）作为它们的运行环境，在其中定义的局部变量可以被这些匿名函数所访问。这就形成了[一个闭包](http://en.wikipedia.org/wiki/Closure_(computer_science\))。我们这次**并不是**要实现完整的闭包功能，只是开头的一小步而已，完整的实现要等到再后面了。

其实说到底，正如编程语言中的其他很多功能一样，闭包也只是又一种语法糖而已。比如说，你可以认为这样其实是定义了一个类，这个类中有唯一一个需要被调用的方法，还有一些作为运行环境的成员变量（或者你也可以反过来[用闭包来实现面向对象系统](http://strlen.com/bla/index.html) -- 这是由 [Wouter van Oortmerssen](http://strlen.com/) 所提出来的观点。自从我发现 [Amiga E](http://strlen.com/e/index.html) 这个项目之后，作为作者的他就成了我的偶像。如果你是一个编程语言方面的极客的话，那你就一定要去看看 Wouter 所做过的东西） -- 有很多功能其实都是相互正交的。

（ blah blah blah ⋯⋯这个人在说自己很啰嗦之类的，就不翻了）

那么，为了避免定义很多具名小函数的麻烦，同时降低函数重名的概率， lambda 允许你在任何需要它们的地方进行定义，并且返回一个表示所定义函数的值。

我们现在所要增加的是像下面这样的东西：

``` cl
(lambda (args) body)
(call f (args))
```

第一个表达式会返回所定义的函数（目前来说，其实就是这个函数的起始地址），而不是去执行这个函数。而 `call` ，当然就是以指定的参数列表，来调用传给它的那个函数地址。

那么，这跟“真正”的闭包又有什么区别呢？

最重要的一点就是，当你在 lambda 中引用了外层作用域中的某个变量后，那么，当以后对同一个 lambda 进行调用时，这个变量还是可以访问的。这样的变量与 lambda 绑定在了一起。当然，只是得到一个函数的地址的话，肯定是实现不了这个功能的。让我们来看一种实现闭包的技术吧，这样你就能够了解大概所要做的工作了（工作量不大，但也不小）：

- 我们可以创建一个“环境”，一个存放那些被引用到的外部变量的地方，以使得当外层函数返回之后，它们也可以继续存在。这个环境必须是在堆中，而且每次对外层函数的调用，都需要创建一个新的环境。
- 我们必须返回一个可以用来访问这个环境的东西。你可以返回一个对象，其中的成员变量就是被 lambda 引用到的那些变量。或者你也可以用一个 thunk （中文叫啥呢？），也就是自动生成出来的一个包含有指向这个对象指针的小函数，它会在调用我们的 lambda 之前将这个对象加载到预先指定的地方。或者你也可以用什么其他的办法。
- 你必须决定有哪些变量需要放入这个环境中。可以是外层函数中的所有局部变量，当然也可以只是那些被引用到的变量。后一种作法可以节省一定的内存空间，但是需要作更多的工作。

好吧，还是让我们先来把匿名函数本身给弄出来吧。就像以前一样，我会一步一步地说明所要做的修改，但我同时也会对之前的代码做一些整理。这些整理的部分我就不一一说明了。

首先是对 lambda 表达式本身进行处理的方法：

``` ruby
  def compile_lambda args, body
    name = "lambda__#{@seq}"
    @seq += 1
    compile_defun(name, args,body)
    puts "\tmovl\t$#{name},%eax"
    return [:subexpr]
  end
```

这个实现应该是很容易理解的吧。我们在这里做的，就是给要定义的匿名函数，生成一个形如 `lambda__[number]` 的函数名，然后就把它当作一个普通函数来处理了。虽然你也可以就把它生成到外层函数的函数体中，但我发现那样做的话，就会显得很乱的样子，所以我现在还是就把它作为单独的函数来处理了。然后我们调用 `#compile_defun` 方法来处理这个函数，这样的话，这个函数其实也就只有对用户来说，才是真正匿名的了。然后我们把这个函数的地址保存在寄存器 `%eax` 中，这里同时也是我们存放子表达式结果的地方。当然，这是一种很懒的作法啦，我们迟早需要为复杂的表达式，实现更加强大的处理机制的。但寄存器分配果然还是很麻烦的一件事情，所以现在就先这样吧（将所有的中间结果压入栈中也是一种可行的作法啦，不过那样比较慢）。

最后，我们返回一个 `[:subexpr]` ，来告拆调用者到哪边可以得到这个 lambda 的值。

之后是一些重构。你也许已经注意到了， `#compile_exp` 中的代码有点乱，因为要处理不同类型的参数。让我们把这部分代码给提取出来：

``` ruby
  def compile_eval_arg arg
    atype, aparam = get_arg(arg)
    return "$.LC#{aparam}" if atype == :strconst
    return "$#{aparam}" if atype == :int
    return aparam.to_s if atype == :atom
    return "%eax"
  end
```

要注意的是，这里又出现了一个新的 `:atom` 类型。借助于此，我们就可以把一个 C 函数的地址传给 `:call` 指令了。反正实现起来也很简单。当然，我们还要在 `#get_arg` 方法中加上如下代码，以使其生效：

``` ruby
    return [:atom, a] if (a.is_a?(Symbol))
```

然后，作为重构的一部分，对 `:call` 的处理被分离了出来，成为一个单独的方法：

``` ruby
  def compile_call func, args
    stack_adjustment = PTR_SIZE + (((args.length+0.5)*PTR_SIZE/(4.0*PTR_SIZE)).round) * (4*PTR_SIZE)
    puts "\tsubl\t$#{stack_adjustment}, %esp"
    args.each_with_index do |a,i|
      param = compile_eval_arg(a)
      puts "\tmovl\t#{param}, #{i>0 ? i*4 : ""}(%esp)"
    end

    res = compile_eval_arg(func)
    res = "*%eax" if res == "%eax" # Ugly. Would be nicer if retain some knowledge of what res contains.
    puts "\tcall\t#{res}"
    puts "\taddl\t#{stack_adjustment}, %esp"
    return [:subexpr]
  end
```

看起来很熟悉对不对？因为它就是原来的 `#compile_exp` 方法，只不过是用 `#compile_eval_arg` 替换掉了其中的一些代码。另外有改动的地方，就是同时也用 `#compile_eval_arg` 方法来得到要调用的函数，并对可能得到的 `%eax` 做了些手脚，在前面加了个星号。

如果你知道这是怎么回事的话，你也许已经开始寻思其他的点子了，而不管那是真正的好事，还只是开枪打自己的脚。上面的代码其实就相当于，你把任意一个表达式的值当作指向一段代码的指针，然后也不做任何的检查，就直接跳过去执行它。如果是往一个随机的地址进行跳转的话，你最有可能得到的就是一个段错误了。当然，你也可以很容易地通过这项技术，来实现面向对象系统中的虚函数跳转表，或者其他的什么东西。因此，安全性将会是以后必须要考虑的东西。还有就是，要实现对一个地址（而不是函数名）的间接调用，你必须要在这个地址前面加上星号。

那么，现在的 `#compile_exp` 方法变成什么样子了呢？简单来说就是，变得整齐多了：

``` ruby
  def compile_do(*exp)
    exp.each { |e| compile_exp(e) }
    return [:subexpr]
  end
  def compile_exp(exp)
    return if !exp || exp.size == 0
    return compile_do(*exp[1..-1]) if exp[0] == :do
    return compile_defun(*exp[1..-1]) if exp[0] == :defun
    return compile_ifelse(*exp[1..-1]) if exp[0] == :if
    return compile_lambda(*exp[1..-1]) if exp[0] == :lambda
    return compile_call(exp[1], exp[2]) if exp[0] == :call
    return compile_call(exp[0], *exp[1..-1])
  end
```

看起来很不错，不是吗？`#compile_call` 几乎跟之前的 `#compile_exp` 一模一样，只不过是把一些代码给提取出来，成为了辅助方法而已。

那么就来简单地测试一下吧：

``` ruby
prog = [:do,
  [:call, [:lambda, [], [:puts, "Test"]], [] ]
]
```

（看起来也没那么糟不是吗？）

编译运行之：

``` plain
$ ruby step6.rb >step6.s
$ make step6
cc      step6.s  -o step6
$ ./step6
Test
$
```

这篇的代码在[这里](http://www.hokstad.com/static/compiler/step6.rb)。

## 之后的部分

因为我合并了几篇文章，下面就列一下更新过后的，“已经写完但还需要整理的”文章列表。因为我肯定还会合并下面的某些部分的，所以我想我需要找时间开始写一些新的部分了（为了完成我所定下的 30 篇的计划）。

- 步骤七：再看用匿名函数实现循环，以及对函数参数的访问
- 步骤八：实现赋值语句以及简单的四则运算
- 步骤九：一个更简洁的 `while` 循环语句
- 步骤十：测试这个语言：实现一个简单的输入转换模块
- 步骤十一：重构代码生成器，分离架构相关的部分
- 步骤十二：对某些功能点的讨论，以及未来的前进方向
- 步骤十三：实现数组
- 步骤十四：局部变量以及多重作用域
- 步骤十五：访问变长参数列表
- 步骤十六：再看输入转换模块，重构以支持新功能，并用其解析它自己
- 步骤十七：总结实现自举所需要的功能点
- 步骤十八：开始实现真正的解析器

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
title: "[翻译] 用 Ruby 写编译器之二：函数调用，以及 Hello World"
type: post
---

原文地址：[http://www.hokstad.com/writing-a-compiler-in-ruby-bottom-up-step-2.html](http://www.hokstad.com/writing-a-compiler-in-ruby-bottom-up-step-2.html)

我会选择 Ruby 来作为我的实现语言并没有什么特别的理由。在现阶段，语言的选择并不重要；不过，我确实很喜欢 Ruby。

在这之后，我会采取一系列的步骤令所实现的语言向其实现语言靠拢。我的意思是，我想将编译器实现为可以**自举**的，即它应该能够编译自身。

而这也就意味着，要么我的编译器需要至少支持 Ruby 语言的一个子集，要么就需要一个中间的翻译步骤，来将编译器中的实现翻译成它自己可以编译的语言。

虽然这一点并没有限制你所用的实现语言，但那至少意味着你用来实现的语言跟你要实现的语言之间很相似，除非你对实现编译器的自举没有什么兴趣。

这也同时意味着，如果你想实现编译器的自举，那你最好不要用实现语言中的什么复杂的特性。要记住，你得实现所有你用过的那些语言特性，不然，当你要开始做自举的时候，你就得对整个编译器的架构做大的调整了。那一点都不好玩。

话说回来，使用 Ruby 来作为实现语言的一大优点（同样对于其他某些语言来说也是这样，比如说 Lisp ），就是你可以很容易地构建出一个树形的数据结构出来 -- Ruby 的话就是用数组或者哈希， Lisp 的话就是用列表。

这也就是说，我可以用数组来手工构建抽象语法树，从而避免了实现一个语法分析器的工作。耶！代价就是很丑，不过也可以接受的语法啦。

## Hello World

Hello World 的话看起来会是这个样子的：

``` ruby
[:puts, "Hello World"]
```

这里，我们需要处理的东西非常简单：我们需要将一个参数压入堆栈，然后调用一个函数。

那么就让我们来看一下怎样用 x86 汇编来做这件事吧。我用 `gcc -S` 编译了下面的这段 C 程序：

``` c
int main()
{
    puts("Hello World");
}
```

然后看看输出会是什么样子的。下面给出的是真正相关的部分，是与上一次的输出比较之后的结果：

``` nasm
        .section        .rodata
.LC0:
        .string "Hello World"
        .text
...
        subl    $4, %esp
        movl    $.LC0, (%esp)
        call    puts
        addl    $4, %esp
...
```

如果你懂一些汇编的话，就算以前没有写过 x86 的汇编程序，也应该可以很容易的看懂这段代码吧：

- 这里首先定义了一个字符串常量。
- 通过对堆栈指针的减 `4` 操作，在堆栈上申请了一段 `4` 个字节大小的空间。
- 然后将之前定义的字符串常量的地址，放入刚刚申请的那 `4` 个字节的空间中。
- 接着，我们调用了由 glibc 提供的 `puts` 函数（在这个系列中，我会假设你已经有了 gcc/gas + glibc ；Linux 的话这些东东应该已经有了）。
- 最后，通过一个加 `4` 操作来释放堆栈空间。

那么，我们要怎样在我们的编译器中实现这一切呢？首先，我们需要一种方法来处理那些字符串常量，通过在上次的 Compiler 类的实现中添加下面的代码（我的所有 Ruby 代码都在[这里](http://www.hokstad.com/static/compiler/step2.rb)，这样你就知道该做什么了）：

``` ruby
  def initialize
    @string_constants = {}
    @seq = 0
  end
  def get_arg(a)
    # For now we assume strings only
    seq = @string_constants[a]
    return seq if seq
    seq = @seq
    @seq += 1
    @string_constants[a] = seq
    return seq
  end
```

这段代码就是简单地将一个字符串常量映射到一个整数上，而这个整数则对应着一个标号。相同的字符串常量会对应到相同的整数上，因此也只会被输出一次。用哈希而不是数组来保证这种唯一性是一种很常用的优化手段，不过也不一定非要这样做。

下面这个函数是用来输出所有的字符串常量的：

``` ruby
  def output_constants
    puts "\t.section\t.rodata"
    @string_constants.each do |c,seq|
      puts ".LC#{seq}:"
      puts "\t.string \"#{c}\""
    end
  end
```

最后剩下的就是编译函数调用的代码了：

``` ruby
  def compile_exp(exp)
    call = exp[0].to_s
    args = exp[1..-1].collect {|a| get_arg(a)}

    puts "\tsubl\t$4,%esp"

    args.each do |a|
      puts "\tmovl\t$.LC#{a},(%esp)"
    end

    puts "\tcall\t#{call}"
    puts "\taddl\t$4, %esp"
  end
```

也许你已经注意到这里的不一致性了：上面的代码虽然好像是可以处理多参数调用的样子，但却只从堆栈中减掉了一个 `4` ，而不是按照实际的参数个数而进行相应的调整，从而导致了不同参数间的相互覆盖。

我们马上就会处理这个问题的。对于我们简单的 Hello World 程序来说，目前这样已经足够了。

在这段代码中还有几点需要注意：

- 我们甚至都还没有检查被调用的函数到底存不存在 -- gcc/gas 会帮我们处理这个问题的，虽然这也意味着没啥帮助的错误信息。
- 我们可以调用任何一个可以连接的函数，只要这个函数只需一个字符串作为参数。
- 这段代码目前还有很多需要被抽像出去的地方，比如说得到被调函数地址的方法，还有所有那些硬编码进来的 x86 汇编等。相信我，我会（慢慢）解决这些问题的。

现在我们可以来[试着运行一下这个编译器](http://www.hokstad.com/static/compiler/step2.rb)了。你应该会得到下面这样的输出：

``` nasm
        .text
.globl main
        .type   main, @function
main:
        leal    4(%esp), %ecx
        andl    $-16, %esp
        pushl   -4(%ecx)
        pushl   %ebp
        movl    %esp, %ebp
        pushl   %ecx
        subl    $4,%esp
        movl    $.LC0,(%esp)
        call    puts
        addl    $4, %esp
        popl    %ecx
        popl    %ebp
        leal    -4(%ecx), %esp
        ret
        .size   main, .-main
        .section        .rodata
.LC0:
        .string "Hello World"
```

下面来测试一下：

``` plain
[vidarh@dev compiler]$ ruby step2.rb >hello.s
[vidarh@dev compiler]$ gcc -o hello hello.s
[vidarh@dev compiler]$ ./hello
Hello World
[vidarh@dev compiler]$
```

## 那么，要怎么处理多个参数的情况呢？

我不会再展示用来说明的 C 代码和对应的汇编代码了 -- 进行不同参数个数的调用并查看其输出对你来说应该不难。相反，我就直接给出对 compile_exp 函数所做的修改了（完整的代码在[这里](http://www.hokstad.com/static/compiler/step2b.rb)）：

``` ruby
  PTR_SIZE=4
  def compile_exp(exp)
    call = exp[0].to_s

    args = exp[1..-1].collect {|a| get_arg(a)}

    # gcc on i386 does 4 bytes regardless of arguments, and then
    # jumps up 16 at a time, We will blindly do the same.
    stack_adjustment = PTR_SIZE + (((args.length+0.5)*PTR_SIZE/(4.0*PTR_SIZE)).round) * (4*PTR_SIZE)
    puts "\tsubl\t$#{stack_adjustment}, %esp"
    args.each_with_index do |a,i|
      puts "\tmovl\t$.LC#{a},#{i>0 ? i*PTR_SIZE : ""}(%esp)"
    end

    puts "\tcall\t#{call}"
    puts "\taddl\t$#{stack_adjustment}, %esp"
  end
```

这里做了什么呢？改动的地方没几个：

- 这里不再是申请固定大小的堆栈空间了（上一个版本中是 `4` 个字节），而是根据实际参数的个数来相应的调整堆栈指针。我得承认，我不知道 gcc 为什么会做这样的调整 -- 而且原因并不重要，虽然我猜这是为了堆栈的对齐。优化和清理**以后**再说，并且，当你不知道某事的运行机理时，那就不要去改变它。
- 这之后，如你所见，参数被一个一个地放到堆栈上了。我们还是假定它们全都是相同大小的指针（因此在 x86 上就是 `4` 个字节）。
- 同时你还可以看到，第一个参数是被放在堆栈中最靠下的位置的。如果你还没有写过汇编程序，并且无法想象出这是怎么回事的话，那就把它们画出来吧；还要记住，这里的堆栈是向下扩展的。当申请空间时，我们是将堆栈指针向下移动的，而拷贝参数时则是从下往上（用越来越大的索引来访问 `%esp` ，就像你访问数组时一样）。

这个编译器现在已经可以编译下面这样的代码了：

``` ruby
[:printf,"Hello %s\n","World"]
```

## 至于以后嘛

这就是我们踏出的第一步，而且我保证之后的步骤会越来越实际的，因为只要实现很少的几个功能点，我们就可以编译实际的程序了。而且我会努力令这些步骤更加精炼，更多的说明这样做的原因，而不是仅仅解释做了什么。

下面我会处理多个参数的调用（译者：我们不是才处理过嘛），然后是语句序列、子表达式，以及对返回值的处理，等等。

大约十二个这样难度的步骤之后，我们就会完成函数定义、参数传递、条件判断、运行时库，甚至是一个用来实现匿名函数的简单的 lambda （真正的[闭包](http://en.wikipedia.org/wiki/Closure_(computer_science\))就要到后面了）。

再之后，我们会实现一个简单的文本处理程序，来对一个比 Ruby 的数组和符号更好一点的语法提供支持（只是某种程度啦，真正的语法分析得再多等等）。

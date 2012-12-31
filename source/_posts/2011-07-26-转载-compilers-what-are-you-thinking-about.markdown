---
categories:
  - Compiler
comments: true
layout: post
published: true
status: publish
tags:
  - compiler
title: "[转载] Compilers: what are you thinking about?"
type: post
---

最近在翻看以前加星的 Google Reader 文章，把有用的整理出来打标签，然后就看到了这篇。原文作者的博客现在不知道被丢到哪边去了，搜也搜不到，转到这里，权当保存一下吧。

原文链接：[Compilers: what are you thinking about?](http://www.onebadseed.com/blog/?p=119)。当然，已经打不开了。

-----

## Compilers: what are you thinking about?

Author: Rotten Cotton

My recent post [Compiler bibliography](http://www.onebadseed.com/blog/?p=103), a motley list of compiler papers that had been sitting in a box in my attic, generated a surprising amount of traffic: over a thousand unique visitors. But no comments. For those of you interested in compiler design, I ask, what are you trying to understand? What are you trying to do?

I imagine some of you are interested in understanding how compilers work and, more importantly, how to build them. The first step in learning to design compilers is to build one. (Tautologous, no?) To start, I recommend following the syllabus of a compiler design course. Googling “[compiler syllabus](http://www.google.com/search?hl=en&q=compiler+syllabus&aq=f&oq=&aqi=g1)” returns a massive list of syllabi for compiler for compiler classes. The first link, a [compiler design](http://web.cecs.pdx.edu/~harry/compilers/syllabus.html) class, at Portand State University, follows Louden’s book [Compiler Construction](http://www.amazon.com/gp/product/0534939724?ie=UTF8&tag=wwwonebadseec-20&linkCode=as2&camp=1789&creative=390957&creativeASIN=0534939724) to build a SPARC compiler for a toy programming language, [PCAT](http://web.cecs.pdx.edu/~harry/compilers/PCATLangSpec.pdf) (pdf). MIT’s compiler design class, [6.035](http://ocw.mit.edu/OcwWeb/Electrical-Engineering-and-Computer-Science/6-035Fall-2005/CourseHome/), is on [OCW](http://ocw.mit.edu/). Following one of these classes will teach you the basic challenges involved in designing a compiler and the organizational principles that have emerged to solve these problems. If you’re ambitious, start with a free C front end and build a C compiler for your favorite architecture. There is [ckit](http://www.smlnj.org/doc/ckit/index.html) for ML, [Language.C](http://www.sivity.net/projects/language.c/) for Haskell or [clang](http://clang.llvm.org/) (part of the [LLVM](http://llvm.org/) project) written in, I think, C++. There are others, no doubt. This way, you won’t have to write a front-end and you will have a huge source of potential examples on which to test your compiler and, when your start to design optimizations (the fun part), explore your compiler’s performance.

For a few years after I first took 6.035, whenever the course was taught again and the project (toy language) for that semester was announced, I would spend a long caffeine-fueled weekend hacking out a new compiler. I must have build three or four compilers this way, targeting different architectures and exploring various design choices. This was a very valuable experience.

Once you understand the basics of compiler design, what’s next? There are a number of directions you can go.

You can study programming language features and their implementation in a compiler and runtime. For this, I recommend a book like Scott’s [Programming Language Pragmatics](http://www.amazon.com/gp/product/0123745144?ie=UTF8&tag=wwwonebadseec-20&linkCode=as2&camp=1789&creative=390957&creativeASIN=0123745144) or Turbak and Gifford’s [Design Concepts in Programming Languages](http://www.amazon.com/gp/product/0262201755?ie=UTF8&tag=wwwonebadseec-20&linkCode=as2&camp=1789&creative=390957&creativeASIN=0262201755). I used a draft version of the latter in MIT’s Programming Languages class, 6.821. This will move beyond compiling the standard C- or Pascal-like toy languages standard in most first-year compiler courses. Learn about semantics: denotational, operational and axiomatic. This is important if you (a) want to build a correct compiler, and (b) as a theoretical foundation for program analysis.

At the other end of the spectrum are computer architectures. Go learn about computer architecture: instruction set architectures and micro-architectures, their implementation. There are many interesting architectures out there to study. Write assembly language programs by hand. Learn to extract maximal performance form an architecture on small examples. Pay attention to the techniques used to extract performance when hand-coding. These will be the basis for compiler optimizations. The standard text for computer architecture is probably Hennessy and Patterson’s text, [Computer Architecture: A Quantitative Approach](http://www.amazon.com/gp/product/0123704901?ie=UTF8&tag=wwwonebadseec-20&linkCode=as2&camp=1789&creative=390957&creativeASIN=0123704901).

A deep understanding of both the semantics of your input programming language and your target (micro-)architectures is essential to building a compiler that generates high-performance code. Program analysis and optimization is the bridge between the input program and assembly language. Start with a book on advanced compiler design like Muchnick’s [Advanced Compiler Design and Implementation](http://www.amazon.com/gp/product/1558603204?ie=UTF8&tag=wwwonebadseec-20&linkCode=as2&camp=1789&creative=390957&creativeASIN=1558603204) (or maybe the new dragon book, I haven’t looked at it) or start diving into the literature or program analysis and optimization.

What are you trying to understand about compilers?

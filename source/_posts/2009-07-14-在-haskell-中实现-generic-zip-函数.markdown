---
categories:
  - Programming
comments: true
layout: post
published: true
status: publish
tags:
  - haskell
title: "在 Haskell 中实现 Generic zip 函数"
type: post
---

其实嗫，这个问题已经有标准和其他的解决方案了。标准解决方案参见 `Control.Applicative` 中的 `ZipList` ，不过这东东用起来蛮麻烦的说；其他解决方案见 bff 库的 `Data.Zippable` 模块，嗯，我还没搞明白这玩意怎么用，不过总感觉杀鸡用牛刀了有点（Template Haskell ，以及其他依赖）。

所以，如果你只是跟我一样，看 `Data.List` 中的那一砣 `zipn` 不顺眼的话（其实也只是看着不顺哈，用着还是蛮顺的，反正实现不用我写），一个更简单的方案在此：

``` haskell
> z :: [a -> b] -> [a] -> [b]
> z = zipWith ($)
```

吼吼，够简单的吧。其实这跟 `Control.Monad` 中的 `ap` 和 `Control.Applicative` 中的是一类东东啦，只不过是针对列表的 `zip` 功能滴。

那要怎么用嗫，也不是很麻烦啦，像这样就可以了：

```
*Main> (,) `map` [1,2,3] `z` "abc"
[(1,'a'),(2,'b'),(3,'c')]
*Main> (,,) `map` [1,2,3] `z` "abc" `z` [Nothing, Just False, Just True]
[(1,'a',Nothing),(2,'b',Just False),(3,'c',Just True)]
```

这里的 `map` 所对应的自然就是 `Control.Applicative` 中的啦。

还有更好玩的哦，如果再加上[上一篇博](/blog/2009-05-06/haskell-%E4%B8%AD%E7%9A%84%E5%8F%AF%E5%8F%98%E9%95%BF%E5%8F%82%E6%95%B0%E5%88%97%E8%A1%A8.html)中的不定参函数的话呢：

```
*Main> buildList `map` [1,2,3] `z` [4,5,6] :: [[Int]]
[[1,4],[2,5],[3,6]]
*Main> buildList `map` [1,2,3] `z` [4,5,6] `z` [7,8,9] :: [[Int]]
[[1,4,7],[2,5,8],[3,6,9]]
```

那我这次要说得就这么多啦，至于怎么用，就请大家尽情地发挥你们的想象力吧（其实是我想象力不够 :-(）

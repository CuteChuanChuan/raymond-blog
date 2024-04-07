+++ 
draft = false
date = 2024-04-10T21:10:40+08:00
title = "Scala Learning Note - Chapter 1 (Basic)"
description = ""
slug = ""
authors = []
tags = ["Scala"]
categories = []
externalLink = ""
series = []
+++

# 前言
Scala 可以說是一個蠻神秘但是又非常有趣的語言。例如：(1) Scala 結合了 OOP (object-oriented programming) 和 FP (functional programming)；(2) Scala 可以跑在 JVM 上，但是又有 Java 沒有功能的特色；(3) 當要處理大數據時，Scala 可以帶來很大的幫助 (像是 Apache Spark 正是由 Scala 寫成的)  
然而，相較於其他程式語言 (e.g., Python & Javascript)，網路上有關 Scala 的教學與資料數量少很多，中文的相關資料又更少。因此接下來，我會透過多篇文章來介紹 Scala 給有興趣的人。  
預計會有四篇： Basic, OOP, FP, 和 Pattern Matching，而本篇即為介紹 Scala 的第一篇文章。

# Basic features and concepts of Scala
- 在 Scala 中，所有東西都是 expression：不論如何，程式碼一定會有一個回傳值

```scala
println("Hello, World")
val x = 2
```

{{< highlight Scala "linenos=table" >}}


# some code
echo "Hello World"
{{< /highlight >}}

- 
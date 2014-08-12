---
layout: post
title: "Gradle DSL X Groovy"
description: "关于解读Gradle DSL的一些想法"
category: 
tags: [Gradle, Groovy, Gradle DSL]
---
{% include JB/setup %}
一个常用Python和Java的人可能会在最初接手Gradle的一刹那感到困惑。如果完全没有Groovy的知识，正常人会觉得手头上的build.gradle是一个怪异的标记型语言。如果碰巧你又是个前端开发者，说不定会有一种HTML嵌Javascript的幻觉。

{% highlight groovy %}

task helloWorld {
	ext.world="world"
	println "it's a configure closure"
}

helloWorld << {
	println "hello $world"
	println "it's an action closure"
}
{% endhighlight %}

事实上，某有点半调子lisp基础的人一开始完全没有发现Gradle DSL竟然和lisp非常相似。

我猜一些拿Gradle当Maven用的人应该就是觉得Gradle DSL是一种嵌入可执行脚本的，插件形式比较小清新的东西而已。当然，会有人一直以为Maven就是个Jar的下载管理器。

_但是_ Gradle DSL 就是一种硬正（南京话EN ZEN!)的图灵完全语言，其几乎等价于Groovy，只是通过Groovy所支持的机制加了若干“语法”糖而已。甚至，如果直说Gradle DSL script就是Groovy script，那其实也是不能算错的。

而Groovy的灵活性正是Gradle的真正卖点，这绝不是一小措图灵不全的标记语言所能比拟的。任何说Gradle DSL是标记语言，或一部为标记一部为过程语言的，以及是两种语言的混合的反动观点都是某所反对的。

Gradle的文档很不错，但对于Groovy和Gradle DSL的关系，可能是出于不解释让你用用你就明白的心态，并不直接详说。然而如果Groovy道行不深（Groovy的主页域名都不舍得买，你说多少人道行会深！），错把Gradle DSL看做一门掺混了Groovy的新语言去学，那是会浪费时间和精力的。

要说手上的`build.gradle`为什么就好比 Python/Java8/lisp 的东西，以及为什么是Groovy而不是Python（神一样的lisp凡人就不要用了），那么就得知道Groovy和Gradle的几个性质，其实某初接触Groovy时，还曾觉得其中一个相当坑爹…

<!--more-->

## Gradle DSL/Groovy的几个性质

这些性质就是：

+ [非常方便的closure](http://groovy.codehaus.org/Closures)：`{}`中的代码就是个closure，类似于Java 8中的`()->{}`
+ [调用函数时可省略括号](http://groovy.codehaus.org/Statements#Statements-Optionalparenthesis)：`functionName arg`等同于`functionName(arg)`
+ [AST Transform](http://groovy.codehaus.org/Operator+Overloading)：一种标准的编译时的元编程机制
  + 加入语法糖[(相关讨论在此)](http://groovy.329449.n5.nabble.com/Explain-Gradle-DSL-for-declaring-tasks-tp5715160.html)：可以使得Gradle能够将DSL文件中的`task newTask`在预先转换成类似`tasks.create("newTask")`的形式后再交矛Groovy编译
  + [delegate transformation](http://groovy.codehaus.org/Delegate+transformation)：可以结省大量的delegate boilerplate代码。Gradle非常依赖delegate机制，使得使用者可以节省大量“笔墨”。（想想Java的情况吧……）
+ [运算符重载](http://groovy.codehaus.org/Operator+Overloading)：最常见的是重载`<<`用于将一个closure 右值作为task action closure加入task中。

以上四个性质就是解读Gradle DSL的最基本要素，除了AST Transform外，一个有Python/Java基础能够在相当短的时间里看懂。其实看个大概就行了，只要记住DSL即是Groovy就行了。

## Gradle DSL to 好读一点的Groovy
下面解析，先回顾一下
{% highlight groovy %}
task helloWorld {
	ext.world="world"
	println "it's a configure closure"
}

{% endhighlight %}

应用 _AST_ ，Gradle把task语法糖化为
{% highlight groovy %}

tasks.create("helloWorld")
helloWorld {
	ext.world="world"
	println "it's a configure closure"
}

{% endhighlight %}

这里需要指明的就是`{}`所成的闭包的地位，其实这是应用了 _函数小括号省略_ 的结果，等同于
{% highlight groovy %}

tasks.create("helloWorld")
helloWorld({
	ext.world="world"
	println("it's a configure closure")
})

{% endhighlight %}
这就是我曾觉得点坑的Groovy特性，起初只觉得容易产生岐义，但是，当你发现打完闭包不用再用费一对`()`括上的时候，那个爽感，Lisp大神是不屑一顾的……


Gradle还应用Groovy的 _delegate_ 机制将代表Gradle项目的project和project.tasks下边的对像都暴露到了全局访问域了。
于是有了以下的assert+同义代码

{% highlight groovy %}

assert tasks == project.tasks

tasks.create("helloWorld")
tasks.helloWorld({
	ext.world="world"
	println("it's  configure closure")
})

{% endhighlight %}

最后应用一下`<<` _运算符重载_ ，由此本文开头的那个hello world等价于
{% highlight groovy %}

assert project.tasks == tasks

tasks.create("helloWorld")
assert helloWorld == tasks.helloWorld

helloWorld({
	ext.world="world"
	println "it's a configure closure"
})

helloWorld.leftShift({
	println "hello $world"
	println "it's an action closure"
})
{% endhighlight %}

注意`helloWorld.leftShift`实际等价于`helloWorld.doLast`

由此，Gradle DSL就翻成了标准的Groovy，童叟无欺。Gradle在运行时就是顺序运行相应的`.gradle`脚本，就像Groovy/Python运行一个脚本一样。

## 后吐槽 ps
Groovy相比于Python（Java就算了，不用比了），值得注意的几个不同点是

+ 更方便围着closure转
+ 更方便加编译前处理的元编程操作
  + 特别是更方便的delegate机制

这其中，Python当初不用括号而用缩进使得其closure机制不好搞是个硬伤。因此对于这种config script总会比Groovy看上去烦。

做为一个通用语言，其实我觉得Groovy的竞争力是不如Python的。由于jvm的机制不同于Python，特别是jdk 7之前的byte code specification连动态类型的支持都欠点，造成了Groovy的很多先天不足，为了代码简洁而堆上的一堆特性又影响了代码的可读情（为什么会有ext这个东西，有了this不够还得要owner），而运行时的灵活性又不如Python，这使得靠直接学习和上手Groovy的同学的热情容易被打击。

同样是亮点不少，缺点不小的Javascript在G家V8的推动下早已突起，历史留给这个没有顶级域名的庶子Groovy的空间估计有限吧。

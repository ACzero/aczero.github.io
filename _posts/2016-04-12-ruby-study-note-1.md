---
layout: post
title:  "Ruby基础学习小记(一)"
date:   2016-04-12 21:30:00 +0800
categories: ruby
---

### 1.单引号 vs 双引号
双引号中可以使用`#{}`进行字符串插入，且会对特殊字符串如`\n`进行转义。单引号没有上述功能。

对于双引号（非插值字符串）与单引号间的性能问题，如`"string"`和`'string'`。在ruby会解析成相同的类型，因此在运行的时候性能是完全一样的（忽略其它因素），只有在解析阶段不同。详细可以参考[这里](http://stackoverflow.com/questions/1836467/is-there-a-performance-gain-in-using-single-quotes-vs-double-quotes-in-ruby#answer-1836838)

#### best practice:
不推荐在字符串中带上转义字符串，像这样:

```Ruby
"#{name} said: \"Clap your hands!\""
```
Ruby提供了[构建这类字符串的方法](https://en.wikibooks.org/wiki/Ruby_Programming/Syntax/Literals#The_.25_Notation)。上面这个带`"`的字符串就可以这样写:

```Ruby
%Q[#{name} said "Clap your hands!"]
```

界定符可以任意非字母和数字的符号

如果你不喜欢上面这种方法，可以用[Here document](https://en.wikibooks.org/wiki/Ruby_Programming/Syntax/Literals#.22Here_document.22_notation)来构建字符串。

代码风格看[这里](https://github.com/bbatsov/ruby-style-guide/#strings)

#### 参考链接:
* http://rors.org/2008/10/26/dont-escape-in-strings
* http://stackoverflow.com/questions/1836467/is-there-a-performance-gain-in-using-single-quotes-vs-double-quotes-in-ruby
* https://en.wikibooks.org/wiki/Ruby_Programming/Syntax/Literals#The_.25_Notation
* http://ruby-doc.org/core-2.0.0/doc/syntax/literals_rdoc.html#label-Percent+Strings


### 2.参数表

```Ruby
# 定义了接受不定数量的参数，形参nums是一个数组
def foo1(*nums)
end

# 定义了hash形参
def foo2(options = {})
end

# 要把array作为多个参数传递，可以这样写
arr = [1,2,3]
foo1(*arr)
```

这个[question](https://rubymonk.com/learning/books/1-ruby-primer/chapters/19-ruby-methods/lessons/69-new-lesson#210)的答案写得很简洁

### 3.Array#sort

[文档](http://apidock.com/ruby/Array/sort)上Array#sort一般这样用：

```ruby
a = [ "d", "a", "e", "c", "b" ]
a.sort {|x,y| y <=> x }   #=> ["e", "d", "c", "b", "a"]
```

`<=>`运算符的作用为：对于`a <=> b`，`a < b`返回`-1`； `a == b`返回`0`；`a > b`返回`1`。

再来看看sort方法到底做了什么，下面为[Ruby源码](https://github.com/ruby/ruby/blob/280f7322151655b23c11a673b1adaab8796a358f/array.c#L2439)中的注释：

```c
/*
 *  call-seq:
 *     ary.sort!                   -> ary
 *     ary.sort! { |a, b| block }  -> ary
 *
 *  Sorts +self+ in place.
 *
 *  Comparisons for the sort will be done using the <code><=></code> operator
 *  or using an optional code block.
 *
 *  The block must implement a comparison between +a+ and +b+ and return
 *  an integer less than 0 when +b+ follows +a+, +0+ when +a+ and +b+
 *  are equivalent, or an integer greater than 0 when +a+ follows +b+.
 *
 *  See also Enumerable#sort_by.
 *
 *     a = [ "d", "a", "e", "c", "b" ]
 *     a.sort!                    #=> ["a", "b", "c", "d", "e"]
 *     a.sort! { |x,y| y <=> x }  #=> ["e", "d", "c", "b", "a"]
 */
```

当block中返回负整数时，a会排在b前面，返回正整数时，b会排在a前面。

所以Array#sort的block不是必须用`<=>`运算符的。

### 4.Parallel Assignment
Parallel Assignment有一个[陷阱](http://stackoverflow.com/questions/2895957/parallel-assignment-operator-in-ruby/2895989)。

看看下面的例子就会明白：

```
a = 1
b = 1
a, b = b + 1, a + 1		#=> [2, 2]

a = 1
b = 1
a = b + 1		#=> 2
b = a + 1 		#=> 3
```

原因就是Parallel Assignment会先计算右边的值后再进行赋值，不然怎么叫`Parallel`?

### 5.block，proc和lambda

#### (1)block和proc、lambda的区别
block跟proc、lambda最大的区别是，block不是对象，而是ruby中的语法，而proc和lambda都是对象。block在传递给函数的时候：

```ruby
def foo(a)
	puts a
	puts yield
end

foo("param") { "hello world" }	#=> "param"
								#=> "hello world"  
```
block是隐式传递给函数的，并不在参数表中，因此也无法对这个block做调用以外的操作（block本来就不是对象），只能在函数里面通过yield调用。

#### (2)block与proc、lambda间的转换（语法上）
这样写其实有误导，block不是对象，自然不存在block和proc、lambda的“转换”这种事。不过ruby在语法上是可以转换的，具体是这样的：

```ruby
def foo(&block)
  puts block.call
end

foo { "hello world" }	#=> "hello world"
```

把`&param`作为参数表最后一个参数，这样写就能操作隐式传入的block，不过实际这个是一个proc：

```ruby
def foo(&block)
  puts block.class
end

foo { "hello world" }	#=> proc
```

而将proc或lambda转换成block，则可以通过`&`实现，不过在传递的时候需要显式传递：

```ruby
def foo1
	puts yield
end

a = Proc.new { "proc" }
b = lambda { "lambda" }

#这里实际上是显式传递
foo1 &a	#=> "proc"
foo1 &b	#=> "lambda"

#需要显式传递
def foo2(param)
	puts yield
end

foo2("param", &a) => "proc"
```

#### (3)proc与lambda的区别
proc与lambda主要有两点不同。
proc处理参数上更灵活：

```ruby
a = Proc.new do |a,b|
  puts "a:#{a},#{a.class}"
  puts "b:#{b},#{b.class}"
end

a.call("a", "b")	#=> a:a,String
					#=> b:b,String

a.call("a", "b", "c")	#=> a:a,String
						#=> b:b,String

a.call("a")		#=> a:a,String
				#=> b:,NilClass

a.call(["a", "b"])	#=> a:a,String
					#=> b:b,String
```

proc处理参数的原则是：若传入参数多于参数表，则无视多出来的参数；若传入参数少于参数表，则剩余的形参为`nil`。而且传入一个数组时，把数组的元素当作参数。
而lambda则要严格按照形参表：

```ruby
b = lambda do |a,b|
  puts "a:#{a},#{a.class}"
  puts "b:#{b},#{b.class}"
end

b.call("a", "b")	#=> a:a,String
					#=> b:b,String

b.call("a", "b", "c")	#=> ArgumentError

b.call("a")		#=> ArgumentError

b.call(["a", "b"])	#=> ArgumentError
```

第二点是在proc和lambda中使用`return`的含义不同：

```ruby
def foo1
	a = Proc.new { return }
	a.call
	puts "foo1"
end

def foo2
	b = lambda { return }
	b.call
	puts "foo2"
end

foo1	#=> 无输出
foo2 #=> "foo2"

```

依我的理解，proc中的`return`是对于当前调用这个proc的作用域而言，而lambda的`return`是对于这个lambda中的块而言。

### 6.block的性能问题
对于各种使用block的情况做个测试：

```ruby
require 'benchmark/ips'

def block_pass(&block)
  1 + 1
end

def block_call(&block)
  block.call
end

def just_yield
  yield
end

def block_yield(&block)
  yield
end

def block_reference(&block)
  block
  1 + 1
end

Benchmark.ips do |x|
  x.report("call") do
    block_call { 1 + 1 }
  end

  x.report("yield") do
    block_yield { 1 + 1 }
  end

  x.report("reference") do
    block_reference { 1 + 1 }
  end

  x.report("pass") do
    block_pass { 1 + 1 }
  end

  x.report("pass no block") do
    block_pass
  end

  x.report("just yield") do
    just_yield { 1 + 1 }
  end

  x.compare!
end
```

结果如下：

```
  pass no block:  6726076.0 i/s
     just yield:  6560563.2 i/s - 1.03x slower
      reference:  1539552.6 i/s - 4.37x slower
          yield:  1494083.7 i/s - 4.50x slower
           pass:  1446721.5 i/s - 4.65x slower
           call:  1275053.0 i/s - 5.28x slower
```

最快的是不传递block，重点在于`just yield`与其它比起来有明显的性能差别。可以看出，显式传递block比隐式传递block的性能差了3倍左右。原因是，当显式传递block的时候ruby会创建一个新的proc，包含了昂贵的堆空间分配操作，而隐式传递block时，MRI(Matz's Ruby Interpreter)对其进行了优化，使用跟显式传递时完全不同的处理方法。

不过假如传递的是一个由proc转换的block，情况又不同了:

```ruby
# 把"just yield"的report改成这样
x.report("just yield") do
	a = Proc.new { 1 + 1 }
	just_yield &a
end
```

结果:

```
  pass no block:  6583575.2 i/s
      reference:  1630433.7 i/s - 4.04x slower
          yield:  1553697.5 i/s - 4.24x slower
           pass:  1545284.9 i/s - 4.26x slower
           call:  1392191.2 i/s - 4.73x slower
     just yield:  1361214.9 i/s - 4.84x slower
```

关于显式block的性能差距，更详细的描述参考[这里](https://docs.omniref.com/ruby/2.2.0/symbols/Proc/yield#annotation=4087638&line=711&hn=1)

#### 参考链接:
* http://awaxman11.github.io/blog/2013/08/05/what-is-the-difference-between-a-block/
* https://news.ycombinator.com/item?id=9118176
* https://rubymonk.com/learning/books/4-ruby-primer-ascent/chapters/18-blocks/lessons/51-new-lesson

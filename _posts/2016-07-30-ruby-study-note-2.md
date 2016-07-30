---
layout: post
title:  "Ruby基础学习小记(二)"
date:   2016-07-30 15:30:00 +0800
categories: ruby
---
###1. Ruby中的“===”符号
Ruby中每个对象都含有`===`方法，不过从意义上来说，`===`跟“相等”没有任何关系。先看看下面的例子:

```Ruby
1 === 1				#=> true
(1..9) === 3 		#=> true
(1..9) === 11		#=> false
/ab/ === "abcd"		#=> true
/ab/ === "def"		#=> true
Integer === 1		#=> true
1 === Integer		#=> false
Object === Integer	#=> true
```

stack overflow上的回答普遍认`a === b`的意义为:“假设用a描述一个集合，b能否放到这个集合中?”。上面的例子基本都符合这个解释，`Range`和`Regexp`自然不用说，而`Integer === 1`则表示为`1`是`Integer`这个集合中的一个，我认为可以理解为数学上所说的集合，`a === b`意为`b ∈ a`，那么`Integer === 1`理解为`1是否属于整数集`，`Object === Integer`理解为`Integer是否属于Object集`（Ruby中每个Class实际上是Object的一个instance）。那么`[1,2,3] === 1`这个表达式会返回什么呢？如果你理解了上面说的，那你就可以答出"false"。为什么？首先`[1,2,3]`是`Array`的一个实例，那我们所说的集合是这样的: `{ [1,2,3] }`，只有一个元素，这个集合肯定不包含`1`了。

`===`这个符号平时很少见，但是实际上你经常用到。`case...when语句`中条件匹配使用的就是`===`符号。所以我们才能这样写：

```
case year
when 1980..1989
 "80后"
when 1990..1999
 "90后"
end
```

参考链接:
http://stackoverflow.com/questions/3422223/vs-in-ruby
http://stackoverflow.com/questions/4467538/what-does-the-operator-do-in-ruby


###2. Ruby中的换行
Ruby解释器根据换行符插入的位置不同而有不同的处理，下面语句能正确地将`x`和`y`的和赋值给`total`

```Ruby
total = x +		# 解释器会认为语句未完结，继续往下解释
 y
```

但这条语句就不同了:

```Ruby
total = x		#解释器会认为这是完整语句，将x赋值给total
 + y			#另一个语句，对y求值
```

有三种情况下解释器会认为这是未完成的语句，分行应该以此为标准:

1. 操作符之后插入换行符
2. 方法调用的`.`**之后**插入换行符
3. 数组、哈希或方法调用中用于分隔元素的`,`**之后**插入换行符

从ruby 1.9开始，加入了新规则: 如果一行代码的首个非空白字符是`.`，那么这一行就是上一行的延续，忽略上一行的换行符。要注意这个规则跟上面的换行规则是有不同的。而在ruby 1.9之前没有这个特性。

### 3. 闭包

Ruby的`proc`和`lambda`都是闭包的，比起`block`，闭包还绑定了代码块中的全部变量。

```ruby
def plus(x)
 lambda { x + 1 }
end

my_plus = plus(5)
puts my_plus.call #=> 6
```

上面的例子可见，lambda返回后，调用是仍然能访问局部变量x。再看一个例子：

```ruby
class Printer
 def initialize(message)
   @message = message
 end

 def change_to(message)
   @message = message
 end

 def get_printer
   lambda { puts @message }
 end
end

printer = Printer.new('init message')
my_printer = printer.get_printer
my_printer.call							#=>  "init message"

printer.change_to('changed message')
my_printer.call							#=>	"changed message"
```

`@message`更变后lambda中的`@message`也跟着改变。

除此之外，闭包之间还能共享变量。

```ruby
class MessageManager
 def initialize(message)
   @message = message
 end

 def printer_helpers
   setter = lambda { |x| @message = x }
   getter = lambda { @message }
   return setter, getter
 end
end

setter,getter = MessageManager.new('init message').printer_helpers
puts getter.call				#=> init message

setter['changed message']
puts getter.call				#=> changed message
```

### 其他
1. Ruby默认采用ASCII编码，但支持使用其他编码方式。可以在文件的第一行加入**编码注释(coding comment)**:

 ```ruby
 # -*- coding: utf-8 -*-
 ```

 如果文件第一行是**shebang注释**，那**编码注释**放在第二行。

2. 可以通过`__ENCODING__`获取当前执行的代码的源编码

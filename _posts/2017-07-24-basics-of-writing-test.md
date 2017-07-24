---
layout: post
title:  "写测试的基础入门"
date:   2017-07-24 17:30:00 +0800
categories: test
---

## 前言

本文主要写给想接触测试但不知道如何下手的人。在第一次接触测试的时候，我是一头雾水的：测试到底怎样运行？要怎样写？接下来我会介绍BDD风格的测试写法，并介绍一些技巧与建议。示例分为ruby和javascript两个版本，分别使用[RSpec](http://rspec.info/)和[Sinon](http://sinonjs.org/)两个测试框架，在文中我将使用ruby版本，但你可以去下载对应的示例：[ruby示例]()，[javascript示例](https://github.com/ACzero/mocha_test_example)（PS：本文的例子省略了加载部分的代码，如需要能运行的代码请查看示例）。

## 测试的种类

相信你在网上能找到各种介绍测试种类的文章，测试种类很多，如：单元测试、集成测试、验收测试等等。单元测试的测试对象是单独的模块或类的功能；而集成测试的测试对象则是需要多个模块、类共同工作的功能；验收测试描述最终用户行为，如用户点击某个地方会触发什么行为，通过测试重现这个场景。

下面的示例都是单元测试，但是不用担心，只要掌握了基本的写法和技巧，写起来是没有太大区别的。

## 测试的基本步骤

一般来说测试的步骤分为：

1. 准备前置条件(Setup)
2. 执行(Exercise)
3. 验证(Verification)
4. 清理数据(Teardown)

当然也可能存在其他步骤不同的风格。参考下面一个rails中的测试示例，并看看这些步骤都做了什么：

```ruby
require 'rails_helper'

RSpec.describe Teacher, type: :model do
  describe "create a new teacher" do
    it "will strip and remove space in name" do
      # Setup
      school = School.create!(name: 'test')
      
      # Exercise
      teacher = Teacher.create!(name: ' a bc d', school: school)
      
      # Verification
      expect(teacher.name).to eq('abcd')
      
      # Teardown
      school.destroy!
      teacher.destroy!
    end
  end
end
```

**准备前置条件**：准备测试环境，一般是指插入测试用的数据，以及配置好一些变量等。

**执行**：执行需要测试的功能。

**验证**：验证功能执行之后各对象的状态（或行为）是否跟预期的一致。测试框架一般都会提供丰富的断言方法（后面会介绍）。

**清理数据**：删除本次测试中创建的数据，以及还原配置为默认值，使其不影响后面要运行的测试。

## 描述你的测试

假如你看过BDD风格的测试代码，那你肯定能看到`describe`，`context`，`it`等方法，这些方法其实并没有什么特殊的功能，其实就是用来描述测试的DSL。

首先是我们要测试的是`Calculator`这个module，包含一个`is_odd?`方法用来检验参数是不是奇数。对应的测试代码如下：

```ruby
RSpec.describe Calculator do
  describe '.is_odd?' do
    context 'when argument is odd' do
      it 'will not raise error' do
        expect { Calculator.is_odd?(1) }.not_to raise_error
      end

      it 'return true' do
        expect(Calculator.is_odd?(1)).to be true
      end
    end

    context 'when argument is even' do
      it 'will not raise error' do
        expect { Calculator.is_odd?(2) }.not_to raise_error
      end

      it 'return false' do
        expect(Calculator.is_odd?(2)).to be false
      end
    end
  end
end
```

首先指出，这里describe、context、it后面跟的字符串参数只起到描述作用，甚至可以不填。我们逐一查看，我们用`describe`描述要测试的是Calculator模块，然后又用`describe`表示要测试这个模块的`is_odd?`方法。接下来的`context`代表条件，此处是“当参数为奇数”，在这个`context`方法的块中包含了两个`it`方法，在里面就是我们要执行的测试。将这段测试代码用语言来描述，就是：

* 测试Calculator的`is_odd?`方法，当参数为奇数时，不应该抛出异常。
* 测试Calculator的`is_odd?`方法，当参数为奇数时，返回`true`。
* 测试Calculator的`is_odd?`方法，当参数为偶数时，不应该抛出异常。
* 测试Calculator的`is_odd?`方法，当参数为偶数时，返回`false`。

实际上，当你的测试失败时，rspec就会根据你的描述打印出对应的信息，来帮助你快速定位到哪里出错：

```bash
Failures:

  1) Calculator.is_odd? when argument is even return false
     Failure/Error: expect(Calculator.is_odd?(2)).to be false

       expected false
            got true
     # ./example1/calculator_spec.rb:26:in `block (4 levels) in <top (required)>'

Finished in 0.0201 seconds (files took 0.56765 seconds to load)
4 examples, 1 failure

Failed examples:

rspec ./example1/calculator_spec.rb:25 # Calculator.is_odd? when argument is even return false
```

## 断言(assertion)

你在上面的例子已经看到我们用`expect`来验证测试结果了，这是其中一种验证风格，此外还有其他的风格，如assert。不过他们做的事情都是一样的。

### 相等性检验

```ruby
describe 'expect equality' do
  it do
    foo = 1
    expect(foo).to eq(1)
  end

  it do
    foo = [1, 2, 3]
    expect(foo).not_to equal([1, 2, 3])
  end
end
```

这里有两段测试，需要注意的是这里的`eq`和`equal`方法是用什么方式检验相等性的（根据具体语言有所不同），从文档可以知道`eq`方法通过调用`#==`方法来验证，而`equal`则通过调用`#equal?`方法来验证。则上面两个验证等价于作了这样的验证：

```ruby
# expect(foo).to eq(1)
foo == 1

# expect(foo).not_to equal([1, 2, 3])
!foo.equal?([1, 2, 3])
```

在进行相等性检验时建议先认真阅读文档。

### 检验异常抛出

当需要检验是否抛出某异常（或没有抛出异常时），也有对应的方法：

```ruby
describe 'exception' do
  it do
    expect { nil.split(',') }.to raise_error(NoMethodError)
  end
end
```

### 检验状态变化

rspec提供了方便的写法来检验对象状态变化：

```ruby
describe 'state change' do
  it do
    arr = [1]
    expect { arr += [2, 3] }.to change { arr.size }.by(2)
  end
end
```

另外还有很多不同的helper，请查阅文档。

## 测试替身(Test Double)

`Double`一词来源于拍电影中常用的`stunt double(替身演员)`，顾名思义，替身的作用是用于替换掉功能中的某个部分，通常会应用在下列情况中：

* 要测试的功能需要访问一些外部服务（如web API）
* 要测试的功能由几个模块共同工作，但可能有一个甚至多个模块还没完成。
* 想要验证功能中的模块是不是按预期被调用

使用替身可以屏蔽掉这些外部依赖，让测试关注点回到要测试的功能本身。

替身的种类有很多，这里介绍常见的三种：`stub`、`spy`、`mock`。要注意关于这三者的定义有很多不同见解，这里的定义是我参考了`Sinon`和`RSpec`两个框架总结出来的。为避免先入为主，你可以先自己搜索一下。

### Stub

stub的作用是为特定的方法调用设置返回值。

比如说当你的测试会调用`File.read('fname')`，但你实际并不想他真的去读取一个文件，那就可以用stub屏蔽掉真正的读取操作并设置一个string对象作为返回值。

下面的例子再介绍一个用法：

```ruby
class Calendar
  def today_day_off?
    Date.today.saturday? || Date.today.sunday?
  end
end
```

类`Calendar`中有一个`#today_day_off?`方法判断今日是否休息日，但是因为Calendar类使用`Date.today()`方法去获取当前日期，这导致运行结果会跟测试运行时的日期有关，这显然不是我们想要的。因此我们在测试中使用了`stub`：

```ruby
RSpec.describe Calendar do
  describe '.today_day_off?' do
    context 'when today is sunday' do
      # before中的内容会在该块中每个测试运行前执行
      before do
        # stub Date.today
        allow(Date).to receive(:today).and_return(Date.parse('2017-07-23'))
      end

      it 'return true' do
        expect(Calendar.new.today_day_off?).to be true
      end
    end

    context 'when today is monday' do
      before do
        # stub Date.today
        allow(Date).to receive(:today).and_return(Date.parse('2017-07-24'))
      end

      it 'return false' do
        expect(Calendar.new.today_day_off?).to be false
      end
    end
  end
end
```

注意在`before`的块中我们stub了Date的`today`方法，使其返回一个我们指定的Date对象。这样我们就可以测试Calendar在7月23号和7月24号时运行的行为。

还有一点，使用stub的前提是你必须清楚你测试的对象的内部实现（这里是Calendar类），才能对内部的方法进行stub。

### Spy

spy的作用是记录对象的行为，可用于验证在对象上的方法调用。

看以下例子，这里有个`MyHelper`的module：

```ruby
module MyHelper
  def average_of(array)
    sum = array.reduce(&:+)
    sum.fdiv(array.size)
  end
end
```

需要测试内部是否是使用`reduce`方法来计算总和的，则可以使用spy

```ruby
include MyHelper

RSpec.describe MyHelper do
  describe '#average_of' do
    it 'use reduce to sum' do
      arr_spy = spy([1, 2, 3])
      average_of(arr_spy)

      expect(arr_spy).to have_received(:reduce)
    end
  end
end
```

在这个例子中我们先创建了一个数组对象的spy对象，这个spy对象的行为跟数组一致，但是会记录进行过的方法调用，把arr_spy作为average_of的参数调用后，通过检查arr_spy这个对象是否被调用过`reduce`方法就可以达到目的。此外，spy还可以验证方法调用接收了什么参数。

### mock

mock的功能是设置响应（stub）以及验证预期行为（spy）。一般使用的时候会生成一个mock对象，然后再设置该对象的方法响应。

mock主要用于依赖的模块没有完成时，能正常运行测试。参考下面例子：

```ruby
class Order
  def initialize(warehouse, amount)
    if warehouse.has_enough?(amount)
      warehouse.remove(amount)
      @valid = true
    else
      @valid = false
    end
  end

  def valid?
    !!@valid
  end
end
```

我们有一个`Order`类，需要使用`Warehouse`对象进行初始化。当`Warehouse`这个类还没实现的时候，我们就可以使用mock先制造一个"替身"：

```ruby
RSpec.describe Order do
  describe 'create new order' do
    context 'when inventory is enough' do
      it 'order is valid' do
        warehouse = double('warehouse')
        expect(warehouse).to receive(:has_enough?).with(50).and_return(true)
        expect(warehouse).to receive(:remove).with(50)

        order = Order.new(warehouse, 50)

        expect(order.valid?).to be true
      end
    end

    context 'when inventory is not enough' do
      it 'order is invalid' do
        warehouse = double('warehouse')
        expect(warehouse).to receive(:has_enough?).with(51).and_return(false)

        order = Order.new(warehouse, 51)

        expect(order.valid?).to be false
      end
    end
  end
end
```

在RSpec中使用`double`方法创建mock对象，要注意的是mock在使用上跟spy有一些不同，我们需要先为mock对象设置方法及其响应，mock对象会验证这些方法调用，假如到测试结束时，设置的方法都没有被调用，测试就会报错。而且这个验证是在执行前就定义了，因此我们不需要额外去验证。

### 关于测试替身

上面讲了这么多，或许你已经晕了。对`stub`、`spy`和`mock`的定义有各种不同的观点，以至于RSpec的文档都没有对其作定义，而是扔给你[一些文章](http://www.rubydoc.info/gems/rspec-mocks/frames#Further_Reading)让你自己去纠结。

我建议不需要太执着于这些替身的定义，更重要的是去实现**我们的需求**。先搞清楚我们需要如何使用替身，然后选择测试框架给我们提供的方法去实现就足够了。

## 总结

这些都是我在刚开始学习写测试的时候疑惑的地方，特别是对于替身的使用让我纠结了很久。而对于其他没有说的问题，有些是我还没遇到的，有些则是我认为不会太难理解的，例如准备测试数据(fixture或factory)等。

## 参考链接

* [Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
* [Spy vs Spy](https://robots.thoughtbot.com/spy-vs-spy)
* [RSpec doc](https://relishapp.com/rspec/)
* [Sinon doc](http://sinonjs.org/releases/v2.3.8/)
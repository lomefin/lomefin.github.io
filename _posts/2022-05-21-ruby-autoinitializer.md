---
title: "Ruby auto initializer"
layout: post
date: 2022-05-21 12:00:35 -0400
categories:
---

Following up on the [Use case post](2022-05-20-use-cases-for-ruby) I wanted to expand more on the autoinitializer method that is used in it.

Say, we have several clases that extend from a `Base` class, we have similar behaviours but different parameters. The previous example were classes which implement the Command Pattern. So every class has the same parameter-less method `execute` but what makes them unique is their name and parameters.

This example shows two commands who inherit from the `Command` base class.

{% highlight ruby linenos %}
class Command
  def execute
    throw NotImplementedException
  end
end

class EmailDocument < Command; end

class PrintDocument < Command; end

{% endhighlight %}

So both have a `@document` but they do things way differently inside. So command works nice. 

{% highlight ruby linenos %}
class Command
  def execute
    throw NotImplementedException
  end
end

class EmailDocument < Command

  def execute
    mailer.send @document, to: @email
  end
end

class PrintDocument < Command

  def execute
    @printer.print @document
  end
end
{% endhighlight %}

But this means that every command needs its own parameters in order to work (which is, by definition, correct)

{% highlight ruby linenos %}
class Command
  def execute
    throw NotImplementedException
  end
end

class EmailDocument < Command

  def initialize(document:, email:)
    @document = document
    @email = email
  end

  def execute
    mailer.send @document, to: @email
  end
end

class PrintDocument < Command

  def initialize(printer:, document:)
    @document = document
    @printer = printer
  end

  def execute
    @printer.print @document
  end
end
{% endhighlight %}

But you can see the pattern, every single parameter must be sent into an instance variable, not that hard, but we loose lines of code assigning them.

## The proposal

So we make a version of an autoinitializer

{% highlight ruby linenos %}
class Command 
  def initialize(*args)
    return if args.empty?

    @entered_arguments = args.reduce({}) { |p, c| p.merge(c) }
    method(__method__).parameters.each do |_t, name|
      instance_variable_set("@#{name}", @entered_arguments[name])
    end
  end
end

class PrintDocument
  def initialize(printer:, document:)
    super
    end
  end
end

class EmailDocument
  def initialize(email:, document:)
    super
  end
end
{% endhighlight %}

## The explanation

So what does this does? Lets go line by line. This are my own words and maybe I am interpreting something wrong here, but to high degree, it works.

`def initialize(*args)` defines the constructor as having multiple (0..N) parameters

`return if args.empty?` if there are no parameters, no more initializing is needed here (but maybe the @entered_arguments should be before)

`@entered_arguments = args.reduce({}) { |p, c| p.merge(c) }` every (named)argument is added to a Hash.

Ok, here it gets denser.

`method(__method__)` gets the Method object of the method that we currently are. So the initializer, but if we are in a child class, that child's initializer.

`method.parameters` gives a list of the parameters accepted by the initializer method. So no other parameters are passed. This filters out any not defined parameters.

`instance_variable_set("@#{name}", @entered_arguments[name])` this sets instance variables to the parameter value, so if a value was `price: 12` now `@price` will equal `12`. That means that maybe you could send a parameter names `entered_arguments` and overwrite what we have here? ABSOLUTELY. But for me it doesn't matter because, for now, its just a debugging thing.

## The Caveat

To make it happen we have to build our initializers like so

{% highlight ruby linenos %}
def initialize(a:, b:, c:)
  super
end
{% endhighlight %}

When `super` is called, invokes the same method in its parents **with the same parameters**. This is different than using `super()` which invokes the parent method with no arguments.

Rubocop doesn't like it, because it says that we are not adding any value here, but the value is in the definition of the constructor arguments, and hopefully, in the initializers documentation.

But if you want to keep rubocop under control, you either silence this cop, at least on these cases, or use another way to implement this.

There is also something called like smart initialize that solves the same kind of issue, but I find that the DSL is quite more complicated to document.


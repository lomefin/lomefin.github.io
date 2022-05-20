---
title: "Use cases for ruby"
layout: post
date: 2022-05-20 11:26:35 -0400
categories:
---

I've been working in Rails for **over a decade** and sometimes I feel disappointed.
Not because the framework is not up to my expectations but because I am not up to the creators' experience. Finally, I believe that we have reached some places where Rails is really on its limits and some work can be done by us to improve the way we bring solutions to our customers. We've been refining the concept of the use case for a long while now, so I wanted to share the current draft that we are working on. Shoutout to [@ggalaburri](https://github.com/ggalaburri) from [Migrante](https://migrante.com) which has helped in the development of this piece.

The Use Case is not something that I came upon by my own, some early ideas could be found in Familink's Contexts, and every once in a while a small improvement appeared.

## So, what is the problem that UC solves? 

Rails promotes DRY-ness and single-responsabilities, linked with their MVC but the thing is: models are in charge of persisting themselves correctly, controllers must move requests to where they can be processed, and views have no logic. So where should I put any business logic that includes a little more than creating or updating a model?

At that point you start either beefing up the model with some stuff that belongs to it, but also some stuff that modifies other models. Then realize that that shouldn't be there but in the controller, but the controllers ends up filling up with LOTS of decision making and some redundant code everywhere.

Testing some of these cases is **heavy** because the way to test them is to make requests over to them and doing lots of reading and writing to the database.

Use cases aims to solve that problem, the problem of not having an specific place for the most important thing of your work: business logic.

## The approach

The way we are trying to solve the issue is by using POROs, or the most simple approximation that we can (I'm pretty sure that it already depends on ActiveSupport, but nowadays who doesn't?).

This Ruby classes will have the business logic inside of them, hopefully trying not to use any complex library, for example, if we are using a model called `Car`, have `car` as a parameter instead of `car_id` so we don't have to use `Car.find(car_id)` so we don't rely on ActiveRecord. Every interaction should be as simple as possible and hopefully not depending on any technology implementation.

Lets use an internal presentations app as an example. To build a new presentation we can write something like

{% highlight ruby linenos %}
presentation = create_presentation @organization, starts: @starts, ends: @ends
create_rooms presentation:
@organization.members.each { |member| presentation.add_member member }
{% endhighlight %}

So here of course I am already abstracting the complexity of creating presentations, rooms and assigning members to `create_presentation`, `create_rooms` and `add_member` but that is alright, the core of this code is the business logic. So we should wrap it in a method.

{% highlight ruby linenos %}
class CreatePresentation
  def initialize(organization:)
    @organization = organization
  end

  def run
    presentation = create_presentation @organization, starts: @starts, ends: @ends
    create_rooms presentation:
    @organization.members.each { |member| presentation.add_member member }
  end

  private

  def create_presentation; end
  
  def create_rooms; end

  def add_member; end
end
{% endhighlight %}

I wrote down the other methods to be later implemented, but they are not necessary for this example.

Now we have an structured UseCase that we can call using `CreatePresentation.new(organization: org).run` but how do you know it worked? Exceptions? How can you verify its valid before trying to run it?
The first thing I decided was that the use cases should follow the [Command Pattern](https://refactoring.guru/design-patterns/command), the name of the class already follows that rule, so instead of using `run` I will use `execute()`.


{% highlight ruby linenos %}
class CreatePresentation
  def initialize(organization:)
    @organization = organization
  end

  def execute
    @valid = validate
    return self unless @valid

    @result = run
    self
  end

  def valid? = @valid

  def validate
    # Some validation that you might need, returns a boolean
  end

  def run
    # Business logic
  end 
end
{% endhighlight %}

The Usecase can be now checked if its valid to run, be executed, and get the execution result more than once.

We can call it

`CreatePresentation.new(organization: org).execute`

or 

{% highlight ruby linenos %}
uc = CreatePresentation.new(organization: org)

uc.valid?

uc.execute
uc.result
{% endhighlight %}

So we are reaching a point where we can expand this idea further.

## The current implementation

The way we are implementing use cases is creating first a module for it, in my opinion is better to say `UsesCases::CreatePresentation` than to say `CreatePresentationUseCase` but if you are 100% on board with the idea, you can probably just use `CreatePresentation` but I am always wary of naming conflicts.

There are about 15 implementations with UseCases, each one made on a different time period, so there is no good version tracking of its evolution (that's what I'm hoping to do soon). But here is one of the latest.

I'll be placing parts of the code to explain them separately, we can assume that we will be implementing the `CreatePresentation` usecase, but now it will be defined as

```ruby
module UseCases
  class CreatePresentation < Base
  end
end
```

{% highlight ruby linenos %}
def initialize(*args)
  @stage = :created
  return if args.empty?

  arguments = args.reduce({}) { |p, c| p.merge(c) }
  @entered_arguments = arguments
  method(__method__).parameters.each do |_t, name|
    instance_variable_set("@#{name}", arguments[name])
  end
end
{% endhighlight %}

The constructor is shared with every descendant of UseCase, it doesn't matter how many attributes it has, it stores them as instance variables. So for example

`CreatePresentation.new(organization: org)` will have `@organization` automagically assigned to it. The only caveat, that maybe brings all this concept down is that we need to call it like so

```ruby
module UseCases
  class CreatePresentation < Base
    def initialize(organization:)
      super
    end
  end
end
```

Probably I can write a separate article about this.

Later, to handle execution or validation errors we can add 

{% highlight ruby linenos %}
def error(attribute = nil, type = nil, value = nil)
  @errors << { stage: @stage,
                context: self.class.name.gsub('::', '.'),
                attribute:,
                type:,
                value: }
  false
end
{% endhighlight %}

This the way I believe error reporting working fairly OK, but its up to you. I tried to copy ActiveRecord Validations here, so, if the organization is missing I could do something like `error(:organization, :missing) if @organization.blank?`. Here is where [dry-validations](https://dry-rb.org/gems/dry-validation/1.8/rules/) could be a game changer.

To check if a use case has valid parameters maybe we should expand the validation wrapper

And finally, the execution method should look something like this

{% highlight ruby linenos %}
def execute
  ActiveRecord::Base.transaction do
    prepare!
    return self unless valid?

    before_run
    @result = run
    @stage = :executed
    after_run
  end
  self
rescue StandardError => e
  Rails.logger.error "Failed to execute #{self.class}"
  Rails.logger.error e.message
  e.backtrace.each { |line| Rails.logger.debug(line) }
  self
end
{% endhighlight %}

{% highlight ruby linenos %}
def valid?
  @errors = []
  prepare! unless @prepared
  validate
  @stage = :failed if @errors.present?
  @errors.empty?
rescue StandardError => e
  Rails.logger.error "UC>>StandardError detected #{e.message}"
  e.backtrace.each { |line| Rails.logger.debug(line) }
  @errors << e.message
  false
end
{% endhighlight %}

What the hell is `prepare!`? You already know it. Maybe between initialization and verification some processing must be done. Writing software is all about adding in-between layers. The important thing is that the way to check if something is ok is to check `@errors.empty?`

Adding a few more things, maybe stuff that should not be in there, the final result is the following:

{% highlight ruby linenos %}
module UseCases
  class Base
    class UseCaseError < StandardError; end

    attr_reader :errors, :result

    # build the instance variables for the given args
    def initialize(*args)
      @stage = :created
      return if args.empty?

      arguments = args.reduce({}) { |p, c| p.merge(c) }
      @entered_arguments = arguments
      method(__method__).parameters.each do |_t, name|
        instance_variable_set("@#{name}", arguments[name])
      end
    end

    # Adds an error to the list.
    # The error can be added in different stages, there is a change that a validation
    # error is not shown because it passed into the next stage and got cleared.
    # The error is based on ActiveModel.
    # @example error('name', 'presence') means that it name failed to be present.
    # @example error('user', 'authorization') means the use is not allowed to be here.
    # The idea is that this will help you form the I18n string for showing the errors.
    # @param attribute[String] what attribute failed
    # @param type[String] what failure made it fail
    def error(attribute = nil, type = nil, value = nil)
      @errors << { stage: @stage,
                   context: self.class.name.gsub('::', '.'),
                   attribute:,
                   type:,
                   value: }
      false
    end

    def dump_errors(error_list: [])
      error_list.each do |err|
        error(err.attribute, err.type)
      end
    end

    def execute
      ActiveRecord::Base.transaction do
        prepare!
        return self unless valid?

        before_run
        @result = run
        @stage = :executed
        after_run
      end
      self
    rescue StandardError => e
      Rails.logger.error "Failed to execute #{self.class}"
      Rails.logger.error e.message
      e.backtrace.each { |line| Rails.logger.debug(line) }
      self
    end

    alias perform execute

    def prepare!
      announce
      before_prepare
      prepare
      after_prepare
    end

    def announce
      return if ENV.fetch('ANNOUNCE_USE_CASES', 'NO') == 'ANNOUNCE'

      Rails.logger.info "Executing #{self.class.name}"
    end

    def before_prepare
      @stage = :validation
      @prepared = false
    end

    def after_prepare
      @prepared = true
    end

    def prepare; end

    def execute!
      prepare!
      raise ArgumentError unless valid?

      before_run
      @result = run
      @stage = :executed
      after_run
      self
    rescue StandardError => e
      Rails.logger.error "Failed to execute #{self.class}"
      Rails.logger.error e.message
      e.backtrace.each { |line| Rails.logger.debug(line) }
      raise UseCaseError, e.message
    end

    def after_run; end

    def before_run
      @stage = :execution
      @errors = []
    end

    def validate; end

    def valid?
      @errors = []
      prepare! unless @prepared
      validate
      @stage = :failed if @errors.present?
      @errors.empty?
    rescue StandardError => e
      Rails.logger.error "UC>>StandardError detected #{e.message}"
      e.backtrace.each { |line| Rails.logger.debug(line) }
      @errors << e.message
      false
    end

    def success?
      return false if @errors.nil?

      @errors.empty?
    end

    def error_list
      return [] unless @errors

      @errors.map do |x|
        "#{x[:context].underscore.tr('/', '.')}.#{x[:attribute].to_s.underscore}.#{x[:type].to_s.underscore}"
      end
    end

    def i18n_error_list
      @errors.map do |e|
        h = e.dup.with_indifferent_access
        h[:context] = h[:context].underscore.tr('/', '.')
        h
      end
    end

    def warning(attribute, type, message: nil)
      str = "Warning in #{self.class.name}, failed #{attribute} #{type}"
      str += "(#{message})" if message.present?
      Rails.logger.warn str
      # SlackService.notify(message: str, type: :warning)
    rescue StandardError
      Rails.logger.warn 'Failed to process warning.'
    end

    def t(key, **args) = I18n.t("usecases.#{self.class.to_s.underscore}#{key}", args)

    def perform_later!(_queue: :async_queue)
      raise NotImplementedError
    end

    def entered_arguments = @entered_arguments.dup

    def self.execute(args) = new(**args).execute

    def self.execute!(args) = new(**args).execute!

    def self.perform_later!(args) = new(**args).perform_later!

    protected

    def run
      raise NotImplementedError
    end
  end

  def execute(uc_name, **args)
    ucase = uc_name.to_s.camelize.constantize
    ucase.execute(args)
  end
end
{% endhighlight %}

This has some stuff for trying to work with ActiveJob and some other ways to invoke them, you can chose what you need.

## The long road ahead

This version of UseCase **works** and has simplified our work pipeline tremendously. But there is also room for tweaking so this is what I have planned.

First, rewrite it to use callbacks, so maybe we can do something like

{% highlight ruby linenos %}
class CreatePresentation

  validate :team_has_members?
  validate :organization_valid?

end
{% endhighlight %}

Second, make a gem. This is the weird part, there is already a gem called `use_cases` and its quite new. I contacted the author so maybe some part of this knowledge can help improve that gem or maybe its an alternative to it, the best thing is that if someone else thought about it then its not that bad of an idea.

Third, modularize it, even if it doesn't go as a gem, so if you don't use ActiveRecord, don't use transactions. 

Fourth, build a generator for Rails, this is the other scenario, if you are using Rails, then make something that really helps you to build more and more use cases. Developers are lazy and I've seen how making even an ActiveJob in console makes the developer use it more often is why a use case generator must be built.



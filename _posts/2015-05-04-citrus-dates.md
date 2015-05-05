---
layout: post
title: "Example for Citrus parsing"
date: 2015-05-04
comments: true
tags: rails citrus
categories: coding
---

[Michael Jackson's](https://github.com/mjackson) [Citrus](http://mjackson.github.io/citrus/) [gem](https://rubygems.org/gems/citrus/) is a general purpose
parser generator for Ruby, an alternative to the popular [Treetop](http://treetop.rubyforge.org/index.html) [gem](https://rubygems.org/gems/treetop).
I wanted to play with Citrus, so I wrote a toy date parser.  

<!--more-->

The Citrus [documentation](http://mjackson.github.io/citrus/example.html) goes over the rules for expressing a grammar well, but it doesn't seem to have a
quick example of a full application of Citrus in action.   The full code is at

After making a new Rails app, just add the following to the `Gemfile` and then `bundle install`:

```rb
gem 'citrus'
```

For this example, I'm turning off almost everything related to Rails since I don't want to fuss with a database right now, but I'll keep ActionController
just so Rails is still around.  Obviously, I could have just done this without including Rails at all, but this is just an example demo anyway.

```ruby
# in application.rb
 .
 .
 .
# require 'rails/all'
require "action_controller/railtie"
 .
 .
 .
module CitrusDates
  class Application < Rails::Application
 .
 .
 .
    # config.active_record.raise_in_transactional_callbacks = true

    # Disable ActiveRecord
    config.app_middleware.delete "ActiveRecord::ConnectionAdapters::ConnectionManagement"
  end
end
```

If you're going to take the same path of minimal Rails, be sure to go into the `config/environments/<env>.rb` files and comment out any Rails configuration
settings (including asset pipeline) that are not being used.

Now we're ready to start in on Citrus.  

First, let's put a grammar together.  We'll just toss a `dates.citrus` file into the models directory with the following contents:

```
# in models/dates.citrus
grammar Dates
  rule dateMDY
    whitespace? month delimiter day ( delimiter year)? whitespace?
  end

  rule month
    ('1' [0-2]) | ('0'? [1-9])
  end

  rule day
    ([12] [0-9]) | ('3' [0-1]) | ('0'? [1-9])
  end

  rule year
    ( ('1' '9') | ('2' [01]) )? [0-9] [0-9]  # 2 digits or 4 digits from 1900 through 2199
  end

  rule delimiter
    whitespace? ('/' | '-') whitespace?
  end

  rule whitespace
    [\s]+
  end
end
```

The syntax for Citrus is almost like Treetop. Instead of a slash to designate alternates, you use a bit more conventional vertical-bar symbol.  There are
other differences, but this is a simple parser, so we won't hit them all.

Next, we'll make a singleton model to act as a Parser object.

```ruby
# in models/parser.rb
class Parser

  def self.parse(data)
    Citrus.load(Citrus.require('dates'), force: true)
    Dates.parse(data)
  end

end
```

This is actually simpler than Treetop.  Citrus can navigate the path to our 'dates.citrus' via $LOAD_PATH using its `require` method.  As long as the
grammar file extension is `.citrus`, we're good to go.  

The `force` option is to assure the file gets reloaded each time.  In production, you probably
wouldn't want that, but in development you would because you want to capture changes to your grammar without having to reload the environment.  Instead of
`true`, we could use `Rails.env.development?` to change according to the context.  You'll get some warning messages logged when force is on, but at least you
can rapidly test grammar changes this way.

The load creates an object by the same name as our grammer, so `Dates` is now available to call `parse` against. We don't need a variable to
the parser, but `Citrus.load` does return an array of all the grammar objects created, so we could use the return value to get the parsing object we
wanted, but there's no need.   Just to illustrate, the two lines could be put together in one line as `Citrus.load(Citrus.require('dates'))[0].parse(data)`,
but that's nowhere near as readable.

If we start up a console session, we can immediate try working with it.  We'll give a deliberately invalid input to start.

```bash
$ rails c
2.2.0 :001 > Parser.parse('abc')
```

Notice that clear error handling is already provided in response:

```
Citrus::SyntaxError: Malformed Citrus syntax on line 1 at offset 0
abc
^
        from /home/richard/.rvm/gems/ruby-2.2.0/gems/citrus-3.0.2/lib/citrus/file.rb:361:in `rescue in parse'
        from /home/richard/.rvm/gems/ruby-2.2.0/gems/citrus-3.0.2/lib/citrus/file.rb:358:in `parse'
        from /home/richard/.rvm/gems/ruby-2.2.0/gems/citrus-3.0.2/lib/citrus.rb:47:in `eval'
        from /home/richard/src/citrus-dates/app/models/parser.rb:8:in `parse'
        from (irb):14
```

In Treetop, we'd have to test for a nil response and then build an error message that was anything near as complete.  OTOH, to customize this one,  
we'd probably have to trap the exception and reraise it with our own message or maybe even a custom error inherited from `Citrus::SyntaxError`.


Let's try something that should easily pass:

```
2.2.0 :002 > Parser.parse('3/14/2015')
 => "3/14/2015"
```

Cool, it parsed.  And just as a light test, we'll give it an invalid month:

```
2.2.0 :003 > Parser.parse('36/14/2015')
Citrus::SyntaxError: Malformed Citrus syntax on line 1 at offset 1
```

Obviously we can write proper unit tests and make sure the grammer is correct, but let's move on.

So our valid line just returned a string.  We had that already so hardly very interesting.  Now we'll hookup some logic.

Let's replace the first four rules in our grammar `dates.citrus` file with these contents:

```
  rule dateMDY
    ( whitespace? month delimiter day ( delimiter year)? whitespace? ) {
      Time.mktime(capture(:year).value, capture(:month).value, capture(:day).value)
    }
  end

  rule month
    ( ('1' [0-2]) | ('0'? [1-9]) )  { to_str.to_i }
  end

  rule day
    ( ([12] [0-9]) | ('3' [0-1]) | ('0'? [1-9]) )  { to_str.to_i }
  end

  rule year
    ( ( ('1' '9') | ('2' [01]) )? [0-9] [0-9] )  { to_str.to_i }  # 2 digits or 4 digits from 1900 through 2199
  end
```

There are two main things to notice.  First, each of the rules now has a logic block.  These blocks will be invoked by calling the `.value` method on the
return value from the `.parse` method.  Second, to set the blocks, we first had to enclose the rules in a set of parentheses; otherwise, the block would have
applied to the last element in the rule.

To read the blocks, start with the subordinate `month`, `day`, and `year` rules.  Those will render integer values correponding to the numeric value parsed.
Then in the topmost `dateMDY` rule, we `capture` those parts and take their values to construct a Ruby `Time` object.

Let's try it.

```
2.2.0 :004 > date = Parser.parse('3/14/2015')
 => "3/14/2015"
```

It seems to return a string, but let's try the new `date` variable that we used to assign the return value to.

```
2.2.0 :005 > date.value
 => 2015-03-14 00:00:00 -0600
```

And it worked!  So now, we're capturing actual Ruby Time objects directly from the parser.

A few final notes.  First, the logic for our `year` rule should be improved to add the current century if the integer parsed is less than 100.  That way we
can properly handle convering two-digit year entries into the Time objects appropriately.   Second, Citrus can handle a lot more involved function and method
specifications than what is demonstrated here.  Please check out their documentation.

And third?  Well, look!  We didn't need to make a dedicated `Parser` object!  All we needed were these two lines:

```ruby
    Citrus.load(Citrus.require('dates'), force: true)
    Dates.parse(data)
```

We could have just as easily put them into a controller method or anywhere we wanted parsing to occur.  Given that the `.load` method of Citrus loads all the
grammars, you can see loading them all in `ApplicationController` and then just invoke the `<Grammer>.parse` method from whereever desired when needed.

Pretty nifty.

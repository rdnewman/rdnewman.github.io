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
I wanted to play with Citrus, so I started with a toy date parser.  

<!--more-->

The Citrus [documentation](http://mjackson.github.io/citrus/example.html) goes over the rules for expressing a grammar well, but there's not a self contained
quick example of a full application with Citrus in action.   If you want to cut to the chase, the full code from this example is on [Github](https://github.com/rdnewman/citrus-dates).

To start, after making a new Rails app, just add the following to the `Gemfile` and then `bundle install`:

```rb
gem 'citrus'
```

For this example, I'm turning off almost everything related to Rails since I don't want to fuss with a database right now, but I'll keep *ActionController*
just so Rails is still around.  Obviously, I could have just done this example without including Rails at all and everything would still work.

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
other differences, but this is a simple parser, so that's the main one we'll see here.

Next, we'll make a singleton model to contain our Parser.

```ruby
# in models/parser.rb
class Parser

  def self.parse(data)
    Citrus.load(Citrus.require('dates'), force: true)
    Dates.parse(data)
  end

end
```

This is actually simpler than Treetop.  Citrus can navigate the path to our `dates.citrus` file from $LOAD_PATH using its `require` method.  As long as the
grammar file extension is `.citrus`, we're good to go.  

The `force` option in `.load` is to assure the grammer file gets reloaded each time.  In production, you probably
wouldn't want that, but in development you might want to capture changes to your grammar without having to reload the environment.  Instead of
`true`, we could use `Rails.env.development?` to change according to the context.  Note that you'll get some warning messages logged when force is on (which I won't show in the examples output),
but it is nice to rapidly test grammar changes this way.

The load creates an object by the same name as our grammer, so `Dates` is now available to call `parse` against. We don't need a variable to
the parser, but `Citrus.load` does return an array of all the grammar objects created.  We could use the return value to get the parsing object we
wanted, but there's no need here.   Just to illustrate, for this case, the two lines could be put together in one line as `Citrus.load(Citrus.require('dates'))[0].parse(data)`,
but that's nowhere near as readable and, as you'll see later, something that you're unlikely to ever want to do.

If we start up a console session, we can immediately work with it.  We'll give a deliberately invalid input to start.

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

In Treetop, we'd have to test for a nil response and then build an error message that was anything near as complete.  Like in Treetop, Citrus does include variables that give the line and offset so you could trap their exception
and reraise your own with a custom error message (see [here for an example of a similar error message in Treetop](http://whitequark.org/blog/2011/09/08/treetop-typical-errors/) -- look under "Fancy error reporting").


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


Let's go back to the valid one and look closer by assigning the result of the parsing to a variable.  You see the interpreted result
of the parsing by calling `.value` against the return value.

```
2.2.0 :004 > date = Parser.parse('3/14/2015')
=> "3/14/2015"
2.2.0 :005 > date.value
=> "3/14/2015"
```

So our valid line just returned a string.  We had that already to start with so this isn't very interesting other than to confirm it was a valid entry.  We need to hookup some logic.

Replace the first four rules in our grammar `dates.citrus` file with these contents.  The grammar won't change, but now we have some logic.

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

There are two main things to notice.  First, each of the rules now has a logic block.  These blocks are invoked in order by calling the `.value` method on the
return value when parsed.  Second, to set the blocks, we first had to enclose the rules in a set of parentheses; otherwise, the block would have
applied to the last element in the rule (similar to Treetop).

To read the blocks, start with the subordinate `month`, `day`, and `year` rules.  Those will render integer values correponding to the numeric value parsed.
In the topmost `dateMDY` rule, we then`capture` those parts and take their values to construct a Ruby `Time` object.

Let's try it.

```
2.2.0 :006 > date = Parser.parse('3/14/2015')
 => "3/14/2015"
2.2.0 :007 > date.value
 => 2015-03-14 00:00:00 -0600
```

And it worked!  So now, we're capturing Ruby Time objects directly from the parser.

A few final notes for this example.  First, the logic for our `year` rule should be improved to add the current century if the integer parsed is less than 100.  That way we
can properly handle converting two-digit year entries into the Time objects appropriately.   Second, Citrus can handle a lot more involved function and method
specifications than what is demonstrated here.  Please check out their documentation.

And third?  Well, look!  We didn't need to make a dedicated `Parser` object.  All we needed were these two lines:

```ruby
    Citrus.load(Citrus.require('dates'), force: true)
    Dates.parse(data)
```

We could have just as easily put them into a controller method or anywhere we wanted parsing to occur.  Given that the `.load` method of Citrus loads all the
grammars, you can see loading them all in `ApplicationController` and then just invoke the `<Grammer>.parse` method from whereever desired in any controller.

Pretty nifty.

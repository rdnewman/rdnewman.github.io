---
layout: post
title: "How to use ReactJS to handle AJAX flash messages in a Ruby on Rails 4.0 project"
date: 2014-11-16 20:00:00
comments: true
tags: reactjs rails ajax
categories: coding
---

Normally, Ruby flash messages are rendered server-side with perhaps a basic snippet of
code in a layout view. Something like this HAML could be used in basic Rails 4.0
layout (using Bootstrap):

<span class='codefilename'>app/views/layouts/application.html.haml:</span>

```haml
!!!
%html
  %head
    %title
      Demo

    = stylesheet_link_tag    'application', media: 'all', 'data-turbolinks-track' => true
    = javascript_include_tag 'application', 'data-turbolinks-track' => true
    = csrf_meta_tags

  %body
    %div.container
      %div#flash_messages
        - flash.each do |level, message|
          %div{class: "#{flash_class(level)}"}<
            = message

      = yield
```

<!--more-->
The reference to flash_class is just a helper for applying bootstrap styles.

<span class='codefilename'>app/helpers/application_helper.rb:</span>

```ruby
module ApplicationHelper
  # flash message typing for Bootstrap display
  def flash_class(level)
    case level
    when 'notice'
      'alert alert-info'
    when 'success'
      'alert alert-success'
    when 'error', 'alert'
      'alert alert-error'
    else
      'alert alert-error'
    end
  end
end
```

And this all works as fine as it goes. Flash messages will be displayed on every page render.

## React and AJAX

However, if you're writing a [React](http://facebook.github.io/react/index.html) project, you're likely using a fair amount of AJAX to take advantage of
what React can do for you. These notes don't discuss tying React into a Rails project -- there are several places to read about that -- instead, these
notes assume you're using [react-rails](https://github.com/reactjs/react-rails) to have your project automatically process React's JSX files in the
asset pipeline.

There are several answers on Stack Overflow about scripting for flash messaging in general Javascript; we can adapt this approach to take advantage of
React (for instance, see [here](http://stackoverflow.com/questions/366311/how-do-you-handle-rails-flash-with-ajax-requests)
and [here](http://stackoverflow.com/questions/21032465/rails-doesnt-display-flash-messages-after-ajax-call))

The value of React in this case is that it will automatically update the display when its data state is updated, so really all that needs to be
done is to pipe the data from the server to the client and let React do what it does. The main details are pretty straightforward:

* Transmit flash data from server when there's an update,
* Receive flash data on client and amend React component state data, and
* React component for displaying content.
* Hooking it all together.

### Transmit

The StackOverflow notes pretty well spell this out, but here's my implementation. One advantage of this approach is that it just uses the
basic Ruby store of the flash data to keep processing to a minimum.

Natively, flash messages are kept as an array of arrays. Each child array of the main array is just two elements:
the flash message type (`success`, `error`, etc.) and the supplied message text. Since arrays don't transmit well
without some notation to parse, it's just converted to JSON.

<span class='codefilename'>app/controllers/application_controller.rb:</span>

```ruby
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
  after_filter :flash_to_http_header

private
  def flash_to_http_header
    return unless request.xhr?
    return if flash.empty?
    response.headers['X-FlashMessages'] = flash.to_hash.to_json
    flash.discard  # don't want the flash to appear when you reload page
  end
end
```

### Receiving

This part assumes that you'll put all the flash related handling into a single `.js.jsx`. You don't have to of course,
but it would keep related functionality together. If you do it the same way I did, then the below function would go in that file.

<span class='codefilename'>app/assets/javascripts/react/flash_messages.js.jsx:</span>

```js
function handleFlashMessageHeader(node, xhr) {
  var _message_array = new Array();
  var _raw_messages = xhr.getResponseHeader("X-FlashMessages")
  if (_raw_messages) {
    var _json_messages = JSON.parse(_raw_messages);
    count = 0
    for (var key in _json_messages) {
      _message_array[count] = new Array();
      _message_array[count][0] = key;
      _message_array[count][1] = _json_messages[key];
      count += 1;
    }
  }
  node.messages(_message_array);
}
```

Note that the main processing here is just to convert back to an array of arrays. That way the React
component can simply direct invocations from a RoR view the same as in AJAX processing.

The node parameter is just the DOM node of the rendered React component which we'll outline next.

### Component

The component isn't too complicated, but a couple of things are worth a closer look. First, the props are
immediately converted to state for use in the render function. This normally is an anti-pattern in React,
but it allows the state to be updated dynamically in response to AJAX calls. The messages function is needed
for non-React JS calls to update the state with new information (which is why props don't work well).
Finally, the render function expects an array of 2-element arrays to produce each flash message -- again,
this allows the component to be used directly from a Rails view without preprocessing the data just
like in the non-React approach at the start of this post.

You'll also see the use of `flash_class` in the render and the corresponding local member function.
Again, this is just for Bootstrap styling. If your flash messages are going to be handled exclusively
client-side, then the server-side `application_helper.rb` need not handle it (and so can DRY out the code a bit).

<span class='codefilename'>app/assets/javascripts/react/flash_messages.js.jsx</span>

```js
/** @jsx React.DOM */

var FlashMessages = React.createClass({
  getInitialState: function() {
    return {messages: this.props.messages};
  },

  messages: function (messageArray) {
    this.replaceState({messages: messageArray});
  },

  render: function() {
    return (
      <div className='flash_messages_component'>
        {this.state.messages.map(function(message, index) {
          _level = message[0];
          _text  = message[1];
          return (
            <div key={index} className={this._flash_class(_level)}>
              {_text}
            </div>
          );
        }.bind(this))}
      </div>
    )
  },

  _flash_class: function(level) {
    var _result = 'alert alert-error';
    if (level === 'notice') {
      _result = 'alert alert-info';
    } else if (level === 'success') {
      _result = 'alert alert-success';
    } else if (level === 'error') {
      _result = 'alert alert-error';
    } else if (level === 'alert') {
      _result = 'alert alert-error';
    }
    return _result;
  }

});
```

### Rendering

Here's a quick invocation for when the page is loaded. Notice that flashDiv below
came from a direct `React.render` (previously `React.renderComponent`) call. Note
too that it uses a div id'ed by `#flash_messages`.

By simply calling the receiver function whenever an AJAX call completes, the
React component's state is updated triggering an immediate re-render of the incoming
flash messages. Obviously, other JQuery events could be used to trigger a flash
check, but for AJAX calls, this is the most obvious.

<span class='codefilename'>app/assets/javascripts/react/flash_messages.js.jsx:</span>

```js
$(document).ready(function() {
  var flashDiv = React.render(<FlashMessages/>, $('#flash_messages')[0]);

  $(document).ajaxComplete(function(event, xhr, settings) {
    handleFlashMessageHeader(flashDiv, xhr);
  });
});
```

Beware that in this implementation the `flashDiv` variable from the call to `React.render`
and the earlier code with the function `handleFlashMessageHeader` are in global
namespace on the JS side which may not be appropriate for some production applications.

Finally, we change the view of the application layout (see above) to just mark where the flash messages should be placed via
the ``#flash_messages` placeholder from

<span class='codefilename'>partial of app/views/layouts/application.html.haml:</span>

```haml
%div#flash_messages
  - flash.each do |level, message|
    %div{class: "#{flash_class(level)}"}&lt;
      = message
```
to just

```haml
%div#flash_messages
```

### Non-AJAX Rendering

If, however, AJAX isn't that important, the React component can be used directly from the view:

```haml
= react-component('FlashMessages', flash, 'div#flash_messages')
```
or even just

```haml
= react-component('FlashMessages', flash, :div)
```
Probably not a lot of value using React though if AJAX isn't involved, but it does allow
more DRY code if some pages' functionality depends on AJAX and others don't.

### Wrapup

Here is the complete JSX file that handles the client-side processing with ReactJS:

<span class='codefilename'>app/assets/javascripts/react/flash_messages.js.jsx:</span>

{% highlight js linenos %}
/** @jsx React.DOM */

var FlashMessages = React.createClass({
  getInitialState: function() {
    return {messages: this.props.messages};
  },

  messages: function (messageArray) {
    this.setState({messages: messageArray});
  },

  render: function() {
    return (
      <div className='flash_messages_component'>
        {this.state.messages.map(function(message, index) {
          _level = message[0];
          _text  = message[1];
          return (
            <div key={index} className={this._flash_class(_level)}>
              {_text}
            </div>
          );
        }.bind(this))}
      </div>
    )
  },

  _flash_class: function(level) {
    var _result = 'alert alert-error';
    if (level === 'notice') {
      _result = 'alert alert-info';
    } else if (level === 'success') {
      _result = 'alert alert-success';
    } else if (level === 'error') {
      _result = 'alert alert-error';
    } else if (level === 'alert') {
      _result = 'alert alert-error';
    }
    return _result;
  }

});

function handleFlashMessageHeader(node, xhr) {
  var _message_array = new Array();
  var _raw_messages = xhr.getResponseHeader("X-FlashMessages")
  if (_raw_messages) {
    var _json_messages = JSON.parse(_raw_messages);
    count = 0
    for (var key in _json_messages) {
      _message_array[count] = new Array();
      _message_array[count][0] = key;
      _message_array[count][1] = _json_messages[key];
      count += 1;
    }
  }
  node.messages(_message_array);
}

$(document).ready(function() {
  var dummy = new Array();
  var flashDiv = React.render(<FlashMessages messages={dummy} />, $('#flash_messages')[0]);

  $(document).ajaxComplete(function(event, xhr, settings) {
    handleFlashMessageHeader(flashDiv, xhr);
  });
});
{% endhighlight %}

*A version of this post also appears on
[Stack Overflow](http://stackoverflow.com/questions/26964974/handle-rails-flash-messages-after-ajax-calls-using-reactjs)&nbsp;&nbsp;
<a href='http://stackoverflow.com/questions/26964974/handle-rails-flash-messages-after-ajax-calls-using-reactjs', class='about-icon fa fa-stack-overflow'></a>.*

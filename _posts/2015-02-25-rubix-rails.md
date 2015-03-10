---
layout: post
title: "Integrating 'Rubix' Bootstrap theme into Rails 4.0"
date: 2015-02-25
comments: true
tags: reactjs bootstrap rubix
categories: coding
---
[Sketchpixy's](https://github.com/sketchpixy) ReactJS-based [Rubix](https://wrapbootstrap.com/theme/rubix-reactjs-powered-admin-template-WB09498FH)
can be a great fit for a React/RoR (*RRoR*, anyone?) project given its robust set of UI tools.  However, it's meant as a standalone React theme and
doesn't natively take advantage of Rail's asset pipeline or take advantage of the [react-rails](https://github.com/reactjs/react-rails) gem.  Sketchpixy has
stated that a gem for directly supporting Rubix is forthcoming, but no timeline has been published.

So in the meantime, this post is the first of a series that describes an integration implementation until the new gem is released.  We'll start by
looking at the Rubix theme and decide a strategy for implementation.

**UPDATE (3/10/2015): Sketchpixy has promised (see
[his comments](https://wrapbootstrap.com/theme/rubix-reactjs-powered-admin-template-WB09498FH/comments#C24276on))
to release a ready-for-Ruby version this theme within the next several days.  Once that is released,
the second part of this posting will review that integration rather than try to integrate the current version of Rubix.**

<img src='/images/WB09498FH.png'/>

<!--more-->

### Looking at Rubix
Rubix is available by purchase ($20 as of this writing) and is delivered as a `products-WB09498FH.zip` file.  The zip file contents
(as of Rubix version 2.1) are extensive with over 4,000 files and 355 directories.  Since this theme requires purchase and licensing from Sketchpixy,
I don't want to show any of its details, but the directory tree is outlined below.

<span onclick='toggle_visibility("dirtree")' class='togglevisibility'>
  Click here to toggle tree display (without individual files)
</span>
<pre id='dirtree' class='dirtree' style='display: none;'>
rubix-v2.2 (individual files not shown)
├── documentation
│   ├── Bootstrap
│   ├── Gulpfiles
│   └── Rubix
└── rubix
    ├── prebuild
    │   └── scaffold
    │       ├── jsx
    │       │   ├── common
    │       │   ├── react-styles
    │       │   │   └── src
    │       │   └── routes
    │       │       └── app
    │       └── sass
    │           ├── fonts
    │           ├── pages
    │           ├── print
    │           └── theme
    │               ├── components
    │               └── sections
    ├── public
    │   ├── css
    │   │   ├── app
    │   │   │   ├── blessed
    │   │   │   │   ├── ltr
    │   │   │   │   └── rtl
    │   │   │   ├── min
    │   │   │   │   ├── ltr
    │   │   │   │   └── rtl
    │   │   │   └── raw
    │   │   │       ├── ltr
    │   │   │       └── rtl
    │   │   ├── demo
    │   │   │   ├── blessed
    │   │   │   │   ├── ltr
    │   │   │   │   └── rtl
    │   │   │   ├── min
    │   │   │   │   ├── ltr
    │   │   │   │   └── rtl
    │   │   │   └── raw
    │   │   │       ├── ltr
    │   │   │       └── rtl
    │   │   ├── fonts
    │   │   │   ├── app
    │   │   │   └── demo
    │   │   └── vendor
    │   │       ├── morris
    │   │       ├── pace
    │   │       └── perfect-scrollbar
    │   ├── favicons
    │   ├── fonts
    │   │   ├── dropbox
    │   │   │   ├── app
    │   │   │   └── demo
    │   │   ├── glyphicon
    │   │   └── Lato-others
    │   ├── imgs
    │   │   ├── avatars
    │   │   ├── blueimp-gallery
    │   │   ├── datatables
    │   │   ├── dropzone
    │   │   ├── flags
    │   │   │   ├── flags
    │   │   │   │   └── flat
    │   │   │   │       ├── 16
    │   │   │   │       ├── 24
    │   │   │   │       ├── 32
    │   │   │   │       ├── 48
    │   │   │   │       ├── 64
    │   │   │   │       ├── icns
    │   │   │   │       └── ico
    │   │   │   └── flags-iso
    │   │   │       └── flat
    │   │   │           ├── 16
    │   │   │           ├── 24
    │   │   │           ├── 32
    │   │   │           ├── 48
    │   │   │           └── 64
    │   │   ├── gallery
    │   │   ├── homepage
    │   │   ├── jcrop
    │   │   ├── leaflet
    │   │   ├── select2
    │   │   ├── shots
    │   │   ├── timeline
    │   │   │   └── user-interface
    │   │   ├── trumbowyg
    │   │   ├── unsplash
    │   │   ├── wefunction
    │   │   └── xeditable
    │   ├── js
    │   │   ├── app
    │   │   ├── common
    │   │   │   ├── react
    │   │   │   ├── react-bootstrap
    │   │   │   ├── react-l20n
    │   │   │   ├── react-responsive
    │   │   │   ├── react-router
    │   │   │   ├── rrouter
    │   │   │   └── rubix
    │   │   ├── demo
    │   │   ├── minified
    │   │   ├── polyfills
    │   │   └── vendor
    │   │       ├── blueimp-gallery
    │   │       ├── bootstrap
    │   │       ├── bootstrap-datetimepicker
    │   │       ├── bootstrap-slider
    │   │       ├── c3js
    │   │       ├── chartjs
    │   │       ├── codemirror
    │   │       │   ├── addon
    │   │       │   │   ├── comment
    │   │       │   │   ├── dialog
    │   │       │   │   ├── display
    │   │       │   │   ├── edit
    │   │       │   │   ├── fold
    │   │       │   │   ├── hint
    │   │       │   │   ├── lint
    │   │       │   │   ├── merge
    │   │       │   │   ├── mode
    │   │       │   │   ├── runmode
    │   │       │   │   ├── scroll
    │   │       │   │   ├── search
    │   │       │   │   ├── selection
    │   │       │   │   ├── tern
    │   │       │   │   └── wrap
    │   │       │   ├── bin
    │   │       │   ├── demo
    │   │       │   ├── doc
    │   │       │   ├── keymap
    │   │       │   ├── lib
    │   │       │   ├── mode
    │   │       │   │   ├── apl
    │   │       │   │   ├── asterisk
    │   │       │   │   ├── clike
    │   │       │   │   ├── clojure
    │   │       │   │   ├── cobol
    │   │       │   │   ├── coffeescript
    │   │       │   │   ├── commonlisp
    │   │       │   │   ├── css
    │   │       │   │   ├── cypher
    │   │       │   │   ├── d
    │   │       │   │   ├── diff
    │   │       │   │   ├── django
    │   │       │   │   ├── dtd
    │   │       │   │   ├── dylan
    │   │       │   │   ├── ecl
    │   │       │   │   ├── eiffel
    │   │       │   │   ├── erlang
    │   │       │   │   ├── fortran
    │   │       │   │   ├── gas
    │   │       │   │   ├── gfm
    │   │       │   │   ├── gherkin
    │   │       │   │   ├── go
    │   │       │   │   ├── groovy
    │   │       │   │   ├── haml
    │   │       │   │   ├── haskell
    │   │       │   │   ├── haxe
    │   │       │   │   ├── htmlembedded
    │   │       │   │   ├── htmlmixed
    │   │       │   │   ├── http
    │   │       │   │   ├── jade
    │   │       │   │   ├── javascript
    │   │       │   │   ├── jinja2
    │   │       │   │   ├── julia
    │   │       │   │   ├── kotlin
    │   │       │   │   ├── livescript
    │   │       │   │   ├── lua
    │   │       │   │   ├── markdown
    │   │       │   │   ├── mirc
    │   │       │   │   ├── mllike
    │   │       │   │   ├── nginx
    │   │       │   │   ├── ntriples
    │   │       │   │   ├── octave
    │   │       │   │   ├── pascal
    │   │       │   │   ├── pegjs
    │   │       │   │   ├── perl
    │   │       │   │   ├── php
    │   │       │   │   ├── pig
    │   │       │   │   ├── properties
    │   │       │   │   ├── puppet
    │   │       │   │   ├── python
    │   │       │   │   ├── q
    │   │       │   │   ├── r
    │   │       │   │   ├── rpm
    │   │       │   │   │   └── changes
    │   │       │   │   ├── rst
    │   │       │   │   ├── ruby
    │   │       │   │   ├── rust
    │   │       │   │   ├── sass
    │   │       │   │   ├── scheme
    │   │       │   │   ├── shell
    │   │       │   │   ├── sieve
    │   │       │   │   ├── slim
    │   │       │   │   ├── smalltalk
    │   │       │   │   ├── smarty
    │   │       │   │   ├── smartymixed
    │   │       │   │   ├── solr
    │   │       │   │   ├── sparql
    │   │       │   │   ├── sql
    │   │       │   │   ├── stex
    │   │       │   │   ├── tcl
    │   │       │   │   ├── tiddlywiki
    │   │       │   │   ├── tiki
    │   │       │   │   ├── toml
    │   │       │   │   ├── turtle
    │   │       │   │   ├── vb
    │   │       │   │   ├── vbscript
    │   │       │   │   ├── velocity
    │   │       │   │   ├── verilog
    │   │       │   │   ├── xml
    │   │       │   │   ├── xquery
    │   │       │   │   ├── yaml
    │   │       │   │   └── z80
    │   │       │   ├── test
    │   │       │   │   └── lint
    │   │       │   └── theme
    │   │       ├── d3
    │   │       ├── datatables
    │   │       ├── dropzone
    │   │       ├── eventemitter2
    │   │       ├── fullcalendar
    │   │       │   ├── demos
    │   │       │   │   ├── json
    │   │       │   │   └── php
    │   │       │   ├── lang
    │   │       │   └── lib
    │   │       │       └── cupertino
    │   │       │           └── images
    │   │       ├── gmaps
    │   │       ├── holder
    │   │       ├── ion.rangeSlider
    │   │       ├── ion.tabs
    │   │       ├── jcrop
    │   │       ├── jquery
    │   │       ├── jquery-bootgrid
    │   │       ├── jquery.knob
    │   │       ├── jquery-steps
    │   │       ├── jquery-ui
    │   │       │   └── external
    │   │       │       └── jquery
    │   │       ├── jquery-validate
    │   │       ├── l20n
    │   │       ├── leaflet
    │   │       ├── messenger
    │   │       ├── moment
    │   │       ├── morris
    │   │       ├── nestable
    │   │       ├── pace
    │   │       ├── prism
    │   │       ├── p-scrollbar
    │   │       │   ├── examples
    │   │       │   ├── min
    │   │       │   └── src
    │   │       ├── raphael
    │   │       ├── select2
    │   │       ├── sparklines
    │   │       ├── switchery
    │   │       ├── tablesaw
    │   │       ├── timeline
    │   │       ├── trumbowyg
    │   │       │   ├── langs
    │   │       │   └── plugins
    │   │       │       ├── base64
    │   │       │       └── upload
    │   │       ├── typeahead
    │   │       ├── vex
    │   │       └── xeditable
    │   ├── locales
    │   │   ├── app
    │   │   │   └── en-US
    │   │   └── demo
    │   │       ├── ar
    │   │       ├── ch
    │   │       ├── en-US
    │   │       ├── fr
    │   │       ├── ge
    │   │       └── it
    │   └── video
    │       └── homepage
    └── src
        ├── global
        │   ├── requires
        │   ├── sass
        │   │   ├── rubix
        │   │   │   ├── base
        │   │   │   ├── layout
        │   │   │   ├── module
        │   │   │   └── overrides
        │   │   └── vendor
        │   │       ├── blueimp-gallery
        │   │       ├── bootstrap
        │   │       │   └── bootstrap
        │   │       │       └── mixins
        │   │       ├── bootstrap-datetimepicker
        │   │       ├── bootstrap-old
        │   │       │   └── bootstrap
        │   │       │       └── mixins
        │   │       ├── bootstrap-slider
        │   │       ├── c3js
        │   │       ├── csstyle
        │   │       ├── datatables
        │   │       ├── dropzone
        │   │       ├── fullcalendar
        │   │       ├── hubspot
        │   │       ├── ion
        │   │       ├── jcrop
        │   │       ├── jquery-steps
        │   │       ├── leaflet
        │   │       ├── nestable
        │   │       ├── prism
        │   │       ├── sass-list-maps
        │   │       ├── select2
        │   │       ├── switchery
        │   │       ├── tablesaw
        │   │       ├── timeline
        │   │       ├── trumbowyg
        │   │       ├── typeahead
        │   │       └── xeditable
        │   └── vendor
        │       ├── bootstrap
        │       └── l20n
        ├── jsx
        │   ├── app
        │   │   ├── common
        │   │   ├── react-styles
        │   │   │   └── src
        │   │   └── routes
        │   │       └── app
        │   └── demo
        │       ├── common
        │       ├── react-styles
        │       │   └── src
        │       └── routes
        │           └── app
        │               ├── blog
        │               ├── charts
        │               │   └── rubix
        │               ├── colors
        │               ├── docs
        │               │   ├── bootstrap
        │               │   ├── common
        │               │   └── snippets
        │               └── fonts
        └── sass
            ├── app
            │   ├── fonts
            │   ├── pages
            │   ├── print
            │   └── theme
            │       ├── components
            │       └── sections
            └── demo
                ├── fonts
                ├── pages
                ├── print
                └── theme
                    ├── components
                    └── sections

360 directories
</pre>

Some of this content is to support building the theme for use outside of rails, but we don't needs those parts.  The author provides updates for his theme
and so it's easiest if we just park the whole thing under `/vendor/assets` even if it increases the storage footprint.  Our strategy will be to target
the asset portions needed without having to disrupt the directory structure in place.

### Basic Rails app with React
The next post on this subject will get specific with regard to Rubix, but for now, we can at least prepare a basic Rails and React app as a baseline.

#### <u>Make basic Rails 4.2 app with Bootstrap and verify it displays text from a view</u>

First, just make a new app:

```bash
$ rails new rubix-rails
$ cd rubix-rails
```

Add the following to the `Gemfile` and then `bundle install`:

```rb
gem 'bootstrap-sass'
gem 'slim-rails'
```

Only `bootstrap-rails` is necessary to make this work.  I like [slim](http://slim-lang.com/), so I'm tossing in
the [gem](https://github.com/slim-template/slim-rails) for that.  If you don't want Slim,
you can skip this gem and the next step (or substitute with Haml instead).  Most of the UI will be React based, so it won't much matter whether you
use erb, haml, or slim.

To integrate Bootstrap, first move the `application.css` to be a `.scss` file:

```bash
$ mv app/assets/stylesheets/application.css app/assets/stylesheets/application.scss
```

And then append the following lines into that file:

```sass
@import "bootstrap-sprockets";
@import "bootstrap";
@import 'bootstrap/theme';
```

If you do use slim, here's the initial `app/views/layouts/application.html.slim` that restates the default `app/views/layouts/application.html.erb`
(be sure to delete the `.erb` version):

```slim
doctype html
html
  head
    title Rubix on Rails
    = stylesheet_link_tag    'application', media: 'all', 'data-turbolinks-track' => true
    = javascript_include_tag 'application', 'data-turbolinks-track' => true
    = csrf_meta_tags

  body
    = yield
```

To get our test view going, we'll just do a basic Welcome index page.  

```bash
$ mkdir -p app/views/welcome
```

Add this as the content of `app/views/welcome/index.html.slim`

```slim
h3 Welcome!

p This line was produced from the Rails view.
```

Add an empty `WelcomeController` under `app/controllers` to simply read as follows:

```ruby
class WelcomeController < ApplicationController
end
```

Add the view to `config/routes.rb` by just uncommenting out the line `root 'welcome#index'` that rails already put in there.

Verify that everything is working so far: running rails (`rails s`) should give you a nice bold "Welcome to my new page" using a Bootstrap font.  
If so, we're ready to install `react-rails`; otherwise fix the basic rails app (useful references abound).  

#### <u>Install react-rails and verify it adds new text to the page</u>

These next steps are taken from the [`react-rails` gem site](https://github.com/reactjs/react-rails), so refer there if questions.

Update the `Gemfile` and...

```rb
gem 'react-rails', '~> 1.0.0.pre', github: 'reactjs/react-rails'
```

...install it:

```bash
$ rails g react:install
```

Under the new `app/assets/javascripts/components` subdirectory just created, create a new `demo_line.js.jsx` file with these contents:

```js
/** @jsx React.DOM */

var DemoLine = React.createClass({
  render: function() {
    return (
      <span className='label label-success'>This line was produced in ReactJS.</span>
    );
  }
});
```

Go back under the `app/views/welcome/index.html.slim` view and add this line below :

```slim
= react_component 'DemoLine', {}, :div
```

And, finally, test again.  The "This line was produced in ReactJS." text should appear in the green background of a Bootstrap success label.

If so, we've demonstrated React and Bootstrap are installed and integrated.  We're ready to go on to getting Rubix integrated!

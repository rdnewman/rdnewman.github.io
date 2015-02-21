---
layout: post
title: "Integrating 'Rubix' Bootstrap theme into Rails 4.0"
date: 2014-11-16 20:00:00
comments: true
tags: reactjs bootstrap rubix
categories: coding
published: false
---

[Sketchpixy's](https://github.com/sketchpixy) ReactJS-based [Rubix](https://wrapbootstrap.com/theme/rubix-reactjs-powered-admin-template-WB09498FH)
can be a great fit for a React/RoR (*RRoR*, anyone?) project given its robust set of UI tools.  

However, it's meant as a standalone React theme and doesn't natively take advantage of Rail's asset pipeline or integrate with 
the [react-rails](https://github.com/reactjs/react-rails) gem.   This post describes an integration implementation.

<!--more-->

=== Ingredients
Rubix is available by purchase ($20 as of this writing) and is delivered as a `products-WB09498FH.zip`.  We're going to start with a completely new Rails 4.0 app and load the `react-rails` gem and show a basic `hello world` with vanilla React.  Because I like Sass and Slim, those will be installed as well.  Then we'll add the Rubix theme into the app's vendor assets and demonstrate that we can use the React components it provides.

The zip file contents (as of Rubix version 2.1) are extensive with over 4000 files and 355 directories.  Here's the tree (click to expand):

products-WB09498FH
```
products-WB09498FH
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

355 directories
```




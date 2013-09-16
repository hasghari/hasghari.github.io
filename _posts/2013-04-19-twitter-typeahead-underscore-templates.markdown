---
layout: post
title:  "Using underscore.js templates with Twitter Typeahead"
date:   2013-04-19
tags: java avro idl
comments: true
---

Recently I decided to use the [Twitter Typeahead][tt] autocomplete javascript library. One of the nice features of the library
is that it allows for specifying custom templates and template engines. The template engine, however, must comply to an
[interface][template-interface].

As I was already using underscore templates and underscore by default is not compatible, it was very easy to write a
mixin to become compatible:

{% highlight javascript %}
_.mixin({
  compile: function (str) {
    var compiled = _.template(str);
    compiled.render = function (context) {
      return this(context);
    };
    return compiled;
  }
});
{% endhighlight %}

[tt]: https://github.com/twitter/typeahead.js
[template-interface]: https://github.com/twitter/typeahead.js#template-engine-compatibility

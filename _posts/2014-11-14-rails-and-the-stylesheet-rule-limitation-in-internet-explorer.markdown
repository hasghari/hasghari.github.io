---
layout: post
title:  "Rails and the Stylesheet Rule Limitation in Internet Explorer"
date:   2014-11-14
tags: rails css internet-explorer
comments: true
---

It is a well-known problem that in order to be compatible with Internet Explorer 6 through 9, a single stylesheet may
contain up to 4095 rules. If you exceed that limit, the styles on the page will break and it could be hard to track down
the issue if you're not familiar with this limitation.

Not too long ago at my workplace, we encountered this very same problem. Our `application.css` file had far too many
rules for IE's liking. Fortunately for us, our experienced designer had encountered this problem before so we didn't
spend a lot of time debugging the source of the problem.

I don't want to get into all the different solutions for splitting up your stylesheets but instead outline what worked
best for us and how we can prevent this from happening moving forward.

Controller-specific Stylesheets
-------------------------------

By convention, the Rails scaffold generator will create a separate stylesheet per controller. Instead of including each
controller-specific stylesheet in the `application.css` file that is globally included on all of our pages, we decided
to dynamically include the stylesheets based on which controller handles the request:

1. Move all controller stylesheets to the `app/assets/stylesheets/pages` directory.

2. Define a helper method in `ApplicationHelper`:

    ```ruby
    def controller_stylesheet
      if Rails.application.assets.find_asset("pages/#{params[:controller]}")
        stylesheet_link_tag "pages/#{params[:controller]}", media: :all
      end
    end
    ```

3. Include the controller stylesheet in your `application.html` layout:

    ```erb
    <%= stylesheet_link_tag 'application', media: 'all' %>
    <%= controller_stylesheet %>
    ```

By taking this approach, we were able to get under the imposed limit on all of our stylesheets but how do we detect
this problem before deployment in the future?

Validate Stylesheets
--------------------

With the help of the [css_splitter][css-splitter] gem, we created a rake task to run as a post build step:

```ruby
namespace :stylesheets do
  task validate: :environment do
    require 'css_splitter'

    manifest = ActionView::Base.assets_manifest
    limit = 4095
    message = "%{key} has %{count} selectors which exceeds the limit of #{limit}"

    manifest.assets.select do |key, _|
      File.extname(key) == '.css'
    end.each do |key, value|
      count = CssSplitter::Splitter.count_selectors File.join(manifest.dir, value)
      puts "#{key}: #{count}"
      fail message % { key: key, count: count} if count > limit
    end
  end
end

```

The task above requires that the assets be precompiled so to put it all together, we run the following commands as a
post build step and should we exceed the limit, the task will fail with a message pointing to the offending file.

```
bundle exec rake assets:precompile
bundle exec rake stylesheets:validate
```

[css-splitter]: https://github.com/zweilove/css_splitter

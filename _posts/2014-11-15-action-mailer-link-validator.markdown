---
layout: post
title:  "ActionMailer Link Validator"
date:   2014-11-15
tags: rails mailer
comments: true
---

Recently we encountered a bug where a link in a delivered email was not working. A quick investigation revealed that
the problem was due to a relative url being generated instead of absolute. The problem was easy to fix but we wanted to
put a safeguard in place to prevent this from happening in the future.

As far as I know, Rails provides 3 different helper methods to generate urls:

1. `url_for` helper
2. `link_to` helper which delegates to `url_for`
3. Named route helpers such as `product_path(@product)` which delegate to `url_for`

Therefore it seems like as long as we override the implementation of `url_for` in our mailers and throw an
exception when the provided url is relative, we can prevent the mailer from being delivered:

```ruby
# config/initializers/mailer_link_validator.rb

ActionMailer::Base.helper do
  def url_for(options = nil)
    url = super(options)
    return url if absoulte_url?(url) || fragment?(url)
    fail 'Can not provide relative urls in mailers'
  end

  private

  def absoulte_url?(url)
    URI.parse(url).is_a? URI::HTTP
  rescue
    false
  end

  def fragment?(url)
    url.start_with? '#'
  end
end

```

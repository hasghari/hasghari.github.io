---
layout: post
title:  "ActiveRecord: Enhanced Query Objects"
date:   2013-09-15
tags: activerecord rails
comments: true
---

Your ActiveRecord models are usually the first place in your application where the unwieldy code begs for refactoring.

In an excellent post by Bryan Helmkamp on the Code Climate Blog, he outlined
[7 Patterns to Refactor Fat ActiveRecord Models][code-climate-blog]. One of the patterns from this blog post that I want
to focus on is **Extract Query Objects**.

We have been using this pattern for a while but I missed the convenience of chainable and reusable scopes. Here's an
example:

{% highlight ruby linenos %}
class Product < ActiveRecord::Base
  has_many :reviews
end

class PopularProductQuery
  def initialize(relation = Product.scoped)
    @relation = relation
  end

  def popular(time)
    @relation.joins(:reviews).where(reviews: { created_at: time..Time.now,
                                               available: true })
  end

  def with_recent_activity(time)
    @relation.joins(:reviews).where(reviews: { created_at: time..Time.now })
  end

  def with_available_reviews
    @relation.joins(:reviews).where(reviews: { available: true })
  end
end
{% endhighlight %}

The query object above defines three utility methods to return records of `Product` with certain properties. However,
you will notice that `PopularProductQuery#popular` is combining the logic of `#with_recent_activity` and
`#with_available_reviews`. The trivial solution to keeping this DRY is defining scopes on the `Product` model:

{% highlight ruby linenos %}
class Product < ActiveRecord::Base
  has_many :reviews

  scope :popular, ->(time) {
    with_recent_activity(time).with_available_reviews
  }

  scope :with_recent_activity, ->(time) {
    joins(:reviews).where(reviews: { created_at: time..Time.now })
  }

  scope :with_available_reviews, ->(time) {
    joins(:reviews).where(reviews: { available: true })
  }
end
{% endhighlight %}

Ideally we would like to define these scopes on our query objects to prevent our models from growing "fat" over time. If
these scopes were so common that they would be used across many different contexts in our application, we would probably
want to keep them on the model but for the purpose of this post, let's assume that these are very specific scopes that
we would like to isolate to the query object.

An existing but rarely advertised feature of ActiveRecord is that you have the ability to extend any `ActiveRecord::Relation`
object with your custom scopes:

{% highlight ruby linenos %}
class PopularProductQuery
  def initialize(relation = Product.scoped)
    @relation = relation.extending(Scopes)
  end

  def popular(time)
    @relation.with_recent_activity(time).with_available_reviews
  end

  module Scopes
    def with_recent_activity(time)
      joins(:reviews).where(reviews: { created_at: time..Time.now })
    end

    def with_available_reviews
      joins(:reviews).where(reviews: { available: true })
    end
  end
end
{% endhighlight %}

Here we are taking advantage of the `ActiveRecord::QueryMethods#extending` method to add custom scopes to our query object
without polluting the model space. In other words, `Product.with_available_reviews` is **not** valid. To put it all
together, you would use the enhanced query object like so:

{% highlight ruby %}
PopularProductQuery.new.popular(2.weeks.ago)
{% endhighlight %}

I've come to really like this pattern to adhere to the Single Responsibility Principle and keep my models manageable.

[code-climate-blog]: http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/
---
layout: post
title:  "Skinny Rails Controllers"
date:   2014-01-26
tags: rails
comments: true
---

Many times when we try to keep our controllers skinny, we end up moving our business logic to the model. In general, this
is good practice but there are cases where models are not a good fit to our needs either.

One such case is when we need to trigger mailers upon certain user actions. For example:

{% highlight ruby linenos %}
class PostsController < ApplicationController
  def create
    @post = Post.new permitted_params

    respond_to do |format|
      if @post.save
        PostMailer.notify(@post).deliver
        format.html { redirect_to @post, notice: 'Successfully created Post' }
      else
        format.html { render action: 'new' }
      end
    end
  end
end
{% endhighlight %}

One approach I've personally been guilty of in the past is tying mailer invocations to the model lifecycle:

{% highlight ruby linenos %}
class Post < ActiveRecord::Base
  after_create do
    PostMailer.notify(self).deliver
  end
end
{% endhighlight %}

Unfortunately this approach has several pitfalls, the main one in my use case being that multiple sources in our application
can trigger the model lifecylce callbacks and it becomes a maintenance headache trying to prevent unwanted execution
of these actions that really should only be triggered when instigated by a user request.

Fortunately for us though, Rails is natively equipped with a great notification system, namely `ActiveSupport::Notifications`.
By taking advantage of this notification system and controller `after_action` callbacks, we will create a mailer module
using the observer pattern.

{% highlight ruby linenos %}
module MailerCallbacks
  module ControllerExtensions
    def self.included(base)
      base.after_action do |controller|
        ActiveSupport::Notifications.instrument(
          "mailer_callbacks.#{controller_path}##{action_name}", controller: controller
        )
      end
    end
  end

  module Listener
    def listen_to(action, &block)
      ActiveSupport::Notifications.subscribe("mailer_callbacks.#{action}") do |*args|
        event = ActiveSupport::Notifications::Event.new(*args)
        controller = event.payload[:controller]
        controller.instance_eval(&block)
      end
    end
  end
end
{% endhighlight %}

Now all we need is to create an initializer and register our mailer callbacks:

{% highlight ruby linenos %}
# config/initializers/mailer_callbacks.rb
ActiveSupport.on_load(:action_controller) do
  include MailerCallbacks::ControllerExtensions
end

class MailerListeners
  extend MailerCallbacks::Listener

  # register as many listeners as you would like here

  listen_to 'posts#create' do
    PostMailer.notify(@post).deliver if @post.persisted?
  end
end
{% endhighlight %}

From a code maintenance point of view, most if not all of our mailer invocations could now be found in a single source file.

This is a general purpose pattern that I've adopted in different contexts that allows me to keep my controllers skinny and
isolate independent concerns.


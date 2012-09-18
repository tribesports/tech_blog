---
layout: post
title: "Pub/Sub system events for post-action tasks"
date: 2012-08-21 20:33
comments: true
author: "Simon Coffey"
categories: [Rails, Controllers, Pub/Sub]
---

## TL;DR

Rails controllers are often a dumping ground for unrelated tasks. We cleaned ours up using a pub/sub system of announced events and task-specific listeners. This made our app easier to modify, easier to test, and easier to read. Probably.

## Introduction

In many Rails applications, controllers end up cluttered with a wide array of post-action tasks: sending emails; posting to Facebook; custom analytics/logging, and more.

These cross-cutting concerns (and the logic associated with them) can end up dominating controller actions, to the point where the controller can no longer be easily tested, and the main purpose of the action is hard to discern. The following might be an example, although we had much more complex actions:

{% codeblock app/controllers/comments_controller.rb %}
class CommentsController < ApplicationController

  def create
    comment_params = params[:comment].merge(:creator => current_user)
    @comment = Comment.new(comment_params)
    if @comment.save
      unless current_user == @comment.commentable.creator
        UserMailer.comment(@comment.commentable.creator, @comment).deliver
      end
      if current_user.twitters? && current_user.setting("twitter_post_comment")
        TwitterPoster.new(current_user).post(@comment)
      end
      current_user.reward_for('create_comment')
    end
    respond_with(@comment)
  end

end
{% endcodeblock %}

The majority of the action is concerned with doing things only indirectly related to the actual creation of the comment. As a result there are multiple code paths caused by the conditional logic that are laborious to exhaustively test (and slow, because controller tests are slow).

Some of the options for reducing this complexity are to move these associated tasks into:

1. ActiveModel lifecycle callbacks
2. ActiveModel observers
3. Controller filters
3. Service objects
4. Pub/sub system events

Anyone who has made extensive use of lifecycle callbacks for this sort of thing will be familiar with the problems that triggering behaviour with implicit events can cause (particularly when testing or debugging) - we're looking for an explicit mechanism where it will still be obvious in the controller that our intention is to trigger associated actions. And just as we don't want to have our controllers responsible for all of these secondary concerns, we don't want to make our mode responsible for them either.

We avoid ActiveModel::Observers for much the same reasons, and while before/after filters in Rails' controllers are marginally more explicit, being at least localised to the controller doing the action, we've found that when they start stacking up they still tend to cause surprising and hard-to-debug problems.

This leaves us with:

## Service objects

We could choose to create a service object that is responsible for handling these post-commenting tasks. For instance, we might place our comment creation logic in its own class as follows:

{% codeblock lang:ruby %}
# app/controllers/comments_controller.rb
class CommentsController < ApplicationController

  def create
    @comment = CommentCreationService.new.create(params[:comment])
    respond_with(@comment)
  end

end

# app/services/comment_creation_service.rb
class CommentCreationService

  def create(params)
    comment = Comment.new(params)
    if comment.save
      mail_user(comment)
      post_to_twitter(comment)
      reward_user_for(comment)
    end
    comment
  end

  def post_to_twitter
    # ...
  end

  # ...

end
{% endcodeblock %}

This cleans up the controller, and the service object itself is more amenable to testing, but we're still left with an object with far too many collaborators, addressing far too many unrelated concerns. Service objects may well be useful if the actual creation of a comment involves more complex manipulation than a simple `#create` call, but they probably shouldn't be used as a dumping-ground for tasks that aren't directly involved in the creation of a `Comment`.

## Pub/Sub

To decouple post-action tasks from the events that prompted them, we've used a pub/sub system. Controllers (or service objects) announce system events via a central message broker (we've called it an "announcer"), while specialised listeners register their interest in particular events with the announcer, and respond to those events appropriately.

First, let's take a look at the announcer:

{%codeblock lib/announcer.rb %}
class Announcer

  attr_reader :listeners

  def initialize
    @listeners = []
  end

  def add_listener(listener)
    @listeners << listener
  end

  def announce(message, *args)
    listeners.each do |listener|
      listener.send(message, *args) if listener.respond_to?(message)
    end
  end
end
{% endcodeblock %}

{% codeblock spec/lib/announcer_spec.rb %}
require_relative '../../lib/announcer'

describe Announcer do
  let(:announcer) { Announcer.new }

  describe "#add_listener" do
    it "adds a listener to the Announcer's listeners" do
      listener = mock("Listener")
      announcer.add_listener(listener)
      announcer.listeners.should include listener
    end
  end

  describe "#announce" do
    let(:listener) { stub("Listener")                 } 
    before         { announcer.add_listener(listener) } 

    it "notifies each listener with the given message" do
      listener.stub(:respond_to?).with(:hello).and_return(true)
      listener.should_receive(:hello).with("some", "args")

      announcer.announce(:hello, "some", "args")
    end

    it "does not notify listeners that don't respond to the given message" do
      listener.stub(:respond_to?).with(:hello).and_return(false)
      listener.should_not_receive(:hello)

      announcer.announce(:hello, "some", "args")
    end
  end
end
{% endcodeblock %}

This allows listeners to be registered, and sends any announcements it receives to interested listeners, where "interested" is defined as "responds to the announced message". If an event is announced that no-one is interested in, it will be silently dropped.

Now let's take a look at an example listener:

{% codeblock app/listeners/mail_listener.rb %}
class MailListener

  def initialize(announcer, user_mail_job_klass = MailJob)
    @user_mail_job_klass = user_mail_job_klass
    announcer.add_listener(self)
  end

  def create_comment(comment)
    commentable = comment.commentable
    unless comment.creator == commentable.creator
      mail(:comment, commentable.creator, comment)
    end
  end

  private

  def mail(*args)
    @user_mail_job_klass.new(*args)
  end

end
{% endcodeblock %}

{% codeblock spec/listeners/mail_listener_spec.rb %}
require_relative '../../app/listeners/mail_listener'

describe MailListener do
  let(:job_klass) { stub_everything("Class:MailJob") } 
  let(:listener)  { MailListener.new(job_klass)      } 

  describe "#create_comment" do
    let(:user)        { stub("User")                          } 
    let(:commenter)   { stub("User:commenter")                } 
    let(:commentable) { stub("Commentable", :creator => user) } 
    let(:comment)     { stub("Comment",
                             :creator => commenter,
                             :commentable => commentable) } 

    it "sends a comment email to the commentable's creator" do
      job_klass.should_receive(:new).with(:comment, user, comment)

      listener.create_comment(comment)
    end

    it "does nothing if the commenter created the commentable" do
      commentable.stub(:creator => commenter)
      job_klass.should_not_receive(:new)

      listener.create_comment(comment)     
    end
  end
end
{% endcodeblock %}

We initialize the listener with an announcer, and the listener registers itself. Our `MailListener` class is interested in handling the `:create_comment` event, so we define a `#create_comment` method with the appropriate signature. Then we do whatever we want: in this case, emailing the owner of the entity being commented on to let him know that someone has posted a new comment. We also handle the case where the owner has commented on his own item, and we write a test case.

We inject the mail job class in the listener's constructor rather than create a hard dependency - as seen in the associated tests, this allows us to pass in a test double on which we can place message expectations. We supply a default argument in the `#initialize` method, however, so we don't always have to pass in the `MailJob` class in normal use.

## Integration

Now we've got an announcer that will dispatch messages to interested parties, and a fully-tested listener that responds to a particular event. We just need to hook everything together, and announce our event in the controller.

For controllers to announce events, we need to put together an `Announcer` instance for them to use. In our `ApplicationController`, we add the following:

{% codeblock app/controllers/application_controller.rb %}
class ApplicationController < ActionController::Base
  # ...
  def announcer
    @announcer ||= build_announcer
  end

  def announce(*args)
    announcer.announce(*args)
  end

  private

  def build_announcer
    announcer = Announcer.new
    MailListener.new(announcer)
    announcer
  end
end
{% endcodeblock %}

We've got the infrastructure to announce events - now our original controller action becomes the following (let's assume we've written similar listeners for the other post-action tasks):

{% codeblock app/controllers/comments_controller.rb %}
class CommentsController < ApplicationController

  def create
    comment_params = params[:comment].merge(:creator => current_user)
    @comment = Comment.create(comment_params)
    announce(:create_comment, @comment) if @comment.persisted?
    respond_with(@comment)
  end

end
{% endcodeblock %}

{% codeblock spec/controllers/comments_controller_spec.rb %}
require 'spec_helper'

describe CommentsController do
  before do
    @user = FactoryGirl.create(:user)
    sign_in(@user)
  end

  describe "POST 'create'" do
    it "creates a comment and announces a :create_comment event" do
      comment = mock_model(Comment, :persisted? => true)
      comment_params = { 'commentable_type' => 'Activity',
                         'commentable_id' => 5,
                         'comment' => 'hello' }
      Comment.should_receive(:create).
        with(comment_params.merge(:creator => @user)).
        and_return(comment)

      post('create', 'comment' => comment_params)
      announcer.should have_announced(:create_comment).with(comment)
      response.should redirect_to(comment)
    end

    it "renders the 'new' template and announces nothing if the comment was not saved" do
      comment = mock_model(Comment, :persisted? => false)
      Comment.stub(:create => comment)

      post('create', 'comment' => {})
      announcer.should_not have_announced(:create_comment)
      response.should render_template('new')
    end
  end
end
{% endcodeblock %}

Our controller action is now much cleaner. It manipulates user input, uses it to create the comment, then announces the relevant event and hands over to Rails' responder to handle rendering or redirecting as appropriate. And that's it. It doesn't need to know about emails, or posting to Twitter, or site gamification rewards, or anything other than creating a comment.

Our test is also simpler, and fully covers the action's logic. We're using a stub announcer and a custom RSpec matcher to allow us to test whether events have been announced, but we won't go in to the details of that in this post. Each test contains multiple assertions, which isn't great, but controller tests run slowly enough that it can be worth making an exception to this rule of thumb.

## Integration Tests

Our listeners are now decoupled from our controller, and our controller tests examine only the controller's behaviour. So while we've gained coverage in some areas, we've lost some elsewhere. Our original controller tests, despite being gross, slow and unwieldy (not to mention incomplete) did at least exercise quite a lot of our application stack.

What to do about this? Well, write some *proper* integration tests, as opposed to integration tests masquerading as unit tests. In our case, we've expanded our cucumber suite, making sure that at least one scenario exercises each of our listeners' core functionality. This way we can be confident that we've hooked things up properly. The level of coverage we're applying depends on how critical the functionality is. Things that drive user interaction are of paramount importance, like emails and activity recording. Other things (like custom analytics) might not be so important.

## Conclusions and caveats

Okay, so we've done all this work. So what? What's better about our system now? We feel the pros and cons break down as follows.

**Advantages**:

1. Controller actions aren't cluttered up with tangential tasks
2. Logic related to mailing, twitter posting etc. is kept in one place
3. Said logic can be tested in isolation from the controllers
4. Controller tests are simpler because you're dealing with fewer immediate collaborators
5. Adding a new type of post-action task (e.g. analytics handling) only requires a new listener - we don't need to make changes in multiple controller actions
6. It's now easier to make responses to system events asynchronous, speeding up our page render times.

**Disadvantages**:

1. It's no longer so easy to see at a glance all of the things that happen in response to a given controller action
2. Because the listeners are more loosely coupled, and tested in isolation from the controllers, integration tests become more important.

The disadvantages are not to be dismissed. Controller tests are simpler, but they're also exercising fewer of our app's components (since we're providing a stub announcer that does no forwarding), so coverage is decreased. Listener logic may be thoroughly tested, but in isolation: we need to make sure that we're putting these loosely-coupled components together properly in our full app, and that the listeners interact properly with real collaborators, as opposed to our test doubles. This places a greater responsibility on our integration tests.

The advantages, however, are really significant. A change to, for example, our user activity recording now no longer requires changes in tens of controller actions; those changes are localised to one place, and we no longer need to fret about finding every instance of activity recording in our app. Lots of logic that was previously untestable is now covered in listener tests, increasing our confidence in the correct behaviour of our app. And these tests are blazing fast, because they don't depend on Rails.

Looking further forward, we're likely to want to move these post-action tasks to an asynchronous system, to get all but the most critical actions out of our render path. Having a single point of entry for system events (the announcer) makes it far easier for us to achieve this.

## Should you do this in your app?

Maybe.

This was a response to our controllers having built up a thick crust of cross-cutting concerns during the lifetime of our app. At the moment we've got 8 listeners handling around 150 different post-action tasks, and there's more work to be done. For us, the benefits of moving to this system have been pretty clear already. For a smaller app, or one with a shorter projected lifespan, this approach may well be overkill.

It should also be pointed out that we've only been living with this system for a short while - it remains to be seen how it pans out in the long run. So far, though, so good.

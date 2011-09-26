---
layout: post
title: "Versioning the Tribesports API"
date: 2011-09-24 09:44
comments: true
author: "Simon Coffey"
categories: [API, Rails, Responders, Renderers]
---

As Andrew is hard at work on Tribesports' upcoming iPhone app, we've been paying a bit of attention to our API. Where previously we had responded to JSON requests in a rather ad-hoc fashion according to our application's internal needs, a full-blown API requires a more systematic approach.

Our goals are as follows:

1. Allow clean API versioning
2. Separate API presentation logic from business models
3. Minimise additional controller complexity

Like all good REST disciples, we know that adding versioning information to URLs will cause the breakdown of civil society, so we define a custom MIME type for use in the Accept header of requests:

{% codeblock config/initializers/mime_types.rb %}
Mime::Type.register "application/vnd.tribesports.api.v1+json", :api_v1
{% endcodeblock %}

This means Rails will now treat our `:api_v1` format as a first-class citizen, so we can use it in controller response blocks, like so:

{% codeblock app/controllers/activities_controller.rb %}
class ActivitiesController < ApplicationController
  respond_to :html, :api_v1

  def show
    @activity = Activity.find(params[:activity_id])
    respond_with(@activity) do |format|
      format.api_v1 { render :text => @activity.to_json }
    end
  end
end
{% endcodeblock %}

Already, then, we're able to recognise and respond appropriately to requests for our custom MIME type. There are two problems here, however. Firstly, we're left with controller-level boilerplate that must be replicated in every method that responds to API calls. And secondly, we're falling back on our business model's default `#as_json` method, which will render the entire object. We could override it, or pass in options, but the former conflicts with our versioning requirement while the latter would spread presentation logic throughout our controllers.

To address these problems, we look to Rails 3's responders and renderers. If we don't supply a `format.api_v1` block in the response block, Rails' [default Responder](https://github.com/rails/rails/blob/master/actionpack/lib/action_controller/metal/responder.rb) will attempt to intelligently render the resource in the given format. The logic for a simple `show` action varies depending on the format type, and runs roughly thus for a given resource:

  * **Navigational formats** (e.g. HTML): Render using the appropriate template (e.g. `'activities/show.html.haml'`)

  * **API formats** (everything else): If the resource responds to a `#to_#{format}` method, and a renderer exists for the format, then use the renderer; otherwise, attempt to render using an appropriate template.

For update and create methods there is additional logic to deal with redirection and error handling, but this represents the basics.

To make our custom format work properly with the responder logic, we therefore need to define a `#to_api_v1` method on all our models, and define a custom renderer so Rails will know what to actually do with this method.

Let's have a look at the built-in [JSON renderer](https://github.com/rails/rails/blob/v3.0.10/actionpack/lib/action_controller/metal/renderers.rb#L73):

{% codeblock actionpack/lib/action_controller/metal/renderers.rb %}
module ActionController
  module Renderers
    # ...

    add :json do |json, options|
      json = json.to_json(options) unless json.kind_of?(String)
      json = "#{options[:callback]}(#{json})" unless options[:callback].blank?
      self.content_type ||= Mime::JSON
      json
    end
  end
end
{% endcodeblock %}

We can see the renderer is passed the resource (potentially as an already-JSON-encoded string) and a set of options. If the resource isn't already encoded, the renderer encodes it using the supplied options. It then wraps it in an optional callback, sets the response MIME type, and returns the output. So our custom renderer might appear as follows:

{% codeblock lang:ruby %}
# app/renderers/api_v1_renderer.rb 
ActionController::Renderers.add :api_v1 do |resource, options|
  self.content_type = Mime::API_V1
  self.response_body = resource.to_api_v1(options)
end

# app/controllers/application_controller.rb
require_relative '../renderers/api_v1_renderer'

class ApplicationController < ActionController::Base
  #...
end
{% endcodeblock %}

Now all we have to do is define `#to_api_v1` on every model we want to represent in the API (yes, yes, we know - a horrible mixing of concerns. More on that later.):

{% codeblock app/models/activity.rb %}
class Activity
  #...

  def to_api_v1(options)
    {
      id: id,
      content: content,
      creator_name: creator.name
    }.to_json(options)
  end
end
{% endcodeblock %}

As a result of all that, there's no longer any need for the custom response block in our controller methods:

{% codeblock app/controllers/activities_controller.rb %}
class ActivitiesController < ApplicationController
  respond_to :html, :api_v1

  def show
    @activity = Activity.find(params[:activity_id])
    respond_with(@activity)
  end
end
{% endcodeblock %}

We can now send a request using our custom MIME type in the Accept header, and [receive an appropriate response](https://gist.github.com/1239258).

So far, so good. We've got the Rails responder doing all our heavy lifting, our controllers are clean, and we have versioned resource presentation. That's two of our goals accomplished, but we've still got presentation logic cluttering up our business models. To address that we'll be using Decorators, which will be the subject of our next blog post.

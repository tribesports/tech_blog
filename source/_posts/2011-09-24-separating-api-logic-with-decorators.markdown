---
layout: post
title: "Separating API Logic With Decorators"
date: 2011-09-24 13:49
comments: true
published: false
author: "Simon Coffey"
categories: [API, Rails, Decorators]
---

In our [last post](/blog/2011/09/24/versioning-the-tribesports-api) we demonstrated how to use Rails 3's response and rendering system to provide a cleanly-versioned API. By defining a custom MIME type and a renderer for that type, we were able to version our API, and handle responses without any changes to our controllers.

However, we were left with unsightly `#to_api_v1` methods on all of our business models. We'd like to separate presentation logic from our business logic, and to do that we're going to use decorators. We're making use of Jeff Casimir's [draper](https://github.com/jcasimir/draper) gem, which provides some nice convenience methods for decorating Rails models.

We start by creating a namespaced decorator for our Activity model, and move the `Activity#to_api_v1` method into it:

{% codeblock app/decorators/api_v1/activity_decorator.rb %}
module API_V1
  class ActivityDecorator < Draper::Base
    decorates :activity

    def to_api_v1(options = {})
      {
        id: model.id,
        content: model.content,
        creator_name: model.creator_name
      }.to_json(options)
    end
  end
end
{% endcodeblock %}

This is much better - our presentation logic is out of our model. However, we've created a problem: Rails' default Responder was relying on the `Activity#to_api_v1` method being present to detect whether it was capable of rendering an Activity instance in our API format. We need to create a custom responder that will also render a resource if an appropriate Decorator can be found.

The responder uses [the method `#resourceful?`](https://github.com/rails/rails/blob/v3.0.10/actionpack/lib/action_controller/metal/responder.rb#L173) to determine whether a resource can be rendered in the given format (we have no idea what the MacGuyverish name is about). We can override this method:

{% codeblock lang:ruby %}
# app/responders/application_responder.rb
class ApplicationResponder < ApplicationController::Responder
  def resourceful?
    super || ApplicationDecorator.has_decorator_for?(resource, format)
  end
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  self.responder = ApplicationResponder
  # ...
end
{% endcodeblock %}

This makes use of some decorator lookup methods that we provide in our application's base decorator class:

{% codeblock app/decorators/application_decorator.rb %}
class ApplicationDecorator < Draper::Base

  def self.has_decorator_for?(resource, format)
    true if decorator_class_for(resource, format)
  end

  def self.decorator_for(resource, format)
    if decorator_class = decorator_class_for(resource_format)
      decorator_class.decorate(resource)
    end
  end

  private

  # (<#Activity 1>, :api_v1) => API_V1::ActivityDecorator
  def self.decorator_class_for(resource, format)
    namespace_for(format).const_get(decorator_name_for(resource))
  rescue NameError
    nil
  end

  def self.namespace_for(format)
    Object.const_get(format.to_s.upcase)
  end

  def self.decorator_name_for(resource)
    resource.class.name + "Decorator"
  end

end
{% endcodeblock %}

This means that if we're consistent with our decorator naming, we can look up the appropriate decorator according to resource class and format. The above is slightly simplified - there are extra cases required if the resource is an array or an `ActiveRecord::Relation` (this will happen in index actions, for example), but these aren't particularly interesting.

Now we have a custom responder that will handle decorated resources properly; we have a decorator lookup system that will return an appropriately-decorated resource according to the requested format; and we have the decorator itself. All we need to do now is modify our renderer so that it will decorate a resource appropriately before rendering it.

{% codeblock app/renderers/api_v1_renderer.rb %}
ActionController::Renderers.add(:api_v1) do |resource, options|
  self.content_type = Mime::API_V1
  decorated_resource = ApplicationDecorator.decorator_for(resource, :api_v1) || resource
  render options.merge(:json => decorated_resource)
end
{% endcodeblock %}

This is largely self-explanatory; the main point to note about this renderer is that if it can't find an appropriate decorator, it will fall back on the default json rendering. Isn't that what we wanted to avoid? Yes, but with a caveat. If a resource has errors (for example, after an invalid update), the responder doesn't render the resource itself; it renders the errors. We therefore need to handle the case when the renderer is passed an array of errors, when we simply want to send the JSON representation of that array.

Finally, since we're using the built-in JSON renderer to actually create the output, we modify our decorators to respond to `#as_json` instead of `#to_api_v1`:

{% codeblock app/decorators/api_v1/activity_decorator.rb %}
module API_V1
  class ActivityDecorator < ApplicationDecorator
    decorates :activity

    def as_json(options = {})
      {
        id: model.id,
        content: model.content,
        creator_name: model.creator_name
      }
    end
  end
end
{% endcodeblock %}

To recap, we have:

1. Defined a custom MIME type to inform Rails about our API format,
2. created a custom responder to let Rails know what resources can be renderered,
3. created a custom renderer to decorate our resources before rendering, and
4. built a decorator lookup to locate the appropriate decorator for each resource.

As a result, we're able to keep our API presentation logic completely separate, versioned using format-specific namespaces. Adding a new API version is as simple as defining a new MIME type, renderer and the appropriate decorators. And finally, our controller actions are completely untouched - no complex format-specific logic.

We've achieved all of our [original goals](/blog/2011/09/24/versioning-the-tribesports-api), and with only a few tens of lines of code. Time for a particularly smug cup of tea; the Tribesports office favours Oolong for this purpose, although Lapsang Souchong will do in a pinch.

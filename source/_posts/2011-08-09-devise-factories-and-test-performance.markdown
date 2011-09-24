---
layout: post
title: "Devise, Factories and Test Performance"
date: 2011-08-09 15:15
comments: true
published: false
author: "Simon Coffey"
categories: [Devise, Rspec, Performance]
---

(Or, How We Halved Our Specs' Runtime)
--------------------------------------

There are many masochists at [Tribesports](http://tribesports.com). Arnaud, on the morning of his first marathon, felt it an insufficient challenge, and ran the 10km from his house to the starting line. Joe contends with enough Internet Explorers to power a constructive dismissal lawsuit on a daily basis. And despite the fishy miasma emanating from the alley outside our window, we all trudge dutifully back to our desks each morning.

We draw the line, however, at two things: long-running tests, and Steve's taste in music. As our project has progressed, our specs have grown, and even with luxuries like [Spork](https://github.com/timcharper/spork/) and [Guard](https://github.com/guard/guard), things have started to get out of hand.

While updating some fairly innocuous mailer specs today, I was struck by just how long each example took. We use [Rspec](http://relishapp.com/rspec), combined with [Factory Girl](https://github.com/thoughtbot/factory_girl) for fixtures and mocha for mocks and stubs.

At [Tribesports](http://tribesports.com) we follow the golden rule of rolling your own authentication: Don't Roll Your Own Authentication. Instead, we're happy users of Devise, and like all good Devise users we use the default password hashing algorithm, bcrypt.

Bcrypt is designed to be work-intensive, so that brute-force attacks are prohibitively expensive. In the boilerplate devise initializer there's a setting that allows you to specify a work factor:

{% codeblock config/initializers/devise.rb %}
Devise.setup do |config|
  # ...
  config.encryptor = :bcrypt
  config.stretches = 10
  # ...
end
{% endcodeblock %}

On my development machine, an i5 Macbook Pro, it takes approximately 0.15 seconds to compute a password hash with these settings.

---
layout: post
title: "Reducing Rails deploy times"
date: 2012-11-14 19:38
comments: true
author: "Simon Coffey"
categories: [Rails, Capistrano, Deployment, Performance]
---

Rails deploy times can easily get out of hand as they become more
complex. As our Capistrano deployments approached the 6-7 minute range
in the worst case, we found some (hopefully) general principles that
helped us get our deploy time down:

1. Combine related tasks
2. Boot Rails less
3. Minimise asset recompilation

<!--more-->

## Combine related tasks

This is probably the most easily generalised lesson we learned. Where it
makes sense to combine tasks, overhead can be saved, whether it's simply
in ssh connections to servers, or in rake startup times.

Take symlinking as an example. Capistrano creates a set of default
symlinks for configuration and directories that are persistent across
deploys. You'll then almost certainly have a number of custom files that
also need to be symlinked into the release directory. We stole [Github's
approach](https://github.com/blog/470-deployment-script-spring-cleaning)
and combined these steps into a single shell command, as opposed to
having something like the following:

{% codeblock config/deploy.rb %}
task(:custom_symlink) do
  run "ln -s #{shared_path}/config/database.yml #{release_path}/config/database.yml"
  run "ln -s #{shared_path}/cache" #{release_path}/tmp/cache"
  # etc
end
{% endcodeblock %}

Each `#run` invocation opens a new ssh connection to each server, which
is a bit unnecessary. Defining your symlinks as a hash and composing a
single shell command to create them is considerably more efficient:

{% codeblock config/deploy.rb %}
set :custom_symlinks {
  "config/database.yml" => "config/database.yml",
  "cache"               => "tmp/cache",
  # etc
}

task(:custom_symlinks) do
  commands = custom_symlinks.map do |from, to|
    "rm -rf #{release_path}/#{to} && ln -sf #{shared_path}/#{from} #{release_path}/#{to}"
  end

  run "cd #{release_path} && #{commands.join(" && ")}"
end
{% endcodeblock %}

This approach yields particular benefits when rake tasks are being
invoked, as it potentially reduces the number of rails boot times
required by your deploys. Speaking of which...

## Boot Rails less

Rails boot times are obviously highly undesirable. Wherever possible,
avoid invoking rake tasks in your deployment that depend on the
`environment` task.

However, you may not realise that by default, a significant proportion
of Rails gets loaded even for tasks that *don't* require the environment
to be loaded. In the default Rails `Rakefile`, we see the following:

{% codeblock Rakefile lang:ruby %}
#/usr/bin/env rake

require File.expand_path('../config/application', __FILE__)

Tribesports::Application.load_tasks
{% endcodeblock %}

This means that Rails and at least some of your application are being
booted for each rake invocation in your project directory, regardless
of whether either is needed. To see the effects of this, let's try the
following:

{% codeblock lib/tasks/noop.rake lang:ruby %}
task(:noop)
{% endcodeblock %}

```
$ time rake noop

real    0m4.927s
user    0m3.198s
sys     0m1.717s

$ time rake -f noop.rake noop

real    0m1.184s
user    0m0.917s
sys     0m0.271s
```

By forcing rake to ignore our project's `Rakefile` using the -f option,
we quartered the execution time. On my machine, 5s is over half Rails'
boot time, and that's with development lazy loading, stupid GC settings
and an SSD. In production our boot time is up around 20s, and the
speedup achieved by avoiding the default Rails loading is even greater.

Avoiding Rails boots on its own is all very well, but often a task will deliberately load the environment to access (for example) `Rails.root` or `Rails.env`.

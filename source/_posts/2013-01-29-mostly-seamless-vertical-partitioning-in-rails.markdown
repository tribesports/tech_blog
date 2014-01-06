---
layout: post
title: "(Mostly) seamless vertical partitioning in Rails"
date: 2013-01-29 15:37
comments: true
author: "Simon Coffey"
categories: [Rails, Partitioning, ActiveRecord, Performance]
---

TL;DR: Tables, like objects, can become gross and bloated. We show a way
to split up a table backing an ActiveRecord model with minimum changes
to code using the model in question.

Most developers have encountered a God Object or two in their travels.
More like Greek or Norse gods than the current, rather ascetic crop,
God Objects roam freely through applications, taking many forms and
gleefully coupling with all and sundry.

When your business objects are also your persistence objects, as in
Rails, this tendency comes with a counterpart: the God Table. If all
your behaviour is in one class, it's seductively convenient to have all
the data it needs sitting right there.

While this is all very well when you've just got a handful of rows,
and may indeed speed up your development at first, you'll run into
performance problems sooner or later if one of your tables gets wide as
well as long. And if your God Object has been as prolific as they tend
to be, then there may be a great deal of code coupled to this particular
database representation. This in turn stymies your efforts to split your
table into more sensible chunks.

This blog post details an approach to allow semi-transparent vertical
partitioning as a *strictly intermediate step* in refactoring your
God Objects in Rails. We will aim to split a large table into several
smaller ones, while preserving as much of the original model's interface
as possible, minimising the amount of client code that needs to change.
Essentially, we decouple the God Object problem from the God Table
problem, allowing them to be tackled separately.

<!--more-->

## Background

The obvious God Object candidate in a typical Rails app is the User
model. Maybe it started with just some authorization information; then
profile pages were required, so a bio field got added; some users like
to write essays, so it was made a `TEXT` column. Later on some external
integration was added, so the table picked up some credentials for a
third-party site. Throw in some more strings for 

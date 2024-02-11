+++
title = "Class based views are always better*"
date = 2021-06-11T21:48:07.678Z
tags = ["django", "english"]
slug = "django-class-based-views-are-better"
+++

In the Django community, there is this long-running discussion that juxtaposes
view styles that are offered by the web framework. One is a good-old function
based style and the other is class based style. I certainly will not settle
this discussion by asserting my ideas here, but I’ll try to convince you at
least.

Now, at some point in the framework, the only way to write views was using the
functions. Class-based views were introduced with Django 1.3 along with a
_migration_ guide. You can see the generic old-style function views
[here](https://github.com/django/django/tree/stable/1.1.x/django/views/generic).

Inspect these views and think about possible drawbacks; you needed endless
numbers of function chains with long-running parameters, just to implement a
generic view with some custom logic. I’m pretty sure everybody acknowledges
that class based views brought simplicity to generic views, so I will not dwell
on that much. I am also pretty sure everyone will acknowledge that CBV are
(obviously): more readable, more scalable and more extendable.

Here are some common assertions made to justify function-based usage, I have
provided my responses below.

## My logic is so simple!

If your logic is simple, you better use the most primitive base class `View`.
It’s just a plain wrapper that provides get and post methods. You would be more
consistent if the remaining views were CBV, you would worry less about
future-changes that might introduce some complexity.

## Generic views just don’t fit my purposes!

If your logic is too complicated, and you can’t find a corresponding CBV
you’re probably lying or doing something wrong. Just be realistic, at the end
of the day you do one of these: create, read, update and delete, which all have
corresponding views. Let’s say your point is fair and your logic indeed
doesn’t fit into anything, then how come such complexity will be handled in a
function-based view, with endless indentation and conditional logic? Just use
`View` (at worst) and OOP structures to make sure your logic stays
understandable.

## Class based views take up too much space

In some cases, especially when writing simple logic, CBV take more space than
function-based views. But you don’t really count the amount of lines you save
up, do you? Let’s say you did and CBV still had longer lines. Then what? It’s
a good compromise; CBV promotes scalability and readability.

## Class based views are too complicated!

CBV are not complicated, they are complex. It requires a good understanding of
object-oriented paradigm and that’s why being able to use them makes you a
better Django developer. If you understand CBV, you will understand views that
make use of it; if you continue using function-based views, you will be writing
_complicated_ views that even you won’t understand some three months later.

## All the logic is hidden, I don’t understand anything!

All the logic is indeed hidden and that’s one of the things we actually want;
we want to reach  **business logic**  as soon as possible and anything
unrelated can remain hidden at the depths of abstraction (that’s why you use
Django in the first place). Django is open source, just delve into the code and
figure out what is being done.

## How come they didn’t remove function-based views if CBV is superior?

Well, at the heart of every web framework lies a callable that takes a request
and returns a response; that’s just how they work. You just can’t remove this
key internal mechanism; CBV are just wrappers around this structure.

*Besides, it has one great use case:  **learning**. If you are just starting
Django, you better start with function-based views to grasp the mechanisms of
request-response cycle and how they are handled; generic CBV will handle these
for you later.

_I will keep adding my responses if I hear other arguments against CBV._

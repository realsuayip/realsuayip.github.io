+++
title = "Don't develop your app to support Django admin"
date = 2024-04-01T17:56:49Z
tags = ["django", "english"]
slug = "django-admin-development"
+++

The Django admin site is a very useful application, bundled with Django itself.
It's considered one of the selling points of the framework.

Having CRUD stuff ready once you write your models is really convenient. I
mainly use Django admin for these 3 purposes:

1. Debugging and testing in my local machine. For example, creating & updating
   instances and checking things created via endpoints.

2. Troubleshooting in production. Basically the same thing I do in my local
   machine, however in this case, its mostly read operations.

3. Management of system-wide tables. These include periodic celery task
   definitions, and the "settings" table if I have it.

I try not to touch any other table, including the user table. This is because
Django admin works as **a separate endpoint to your database.** And if you
don't build around supporting the admin site, using it might result in some
business logic not firing.

For example, let's say you have an REST API endpoint to create a user. After
the creation of the user, an email containing a welcome message is sent. If you
create a user through Django admin site, this won't happen since your code only
makes email calls in your API view. Now you need to duplicate this logic in
model admin class to achieve same behavior.

> OK, what about using signals to get rid of this duplication? In fact, this
> way we can send welcome email regardless of creation context.

That works, but it has a caveat: signals are a mess; they are quite difficult
to scale and optimize. Of course that one signal to send email won't cause any
problems but keep in mind that you are trying to keep your whole application
running in Django admin via signals.

Regardless, there will be some business code that will certainly be impossible
to execute through chain of signals such as big transaction blocks.

> What about services? We can write some reusable services (e.g., model
> managers) that will get called in both contexts.

This is more likely, but it will still take a lot of work. Since you are in the
context of Django admin, you'll inevitably need to write some custom logic
since you are not just returning a JSON response. You'll probably need to add
custom templates to render computed data and to take custom input that is not a
model field.

Moral of the story is, don't waste your time retrofitting your application to
admin. In fact, *assume* that there is no Django admin in production, and it is
just a development tool for your convenience.

If you are determined, know that you are writing **another endpoint** for your
application, possibly doubling the development time. Otherwise, Django admin
is still powerful for your small management and troubleshooting needs.

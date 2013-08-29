---
layout: post
title: Let's build a framework
subtitle: Request for comments
---

<div>
    <a href="https://news.ycombinator.com/item?id=6294558"><img src="/static/hn.gif" alt="Published to Hacker News"></a>
    <a href="http://www.reddit.com/r/programming/comments/1lbddo/rfc_lets_build_a_nodejs_framework/"><img src="/static/reddit.gif" alt="Published to Reddit"></a>
</div>

I've been developing mostly in Python for the last 7 years or so. Even though I
did some JavaScript, I never built big, scalable applications. Quite recently
the company I worked for started a new project in NodeJS and I wanted to take
part of it.

NodeJS is something that I've been wanting to get my hands on for a long time.
There is so much fuzz going on that I felt curious about it, and now I had the
perfect excuse to do it.

After working with it for a few weeks I decided that the best way to learn was
to build something big on top of it, that would make use of Node's internals as
well as V8's optimizations. So I decided to build a framework.


## TL;DR ##

I built a framework for NodeJS, [Kolba](https://github.com/tooxie/kolba). It's
in a very early stage, a proof of concept of what I want in a framework.  What
makes Kolba different is that it includes a thread-locals concept, which allows
you to access the request from any point in your application.

Of course NodeJS is single threaded, but there are undocumented ways of
achieving such behaviour. This post is mostly a question to the NodeJS
community about why is this useful feature so well hidden.

For a detailed description of all the features, see
[Kolba's documentation](https://github.com/tooxie/kolba).


## What's out there ##

I was a little frustrated with [express](http://expressjs.com), I must admit.
Provides too little structure despite of calling itself a _framework_. Let me
give you an example.

The response includes a [send()](http://expressjs.com/api.html#res.send) method
that just send whatever is given, right to the client. There is no response
object where you save your headers, construct your body and flush them at the
end. Which leads to errors like:

`Error: Can't set headers after they are sent.`

Reminds me of [something](http://stackoverflow.com/q/8028957).

After hearing a talk by [Rendr](https://github.com/airbnb/rendr)'s creator I
decided to give it a try, looked really promising. What I found is that, unlike
express, this _library_ enforces too many decisions on the developer.

Given their goal, which is to write an application that runs both on client and
server, it is completely understandable that they make some assumptions. It's
a very ambitious goal. The problem that I see with it is that you end up
"porting" the restrictions of the browser to the server.


## Meet Kolba ##

[Kolba](https://github.com/tooxie/kolba) is heavily influenced by flask's
simplicity.

{% highlight javascript %}
var Kolba = require('kolba');
var app = new Kolba.App();

app.resource('^/$', function() {
    return 'Hello Kolba!';
});

app.run(3000);
{% endhighlight %}

This is the simplest possible application. Let's see something more
interesting.


### Accessing the current request ###

From any point in your application you can access the current request:

{% highlight javascript %}
var request = require('kolba').getCurrentRequest();
{% endhighlight %}

This is handy but don't abuse it, don't make your code depend on a global. Read
more on [the magic behind getCurrentRequest()](https://github.com/tooxie/kolba#the-magic-behind-getcurrentrequest).


### Fully async ###

In an attempt to embrace Node's non-blocking nature, Kolba communicates
internally through [events](http://nodejs.org/api/events.html), which makes the
app completely async. Sequential execution (e.g. middlewares) is also async
thanks to promises.

Your resources can be async if you want it. Return a Promise/A+ compliant
object and Kolba will continue execution of the request as soon as your
promise resolves.


## Optimizations ##

You will notice that many objects make use of closures to emulate private
variables:

{% highlight javascript %}
AnObject function(args) {
    // Public method
    this.getArgs = function() {
        return args;
    };
}
{% endhighlight %}

This is usually considered a bad practice because creation of objects is slower
and heavier on RAM usage when those objects are reused. I still do it because:

1. Those objects are instantiated **only once**, when the applications is
laoded.
2. Improves readability. I believe that the fact that they are **inside** the
container object gives a strong visual cue that it **belongs** to that object.

All the objects that are created on a per-request basis (Request, Response and
RequestLocals) are optimized:

{% highlight javascript %}
AnObject function(args) {
    this.args = args;
}
AnObject.prototype.getArgs = function() {
    return this.args;
};
{% endhighlight %}


### The resources ###

{% highlight javascript %}
app.resource('^/users$', function() {
    // Generate full HTML page

    return html;
}, ['GET'], 'text/html');

app.resource('^/users$', function() {
    // Just send the raw data as JSON

    return json;
}, ['GET'], 'application/json');
{% endhighlight %}

As you can see, resources consist of:

* A mount point, defined as a regular expression
* A callback
* A list of accepted methods
* The content type that it returns

With this information Kolba checks the `Accept` header sent by the client and
matches it agains the available resources. If none matches, it returns a 406.

This is useful because when you go to `/users` with your browser, it will
request `text/html` content. Once your single page application loads, it will
request `application/json` instead and receive the users in raw JSON. This
means that on the first hit the page will be served by the server and the
consecutive renders will be done on the client.

Internally your application can (and should) abstract the common code away and
reuse it in both resources.


#### Parameter injection ####

Callbacks use a simplified version of
[AngularJS' dependency injection](http://docs.angularjs.org/guide/di). Any of
your callbacks can use the `request` or the `response` simply by declaring it
in the signature:

{% highlight javascript %}
app.resource('^/herp$', function(response) {
    // Do something with the response

    return response;
});

app.resource('^/derp$', function(request, response) {
    // Do something different

    return response;
});
{% endhighlight %}


#### Getting all async and shit ####

You can use any promise library that follows the Promise/A+ spec:

{% highlight javascript %}
var when = require('when');

app.resource('^/fruits$', function() {
    var deferred = when.defer();

    process.nextTick(function() {
        deferred.resolve(['orange', 'kiwi']);
    });

    return deferred.promise;
}, ['GET'], 'application/json');
{% endhighlight %}


#### Return types ####

You will notice that Kolba infers certain things from the types of the values
returned by resources. Numbers will be treated as status codes and strings as
the body. Read more on [how Kolba treats return values](https://github.com/tooxie/kolba#how-kolba-treats-return-values).


## Getting in the middle ##

Kolba provides ways of intercepting the request in different points of the
execution, and based on different conditions.


### Request middlewares ###

When a request comes in, the first thing that gets executed are the
`request middlewares`. These are useful for doing some global checks prior to
the execution of the resource. Let's say we want to deny access to our website
completetly:

{% highlight javascript %}
app.beforeRequest(function() {
    return 403;  // Forbidden
});
{% endhighlight %}

This is possible because the first middleware that returns a value (different
from `undefined`) aborts the execution of the request, and the resource is
never executed.


### Interceptors ###

The `on` method lets you intercept requests based on the status code of the
response. At this point the resource was already executed but the response was
not yet flushed.

{% highlight javascript %}
app.on(404, function() {
    return 'Sorry, what?';
});

app.on(500, function() {
    return 'Something went terribly wrong';
});
{% endhighlight %}

If the interceptor returns something different than `undefined`, that value
will be used as response body instead.


### Response middlewares ###

Once the resource returns, the `response middlewares` are executed. This is the
last thing that gets executed before the response is sent to the client.

{% highlight javascript %}
app.afterRequest(function(response) {
    response.setHeader('Content-Type', 'application/x-gzip');
    response.setBody(gzip(response.getBody()));

    return response
});
{% endhighlight %}

Same rule applies here, if something is returned, the original response is
discarded.


### Post mortem ###

Finally, once the response is sent we can do some cleanup or maintenance tasks
using post mortems:

{% highlight javascript %}
app.postMortem(function(response) {
    logging.logResponse(response);
});
{% endhighlight %}


## Questions to you ##

Now I present you with some questions that I asked myself while developing
Kolba:

* Even if sharing code between client and server is not the main goal, which
steps can the framework take to make this easier for the developer?
* What do you think of the `getCurrentRequest()` helper?
* Do you think the concept of thread locals is a good feature or is it actually
harmful?

I don't have a clear answer for this questions. Your feedback would be really
useful for me.


## What do you think? ##

All that was presented here are just ideas that I have about what I want in a
framework. Please review the code and comment ruthlessly on it. Fork the
project, issue PRs, report bugs. Feel free to request the addition (or removal)
of features.

Find me at twitter as <a href="https://twitter.com/tuxie_">@tuxie_</a> or reach
me by email at <a href="mailto:alvaro@mourino.net">alvaro@mourino.net</a>.

Tell me your ideas, what would you like in a framework, and let's build it
together.

<!-- vim:filetype=markdown
-->

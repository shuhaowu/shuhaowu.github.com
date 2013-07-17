title: Common pitfalls of AngularJS
published: no

[AngularJS](http://angularjs.com) is pretty awesome. It provides a set of tools
that allows for the creation of a product very very fast. At this point in
time, I've worked on two projects with it (and this is not just a simple todo
list app for demonstration. Rather, it has multiple components, views, and many
different functionalities) and I have to say, this will be my frontend framework
of choice for the foreseeable future. The biggest reason for this, for me, is
that I do not have to write a tonne of DOM manipulation code and that saves a
lot of time and headaches when developing a web app.

As good as AngularJS might be, the documentation is lacking on many fronts.
While the overall documentation should get you to approximately the right place,
it often leaves beginners frustrated as a key concept is not clearly documented,
if documented at all. I'm going to try to address this, based on hours of
frustration here and there. I'm sure reading the manual will give you most of
this as well, but this couldn't hurt. :)

--------------------------------------------------------------------------------

\#1: No module named 'myapp'
----------------------------

This is a problem that I've personally had and have seen some people on irc
have problem with. Incidentally, this one is also somewhat
[difficult to search](https://www.google.com/search?q=angularjs+no+module)
for as well. There are many possible reasons why this would happen. Here are
some popular choices (I have made some of these mistakes):

**Instead of having `ng-app="myapp"`, you just have `ng-app` or
`ng-app="something else"`.**

This causes a problem as angular expectes whatever that's fed to `ng-app`
to be a module.

**Solution**: Put `ng-app="myapp"` at the appropriate location.

----------

**You only have `angular.module("myapp")` and never
`angular.module("myapp", [dep1, dep2])`**

As far as I understand, the latter actually creates a new module, the
former only finds an existing module. Of course Angular won't find the
module if you never defined it!

**Solution**: Declare your module with `angular.module("myapp", [...])`.

----------

**You have the `angular.module("myapp", [dep1, dep2])` and
`angular.module("myapp")` in the wrong order.**

You must declare the module first before you can use it. While in most
scenarios angular does not care about the order due to its dependency
injection system, this is one case where the order matters!

**Solution**: Always declare your module before using it!

Quick story: I had a build system that just included js files with random
order. It was okay for my machine as for some reason the file that
declared the module is always included first. However, when a friend of
mine started to work on the project, he kept getting this error. We
eventually discovered that the JS file was included in a different order
on his machine and the file that declared the module was one of the last
ones to load.

\#2: Unknown provider ... or you can't find your controllers/services/whatever
------------------------------------------------------------------------------

There actually could be a lot of reasons why this happens, I probably won't
list all of this. Here is the list that I've encountered:

**You have multiple `angular.module("myapp", [])` statements.**

So what happens here is each time you give the `module` function a second
argument, you actually end up redeclaring (and therefore overwritting)
your module. At this point, you lose all your controller, services, etc.

**Solution**: Make sure you declare your module only once at essentially
the beginning of your JavaScript, right after AngularJS is loaded.
Remember, declaring a module Angular means that you have two arguments to
the function `.module`. For example, you can't spread out your
dependencies over multiple `angular.module` statement like this:

    angular.module("app", ["dep1", "dep2"]);
    angular.module("app", ["dep3", "dep4"]);

Instead, do this

    angular.module("app", ["dep1", "dep2", "dep3", "dep4"]);

----------

**You didn't actually inject your module into your main app module.**

If you have multiple modules, they won't know about each other until you
inject them all into your main app module (or whatever, you get the
point, hopefully :P).

**Solution**: In your `angular.module("myapp", [dep1, dep2])`, make sure
that list includes wherever your module lives.

----------

**You tried to inject `ngResource` or `ngCookies` when you wanted to service.**

That's the module name, not the name of the actual service.

**Solution**: inject `ngResource` and `ngCookies` when declaring your
modules. When you want to use it, inject `$resource` and `$cookies`.

\#3: Getting the controller code to run
---------------------------------------

There are a lot of different issues that I've run into with this and seen
others run into.

**Your controller in the $routeProvider is telling you that it is undefined.**

You're probably using the literal Controller name (so like,
`$r...when({controller: MyController})`). You should be using the string
literal. Angular can't find `MyController` as it is as it say, `undefined`.

**Solution**: Use a string literal so Angular can find the appropriate
controller: `$....when({controller: "MyController"})`

----------

**Your controller is not getting run and you're using partials**

One problem I had is that the partial is actually empty. This means that there
is nothing for the controller to bind to and therefore it will not run.

**Solution**: put a `<div></div>` in your partial.

----------

**Your controller is getting fired twice and you're using partials**

AngularJS implicitly binds to your partial. If your partial has a root element
that also has `ng-controller`, it will be initialized again. Something like
this:

JS file:

    angular.controller("MyController", ...);
    $routeProvider.when("...", {
      controller: "MyController",
      templateUrl: "partial.html"
    });

Partial file

    <div ng-controller="MyController"></div>

**Solution**: remove the `ng-controller` from your html.

\#4: Problem with view renderings and what not
----------------------------------------------

**Your view is not updating, deferred is not resolving/rejecting, basically
that kinda problem where Angular says it will do something but does not**

This is a very broad problem. I always had trouble because I was doing
something outside of the angular world (maybe an onsuccess handler from
IndexedDB, `setTimeout`, or anything else not wrapped by Angular).

**Solution**: You need to read into [$apply](http://docs.angularjs.org/api/ng.$rootScope.Scope).
This means that you need to wrap in your code in `$apply`. Don't overuse this
as it is unnecessary and may cause other issues. Also: use `$timeout` instead
of `setTimeout`.

----------

**$digest cycle already in progress**

This means that you're trying to use `$apply` when Angular is already
`$apply`ing. Usually a sign that your job (I assume you're calling something
async but not via angular) is finishing too fast.

**Solution**: use [$safeApply](https://github.com/mozilla/osumo/blob/master/static/js/develop/app.js#L112).
Some say you can use `$timeout(..., 0);`. I have not tried that personally.

----------

**Your view won't show up when first hit it**

I'll leave you with a link: http://stackoverflow.com/questions/17309488/angularjs-initial-route-controller-not-loaded-subsequent-ones-are-fine

**Solution**: Don't use `$apply` in any initialization code. If you need that,
use `$timeout(..., 0)` instead.
# Recap

A lot has happened since the start of this series. I thought it would be good to take a step back and summarise what’s happened so far. We’re also at the point where we can start looking at individual modules, outside of the cycle of the framework.

## Initial Study

We began by looking at how the framework starts up. Following the flow of content, we can summarise the sections as follows:

### Autoloading

The framework adds four different autoload mechanisms.

- The Composer Autoloader

- The concatenated class file

- The ad-hoc (`Illuminate`) `ClassLoader`

- The Workbench class loader

The Composer autoloader includes everything specified in `composer.json` (including all of the dependencies and their dependencies and so forth). The concatenated class file then reduces the I/O for frequently-loaded classes.

This is followed by the ad-hoc loader, which helps to load anything not picked up by the previous two. Finally, the workbench loader searches for `composer.json` files in the `workbench` directory.

### Application

The framework has a chain of start files, which initialise the `Application` class. This is a subclass of the IoC Container. It has a few initialisation methods which register a number of service providers in the container.

The `Application` class also constructs a chain of middleware classes, which terminate in a call to `Application->handle()`. This goes on to build and send the `Response`.

### Environments

Along the way to building the `Response`, the framework introduces the concept of environments. These help to define which set of configuration settings are used. This allows for differentiation between development/staging/production/testing environments.

The framework also sets a number of paths for things like the public and storage directories.

### Start

The `Application` registers a number of static proxy classes, called Facades. These simplify calls to other classes, by handling the initialisation and instance method dispatching transparently.

`with(new Environment($resolver, $finder, $events))->make()`
becomes `View::make()`, and so forth.

### Configuration

Configuration settings are stored in a few places. First there’s the Plain Ol’ PHP files (in the `app/config` directory). Then there are the `.env.php` and `.env.*.php` files, which populate the `$_ENV` global array.

Configuration also exists to create aliases to frequently-used classes, such as `App` aliases `Illuminate\Support\Facades\App` and so forth.

### Cleaning Up

The framework handles a number of small (yet essential) tasks before sending the `Response`. These include things like enabling HTTP method overrides (so that `POST` + `_method=PUT` is treated like `PUT`), and dispatching post-boot event handlers.

## Further Study

We then continued, by further defining the roles (and composition) of the `Request`, `Router`, `Route` and `Response`…

### Request

The `Request` is based on a symphony component, of the same name. It is tasked with interpreting and formatting the actual request data into something more friendly for the rest of the framework/application.

It introduced us to the concept of `ParameterBag` classes, which Symfony defines to help work with collections of parameters.

The `Request` also demonstrates the Open-Closed Principle, by defining a static property; which defines the name of a factory method. This lets subclasses substitute their own factory methods, the results of which are checked by the original factory method.

### Router

The `Router` is a manger of sorts. It collates the defined routes into a `RouteCollection` and dispatches requests to routes after finding the first matching `Route`.

It also introduces the concept of filters, to the boot process. Filters are sets of functionality which are executed before or after routes are dispatched, and they can alter the response of routes or prevent them from being dispatched in the first place.

### Route

The `Route` is a link between a URL pattern, one or more HTTP methods and a set of functionality (such as a route callback or Controller action).

This can be matched against the current request to select the first applicable route, which is then run. The results are given to a `Response` object…

### Response

The `Response` is based on a Symfony component, of the same name. Its job is to format any return data (from filters, route callbacks, actions, what-have-you) into an acceptable response. For JSON requests (Ajax, or just the right content-type) it will return the correct content type and encode the data appropriately.

It comes in three flavours; Vanilla, Json and Redirect. While `Response` will return JSON data for an explicit request, `JsonResponse` will short-cut the process. `RedirectResponse` just renders a small HTML document which redirects the browser.
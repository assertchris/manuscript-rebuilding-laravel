# Router

The `Router` does all the heavy lifting, when it comes to matching URLs to actions. It is the bridge between the initial application start-up process and the application processing.

## Dispatch

We first encounter the Router class during the boot processes:

{lang=php}
```
public function dispatch(Request $request)
{
  if ($this->isDownForMaintenance())
  {
    $response = $this['events']->until('illuminate.app.down');
    
    if (!is_null($response))
      return $this->prepareResponse($response, $request);
  }
  
  if (
    $this->runningUnitTests() &&
    ! $this['session']->isStarted()
  )
  {
    $this['session']->start();
  }
  
  return $this['router']->dispatch(
    $this->prepareRequest($request)
  );
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

Remember, this was called from:

{lang=php}
```
public function handle(
  SymfonyRequest $request,
  $type = HttpKernelInterface::MASTER_REQUEST,
  $catch = true
)
{
  try
  {
    $this->refreshRequest(
      $request = Request::createFromBase($request)
    );
  
    $this->boot();
  
    return $this->dispatch($request);
  }
  catch (\Exception $e)
  {
    if ($this->runningUnitTests()) throw $e;
  
    return $this['exception']->handleException($e);
  }
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

Dispatching is just another word for “sending” a request to a router, in order for the appropriate actions to be executed. The dispatch method is called, as part of the middleware setup process, and this in turn calls the dispatch method on the `Router` class.

## Filters

This dispatch method looks like this:

{lang=php}
```
public function dispatch(Request $request)
{
  $this->currentRequest = $request;
  
  $response = $this->callFilter('before', $request);
  
  if (is_null($response))
  {
    $response = $this->dispatchToRoute($request);
  }
  
  $response = $this->prepareResponse($request, $response);
  
  $this->callFilter('after', $request, $response);
  
  return $response;
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Routing/Router.php`

We’re introduced to the idea of **filters** — that is; callbacks that are run before and after actions. From this code, we can see that before filters have the power to prevent actions from being executed. If a **before** filter returns anything but null, the method `dispatchToRoute()` will not be run, which means no action.

A> This is further confirmed by the examples in **[http://laravel.com/docs/routing#route-filters](http://laravel.com/docs/routing#route-filters)**.

After an action is run, the response it returns is “prepared” and then passed to the **after** filters. From this code, we can derive a running order:

1. before filters

2. action (if the filters returned null/void)

3. prepare response

4. after filters

The `dispatchToRoute()` method looks like this:

{lang=php}
```
public function dispatchToRoute(Request $request)
{
  $route = $this->findRoute($request);
  
  $this->events->fire(
    'router.matched',
    array($route, $request)
  );
  
  $response = $this->callRouteBefore($route, $request);
  
  if (is_null($response))
  {
    $response = $route->run($request);
  }
  
  $response = $this->prepareResponse($request, $response);
  
  $this->callRouteAfter($route, $request, $response);
  
  return $response;
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Routing/Router.php`

Laravel ships with a few filters, like:

{lang=php}
```
Route::filter('auth', function()
{
  if (Auth::guest()) return Redirect::guest('login');
});
```

A> This is from `app/filters.php`

A> Although a very good example of how filters can be used, this is certainly not the only way. Recently **[Ryan Neilson](https://twitter.com/RyanNielson)** published a tutorial about **[using filters to easily localise](http://nielson.io/2014/05/easy-route-localization-in-laravel-using-filters)**, based on routes.

We’re also seeing quite a few examples of events. Events are callbacks expecting to be fired at key points in the execution cycle of the application. We’ll look at them in more detail, when we dive into the `Event` class.

The dispatch method looks for routes, by passing the request to the `findRoute()` method:

{lang=php}
```
protected function findRoute($request)
{
  $this->current = $route = $this->routes->match($request);
 
  return $this->substituteBindings($route);
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Routing/Router.php`

This in turn calls the `match()` method, on the `RouteCollection` class. The `RouteCollection` class is another of the classes which are similar to `Collection`. This one deals specifically with routes (hence the name, Sherlock!); splitting them by request method and then matching their patterns.

Once a route is found, the `run()` method is called (with the request as an argument). We’ll look at that another time. The `prepareResponse()` method looks like this:

{lang=php}
```
protected function prepareResponse($request, $response)
{
  if ( ! $response instanceof SymfonyResponse)
  {
    $response = new Response($response);
  }
  
  return $response->prepare($request);
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Routing/Router.php`

If the filter(s) or the action returned anything other than a `Symfony\Component\HttpFoundation\Response` instance, it will be wrapped in a response object. Then the `prepare()` method is called on that response.

This means we should study the `Route` and `Response` classes to fully understand the routing process. The rest of the `Router` class has utility methods to do various things, like get the current request or named parameters matched by the route.
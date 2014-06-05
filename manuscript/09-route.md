# Route

The `Route` class works hand-in-hand with the `Router` class. While the `Router` class deals with all possible (defined) URLs, the `Route` class deals with a single URL answering to one or more HTTP Methods.

## New Routes

Routes first get created in the Router:

{lang=php}
```
protected function createRoute($methods, $uri, $action)
{
  if ($this->routingToController($action))
  {
    $action = $this->getControllerAction($action);
  }
  
  $route = $this->newRoute(
    $methods, $uri = $this->prefix($uri), $action
  );
  
  $route->where($this->patterns);
  
  if (count($this->groupStack) > 0)
  {
    $this->mergeController($route);
  }
  
  return $route;
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Routing/Router.php`

Routes may be defined with a proprietary syntax (`controller@method`), which needs to be parsed into something more standard. The `getControllerAction` method calls the `getClassClosure` method. This (finally) splits the `uses` syntax and wraps it in a callback.

## Matching

This leads to the the `Route` being initialised with; one or more defined methods, a URL pattern and a bit of executable code. The idea is that the router will iterate over the defined routes, run a matcher method on each and execute the callback for the first matching route. We see this happening in:

{lang=php}
```
protected function check(array $routes, $request)
{
  return array_first(
    $routes,
    function($key, $value) use ($request)
    {
      return $value->matches($request);
    }
  );
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Routing/RouteCollection.php`

A> Remember that the `Router` class created a `RouteCollection` instance? It did so to maintain the individual `Route` instances, so that a quick match can be done at this very point!

The matches method looks like:

{lang=php}
```
public function matches(Request $request)
{
  $this->compileRoute();
  
  foreach ($this->getValidators() as $validator)
  {
    if ( ! $validator->matches($this, $request)) return false;
  }
  
  return true;
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Routing/Route.php`

## SymfonyRoute

The `compileRoute()` method creates a new `SymfonyRoute` instance, and runs the `compile()` method on it:

{lang=php}
```
protected function compileRoute()
{
  $optionals = $this->extractOptionalParameters();
  
  $uri = preg_replace('/\{(\w+?)\?\}/', '{$1}', $this->uri);
  
  $this->compiled = with(
    new SymfonyRoute(
      $uri,
      $optionals,
      $this->wheres,
      array(),
      $this->domain() ?: ''
    )
  )->compile();
}
```

A> This is from `vendor/symfony/routing/Symfony/Component/Routing/Route.php`

A> The call to `preg_replace()` removes `?` from optional parameters (so that `{foo?}` becomes `{foo}`).

The `SymfonyRoute` (`Symfony\Component\Routing\Route`) class passes its data to the `Symfony\`
`Component\Routing\RouteCompiler` class, which generates a couple regular expressions and compiles input variables so that the route can easily be matched.

The main regular expression generated, is then used in the `UriValidator` class:

{lang=php}
```
public function matches(Route $route, Request $request)
{
  $path = $request->path() == '/' ? '/' : '/'.$request->path();
  
  return preg_match(
    $route->getCompiled()->getRegex(),
    rawurldecode($path)
  );
}
```

A> This is from
A> `vendor/laravel/framework/src/Illuminate/Routing/Matching/`  
A> `UriValidator.php`

## Validators

The `getValidators()` method returns an array of classes tasked with checking certain aspects of the URL, to allow for it to be matched to a route. There are four (so far):

1. HostValidator

2. MethodValidator

3. SchemaValidator

4. UriValidator

They are positive matchers which means their individual `matches()` methods all need to return true for a `Route` to be considered a match. As we’ve just seen; the most important of these is the `UriValidator` class.

### HostValidator

Much like the `UriValidator` class matches the URL path, the HostValidator class matches the host name. In the URL `http://www.example.com/path/to/page`, `www.example.com` is the host name, while `/path/to/page` is the path.

### MethodValidator

This validator checks to see whether the methods this route was defined against contain the request method used to make the request being dispatched.

### SchemeValidator

This validator checks to see whether the route was defined to respond only to SSL or not. If the request being dispatched is SSL based and the route was defined to not respond to SSL based requests, the route will not be matched. The inverse also applies.

## Requirements

The part of compileRoute, which creates the new `SymfonyRequest` instance, passes the property `$this->wheres`. This property is populated with regular expression requirements for named pattern parameters.

These requirements are documented at **[http://laravel.com/docs/routing#route-parameters](http://laravel.com/docs/routing#route-parameters)**, but the basic gist is that you can define “sub-expressions” which are enforced in the matching stage. For example:

{lang=php}
```
Route::get("path/to/page/{fragment}", [...])
  ->where("fragment", "[a-z0-9]");
```

## Running

Once a route is matched, the method which does the final bit of dispatching is:

{lang=php}
```
public function run()
{
  $parameters = array_filter(
    $this->parameters(),
    function($p) { return isset($p); }
  );
  
  return call_user_func_array(
    $this->action['uses'],
    $parameters
  );
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Routing/Route.php`

This method fetches the stored parameters, and url decodes any string values therein. This method is even available for use from within controllers or route callbacks:

{lang=php}
```
$app->make("router")->getCurrentRoute()->parameters();
```

## And The Rest…

…of the `Route` class essentially serves the need for modifying or retrieving route data.
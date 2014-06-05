# Response

The `Response` class acts as the single point of exit for dispatched Laravel requests. Even when route callbacks or actions return plain strings, these are wrapped in a `Response` object. Let’s look at what they do.

## But What Is It Really?

The `Response` class is another class which is built on top of a Symfony class (of the same name). In essence, the `Response` class is given the task of preparing content returning to the browser. This includes things like determining what content-type to respond with and setting cache headers.

It first gets called in:

{lang=php}
```
public function dispatch(Request $request)
{
  if ($this->isDownForMaintenance())
  {
    $response = $this['events']
      ->until('illuminate.app.down');
    
    if ( ! is_null($response))
      return $this->prepareResponse($response, $request);
  }
  
  if ($this->runningUnitTests()
    && ! $this['session']->isStarted())
  {
    $this['session']->start();
  }
  
  return $this['router']
    ->dispatch($this->prepareRequest($request));
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

Ignoring the sections concerned with the application being down for maintenance or running unit tests, the response comes from a call to `Router->dispatch()`.

## Getting The Response

If we remember, back to the start of the framework, the `Illuminate/Foundation/start.php` file makes a call to register the service providers:

{lang=php}
```
$providers = $config['providers'];
 
$app->getProviderRepository()->load($app, $providers);
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

This list of providers includes the `RoutingServiceProvider`, which class binds the router to the application (container):

{lang=php}
```
protected function registerRouter()
{
  $this->app['router'] = $this->app->share(function($app)
  {
    $router = new Router($app['events'], $app);
  
    if ($app['env'] == 'testing')
    {
      $router->disableFilters();
    }
  
    return $router;
  });
} 
```

A> This is from
A> `vendor/laravel/framework/src/Illuminate/Routing/`  
A> `RoutingServiceProvider.php`

Now let’s then remind ourselves how the routes are dispatched:

- `public/index.php` calls `Application->run()`.

- `Application->run()` calls `Application->dispatch()`, via `Application->handle()`.

- `Application->dispatch()` calls `Router->dispatch()`.
`Router->dispatch()` calls `Router->run()`, via `Router->dispatchToRoute()`.

- Results from `Route->run()` are wrapped in an instance of `Symfony\Component\HttpFoundation\`  
`Response`.

- `Application->run()` calls `Response->send()`.

The process of wrapping a route’s response happens in:

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

…which gets called in:

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

This means that anything a route callback (or action) returns will be wrapped in an instance of `Illuminate\Http\Response`, which is a subclass of `Symfony\Component\HttpFoundation\`  
`Response`.

## Response

The constructor for `HttpFoundation\Response` accepts three parameters:

1. content (default: `""`)

2. status (default: `200`)

3. headers (default: `[]`)

That means creating a new `Response` object(from anywhere) is as simple as:

{lang=php}
```
$response = new Response(
  "Hello World", 200,
  ["content-type" => "text/plain"]
);
```

Laravel will automatically render JSON content in a number of situations. If you’ve ever wondered how it knows to do this, it’s because of a setter:

{lang=php}
```
public function setContent($content)
{
  $this->original = $content;
  
  if ($this->shouldBeJson($content))
  {
    $this->headers->set('Content-Type', 'application/json');
  
    $content = $this->morphToJson($content);
  }
  
  elseif ($content instanceof RenderableInterface)
  {
    $content = $content->render();
  }
  
  return parent::setContent($content);
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Http/Response.php`

And the `shouldBeJson()` function:

{lang=php}
```
protected function shouldBeJson($content)
{
  return $content instanceof JsonableInterface ||
    $content instanceof ArrayObject ||
    is_array($content);
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Http/Response.php`

From this, we can conclude that array(-like) structures and classes which implement  
`JsonableInterface` will be encoded. How does this relate to the `JsonResponse` class?

## JsonResponse

`JsonResponse` accepts the same constructor arguments, but includes a fourth: options. These map to the options defined for `json_encode`.

A> You can find these options at `http://www.php.net/manual/en/function.json-encode.php`.

`Illuminate\JsonResponse` subclasses `HttpFoundation\JsonResponse`, which deals specifically in JSON data. `Illuminate\JsonResponse` overrides a `setData` setter (similarly to how  
`Illuminate\Response` overrides the `setContent` setter):

{lang=php}
```
public function setData($data = array())
{
  $this->data = $data instanceof JsonableInterface
    ? $data->toJson($this->jsonOptions)
    : json_encode($data, $this->jsonOptions);
 
  return $this->update();
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Http/JsonResponse.php`

This seems like a bit of duplication, despite small differences in header content-types and encoding strategies.

## RedirectResponse

This is a subclass on top of `HttpFoundation\RedirectResponse`. It is by far the simplest, overriding the `setContent` setter to generate an HTML document which redirects the browser.

Finishing Off
The rest of these three classes/subclasses are all about setting headers and calling platform/situation-specific output methods. For example:

- In a FastCGI environment; the `fastcgi_finish_request()` function is called.

- In a Windows environment; the output buffering level gets special attention due to incorrect depth reporting.

- For each level of output buffering, the `ob_end_flush()` function is called.

- In JSON responses; an `update()` method updates runs (after `setData()`) to update the response to account for JSONP callbacks.

- In JSON responses; special error reporting is added, to account for errors in the encoding process.
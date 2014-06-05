# Cleaning Up

There remains little to do before the application is booted.

## Request Types

Continuing in `vendor/../Foundation/start.php`, we see the following line:

{lang=php}
```
Request::enableHttpMethodParameterOverride();
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

The specifics of this method require a little more time than I want to spend in this article, but the general idea is that it enables the use of hidden form fields to convey the intended request method.

You see, while the HTTP spec allows a number of different types of requests (like `GET`, `POST`, `PUT`, `DELETE` etc.) browsers are notoriously bad at supporting them. Just about the only methods you can expect to see supported, at present, are `GET` and `POST`. For this reason, a common method for simulating the other types has arisen.

With this call, to the `enableHttpMethodParameterOverride()` method, you can design your forms with a hidden form input called _method. This field can contain any of the valid request type names, and the framework will act as though a request of that type was made.

A> You can even make ajax requests using the same request type spoofing.

## More Service Providers

Following this, `vendor/../Foundation/start.php` loads the rest of the service providers:

{lang=php}
```
$providers = $config['providers'];
 
$app->getProviderRepository()->load($app, $providers);
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

Heading over to the Application class file, we see:

{lang=php}
```
public function getProviderRepository()
{
  $manifest = $this['config']['app.manifest'];
 
  return new ProviderRepository(new Filesystem, $manifest);
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

Service Providers come in two flavours — those that are deferred and those that are immediately loaded. This is the purpose of the services manifest file (spoken of in `app/config/app.php`). Let’s see how this is compiled:

{lang=php}
```
public function load(Application $app, array $providers)
{
  $manifest = $this->loadManifest();
  
  if ($this->shouldRecompile($manifest, $providers))
  {
    $manifest = $this->compileManifest($app, $providers);
  }
  
  if ($app->runningInConsole())
  {
    $manifest['eager'] = $manifest['providers'];
  }
  
  foreach ($manifest['when'] as $provider => $events)
  {
    $this->registerLoadEvents($app, $provider, $events);
  }
  
  foreach ($manifest['eager'] as $provider)
  {
    $app->register($this->createProvider($app, $provider));
  }
  
  $app->setDeferredServices($manifest['deferred']);
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/ProviderRepository.php`

The main purpose of this method is to load the manifest file (and sometimes recompile it) in order to determine when each of the service providers should be loaded. The classes marked as `eager` (not the deferred classes) are created and registered.

A> The manifest file contains a copy of the providers array which gets passed, in the call to load(). If the array in the manifests file doesn’t match the array from the configuration file, the manifest is recompiled.
A> 
A> Another thing to note is that deferred providers are not deferred when the application is run from the console.
A>
A> The parts about when are to do with event-registration. We’ll get to that…

If a service provider is set to `defer`, it is only loaded when a class it registers is made (using the `Container`). Each service provider can define a function (called `provides()`) which returns an array of `Container` keys. If one of these keys is needed, it will be linked to the deferred provider (via the manifest file) and the corresponding provider will be registered and booted.

To illustrate this point, assume you have a service provider resembling the following:

{lang=php}
```
class NotifierService
{
  protected $defer = true;
  
  public function register()
  {
    $this->app->bind("notifier", function() {
      return new Notifier();
    });
  }
  
  public function boot()
  {
    // ...do some boot processing
  }
  
  public function provides()
  {
    return ["notifier"];
  }
}
```

A> This service provider will only be registered and booted when the following happens:

{lang=php}
```
$notifier = App::make("notifier");
```

## Booted!

The final bit of this file begins by creating an event callback:

{lang=php}
```
$app->booted(function() use ($app, $env)
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

The code that follows (and is within that callback), will only happen when the rest of the application is booted. We’ll see how that’s determined later.

The remainder of the code, in this file, is dedicated to loading start files. These can be found in `app/start`. The `app/start/global.php` file will always be included, while a file matching the current environment (such as `app/start/local.php` or `app/start/production.php`) will be loaded accordingly.

Finally, should there be an `app/routes.php` file, this too will be loaded.

Heading back to `public/index.php`, we see one more line:

{lang=php}
```
$app->run();
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

We can see that the `Application` class defines the `run()` method as:

{lang=php}
```
public function run(SymfonyRequest $request = null)
{
  $request = $request ?: $this['request'];
  
  $response = with(
    $stack = $this->getStackedClient()
  )->handle($request);
  
  $response->send();
  
  $stack->terminate($request, $response);
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

This method gets the request object and handles it with a thing called a **Stacked Client**. That’s the middleware we saw earlier — a few classes are push onto the stack so that they can affect input and output without modifying the rest of the application code:

{lang=php}
```
protected function getStackedClient()
{
  $sessionReject = $this->bound('session.reject') ? 
    $this['session.reject'] : null;
  
  $client = with(new \Stack\Builder)
    ->push('Illuminate\Cookie\Guard', $this['encrypter'])
    ->push('Illuminate\Cookie\Queue', $this['cookie'])
    ->push(
      'Illuminate\Session\Middleware',
      $this['session'],
      $sessionReject
    );
  
  $this->mergeCustomMiddlewares($client);
  
  return $client->resolve($this);
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

This `Stack\Builder` class acts as a list of modifiers on the results of the `Application` classes `handle()` method. The `handle()` method is responsible for passing the request to the router, so that a route is matched with an action.

A> The specifics of this are troublesome to explain without prior knowledge of the router and the request implementation. We’ll get to know both of these well, but at this point the application is essentially booted!
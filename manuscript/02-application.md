# Application

The second significant line, in `public/index.php` is to load the application. It looks like this:

{lang=php}
```
$app = require_once __DIR__.'/../bootstrap/start.php';
```

A> This is from `public/index.php`

This aspect will take some time to explore! If we hop into that file, we see the first line creates a new `Application` instance:

{lang=php}
```
$app = new Illuminate\Foundation\Application;
```

A> This is from `bootstrap/start.php`

## Container

The `Application` class extends `Illuminate\Container\Container`. This is the first really major concept underpinning the Laravel codebase.

The `Container` class (sometimes referred to as the IoC Container) is an implementation of the `Service Locator` design pattern. The idea is that this one class acts as a registry for dependencies throughout an application. The dependencies (which are just other classes commonly instantiated) are registered so that they can be located and created at runtime.

A> IoC stands for `Inversion of Control`. It’s a reference to the SOLID Dependency Inversion Principle which states that; “high-level code shouldn’t depend on low-level code — both should depend on abstractions”. Instead of classes instantiating their own dependencies (referencing concrete implementations); classes accept dependencies which implement known interfaces. There’s more detail to the principle, which you can find in Taylor’s book: **[Laravel: From Apprentice To Artisan](http://leanpub.com/laravel)**.

To demonstrate an example of this:

{lang=php}
```
$container = new Container();
 
$container->bind("Acme\Class", function() {
  return new Acme\Class();
});
 
$class = $container->make("Acme\Class");
```

As trite an example as that it; it illustrates about 80% of the usage of the IoC container. Laravel is filled with references to `$this->app`, which is this same `Application` instance.

A> You can learn more about the Service Locator design pattern at **<http://en.wikipedia.org/wiki/Service_locator_pattern>**.

## Application

`Application` extends `Container`, but it does more than that. The constructor looks like:

{lang=php}
```
public function __construct(Request $request = null)
{
  $this
    ->registerBaseBindings(
      $request ?: $this->createNewRequest()
    );
  
  $this->registerBaseServiceProviders();
  
  $this->registerBaseMiddlewares();
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

The `Request` defined here is a reference to `Illuminate\Http\Request`. We’ll look at this class, in more detail, in a later article. What you need to know about it now is that it encapsulates the logic behind HTTP request processing and data management. The `registerBaseBindings()` method looks like:

{lang=php}
```
protected function registerBaseBindings($request)
{
  $this->instance('request', $request);
 
  $this->instance('Illuminate\Container\Container', $this);
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

This method stores a reference to the `Request` instance, in the `Container` storage. It uses the `instance()` method which differs slightly from `bind()` in that bind will create new instances when `make()` is called, while `instance()` and `make()` will return the same instance every time.

A> The behavior of the `instance()` method allows Laravel to restrict these classes to having a single instance, without building them according to the Singleton design pattern. You can learn more about the Singleton design pattern at **<http://en.wikipedia.org/wiki/Singleton_pattern>**.

This method also stores a reference to itself in the `Container` storage. That is to say — when someone requests the class `Illuminate\Container\Container` from the container, they will get the shared instance of `Illuminate\Foundation\Application`. It’s interesting that it’s referred to by the full class path, but the reasons for that will become obvious when we look at the `Container` in greater depth.

## Service Providers

The `registerBaseServiceProviders()` method looks like:

{lang=php}
```
protected function registerBaseServiceProviders()
{
  foreach (array('Event', 'Exception', 'Routing') as $name)
  {
    $this->{"register{$name}Provider"}();
  }
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

Aside from demonstrating a badass use of dynamic method names in PHP, this method loops through an array of essential class names, and invokes methods to register the service provider for each.

We’ll just look at one of these (because they’re all the same):

{lang=php}
```
protected function registerEventProvider()
{
  $this->register(new EventServiceProvider($this));
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

All Service Providers are instantiated with a reference to the `Application` instance as the first constructor argument. This lets them pull dependencies from the `Container` storage, without needing to know the `Application` class’s full class path (and thereby be tightly coupled to it).

A> Service Providers are an organisational model for isolating components of functionality which can then be registered and run in a consistent manner. They have `register()` and `boot()` methods to this effect. Everything from databases to queues are registered through Service Providers. We’ll look as Service Providers in greater depth, in future articles.

So from this, we can see that the core Laravel application needs a request, events, exceptions and routing to be in place before dispatching to an action.

The last method called in the constructor is `registerBaseMiddlewares()`, which looks like:

{lang=php}
```
protected function registerBaseMiddlewares()
{
  $this->middleware('Illuminate\Http\FrameGuard');
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

A very simplified description of middleware (in the context of PHP frameworks) is that they are reusable libraries which implement a commonly agreed-upon design. This design allows these libraries to be used in many frameworks for the purposes of giving new input and shaping the output without actually affecting codebases.

A> The middleware pattern implemented, in the libraries Laravel uses, is based on **[StackPHP](http://stackphp.com)**.

The **FrameGuard** middleware looks like this:

{lang=php}
```
public function handle(
  SymfonyRequest $request,
  $type = HttpKernelInterface::MASTER_REQUEST,
  $catch = true
)
{
  $response = $this->app->handle(
    $request,
    $type,
    $catch
  );
 
  $response->headers->set(
    'X-Frame-Options',
    'SAMEORIGIN',
    false
  );
  
  return $response;
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Http/FrameGuard.php`

This method gets the response from Laravel (the entire response after dispatching to the action) and adds a header to it. The `SymfonyRequest` referenced here is `Symfony\Component\
HttpFoundation\Request`. `HttpKernelInterface` also comes from Symfony…
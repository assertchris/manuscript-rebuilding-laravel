# Sessions

Sessions are a bread-and-butter component of many websites. Anything that needs a login, collects items in a shopping-cart or remembers activity/personalisation probably uses a sessions.

## Architecture

Before we look at exactly how sessions work, let's examine the architecture of Laravel sessions. The Session component has a service provider (and facade), which is loaded in config:

{lang=php}
```
'providers' => array(
  ...
  'Illuminate\Session\SessionServiceProvider',
  ...
),
```

A> This is from `app/config/app.php`

There are also session-specific details in `app/config/session.php`. The service provider registers a `SessionManager` instance, with the IoC container:

{lang=php}
```
protected function registerSessionManager()
{
  $this->app->bindShared('session', function($app)
  {
    return new SessionManager($app);
  });
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Session/SessionServiceProvider.php`

This `session` binding is the same one the `Session` facade uses. This means the class you're invoking methods on (when you're using the `Session` facade) is really the `SessionManager` class.

`SessionManager` extends the `Illuminate\Support\Manager` class, so after registering a few session drivers, any calls are redirected through to the specific driver (defined in config):

{lang=php}
```
public function __call($method, $parameters)
{
  return call_user_func_array(
    array($this->driver(), $method),
    $parameters
  );
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Support/Manager.php`

## Manager

This `Manager` class is actually quite cool. It is designed around the idea that a component can have multiple drivers, and these can be handled with minimal effort. That is; `Manager` defines methods around the idea of driver creators.

A component can define methods for each driver it supports, and the correct method will be invoked based on the name of the driver being created. More of these can be added at runtime:

{lang=php}
```
public function extend($driver, Closure $callback)
{
  $this->customCreators[$driver] = $callback;
  
  return $this;
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Support/Manager.php`

Many of the components in Laravel have managers based on this class, so it will become familiar to you as we look at more of the framework.

## Store

When a session driver is finally created, it is wrapped within an instance of `Illuminate\Session\Store`:

{lang=php}
```
protected function buildSession($handler)
{
  return new Store(
    $this->app['config']['session.cookie'],
    $handler
  );
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Support/Manager.php`

The `Store` class is a wrapper for common session functionality, such as regenerating session identifiers and reading/writing session data.

## Middleware

Sessions are registered, for the application, as part of the middleware stack:

{lang=php}
```
public function handle(
  Request $request,
  $type = HttpKernelInterface::MASTER_REQUEST,
  $catch = true
)
{
  $this->checkRequestForArraySessions($request);
  
  if ($this->sessionConfigured())
  {
    $session = $this->startSession($request);
  
    $request->setSession($session);
  }
  
  $response = $this->app->handle($request, $type, $catch);
  
  if ($this->sessionConfigured())
  {
    $this->closeSession($session);
  
    $this->addCookieToResponse($response, $session);
  }
  
  return $response;
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Session/Middleware.php`

This middleware is registered as part of the `Application` boot process:

{lang=php}
```
protected function getStackedClient()
{
  $sessionReject = $this->bound('session.reject')
    ? $this['session.reject'] : null;
  
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

As a result, the session is created and available without any obvious registration/closure. The session key/value pairs are available though the $attributes property of the session driver in use.

## Multiple Connections

Unlike the Database and Cache components; it doesn't appear to be possible to use multiple session providers. This is probably because sessions are registered through middleware. It is possible to create a session (using a single connection) and store references to multiple cache providers. This is idea for situations where you might need to use multiple stores for session-specific data.

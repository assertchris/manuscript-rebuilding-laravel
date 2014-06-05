# Request

The `Request` class is an essential part of the Laravel 4 system. In terms of web applications, handling requests are what separates impotent applications from interactive ones. `Request` is all about handling the methods and parameters passed from the browser.

## But What Is It Really?

The `Request` class is built on top of Symphony components, and the first few lines show us that:

{lang=php}
```
<?php namespace Illuminate\Http;
 
use Symfony\Component\HttpFoundation\ParameterBag;
use Symfony\Component\HttpFoundation\Request as SymfonyRequest;
 
class Request extends SymfonyRequest {
```

A> This is from `vendor/laravel/framework/src/Illuminate/Http/Request.php`

So to truly understand the `Request` class, we need to understand the underlying (Symfony) `Request` class.

## ParameterBag

The code above references a class called `ParameterBag`. This class is functionally similar to the `Collection` class in that it provides a Facade to some array manipulation functionality.

A> When I use (capital “F”) Facade; I’m referring to the design pattern.

The `ParameterBag` class has a bit of extra functionality for filtering and getting different data type representations of the values stored, but that’s because it is focussed on HTTP parameter collections. The `Collection` class has a more general focus.

A> We’ll look at the `Collection` class in-depth, later on in the series. All you need to know is that it is similar to the `ParameterBag` class because it deals with arrays.

## HttpFoundation\Request

The first few public properties in this class are annotated to hold instances of other classes (in the same package). These classes are all instances of (or subclass instances of) the `ParameterBag` class.

They are meant to hold wrapped and filtered copies of the global parameter objects (`$_GET`, `$_POST`, `$_SERVER`, `$_FILES` etc.). This is important because it tells us two things. The first is that we can expect a common interface when working with the various different types of parameter data. The second is that we should use these filtered copies instead of the globals, because they are safer.

A> In practice, this means you should be using things like `Input::get()` instead of `$_GET[]`.

The constructor looks similar to:

{lang=php}
```
public function __construct(
  array $query = array(),
  array $request = array(),
  array $attributes = array(),
  array $cookies = array(),
  array $files = array(),
  array $server = array(),
  $content = null
)
{
  $this->initialize(
    $query,
    $request,
    $attributes,
    $cookies,
    $files,
    $server,
    $content
  );
}
```

A> This is from `vendor/symfony/http-foundation/Symfony/Component/HttpFoundation/Request.php`

As you can see, the class is intended to be initialised with all of the request data. You may not see much point to the constructor [only] calling another method with all the same parameters, but this makes subclassing `Request` much easier. Overriding this method is as simple as remembering to call `parent::initialize()`.

The `initialize()` method follows, though I won’t repeat it here, as it just sets a ton of properties. What you should notice is that it instantiates the `ParameterBag` properties.

## Creating New Requests

Let’s recap how Laravel uses this class:

{lang=php}
```
protected function createNewRequest()
{
  return forward_static_call(
    array(static::$requestClass, 'createFromGlobals')
  );
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

This means we should be looking at the `createFromGlobals()` method, on the base `Request` class, if we want to understand exactly how this class is used.

{lang=php}
```
public static function createFromGlobals()
{
  $request = self::createRequestFromFactory(
    $_GET,
    $_POST,
    array(),
    $_COOKIE,
    $_FILES,
    $_SERVER
  );
  
  if (
    0 === strpos(
      $request->headers->get('CONTENT_TYPE'),
      'application/x-www-form-urlencoded'
    ) && in_array(
      strtoupper($request->server->get(
        'REQUEST_METHOD',
        'GET'
      )),
      array('PUT', 'DELETE', 'PATCH')
    )
  ) {
    parse_str($request->getContent(), $data);
    $request->request = new ParameterBag($data);
  }
  
  return $request;
}
```

A> This is from `vendor/symfony/http-foundation/Symfony/Component/HttpFoundation/Request.php`

Heading over to the `createRequestFromFactory()` method, we see:

{lang=php}
```
private static function createRequestFromFactory(
  array $query = array(),
  array $request = array(),
  array $attributes = array(),
  array $cookies = array(),
  array $files = array(),
  array $server = array(),
  $content = null
)
{
  if (self::$requestFactory) {
    $request = call_user_func(
      self::$requestFactory,
      $query,
      $request,
      $attributes,
      $cookies,
      $files,
      $server,
      $content
    );
  
    if (!$request instanceof Request) {
      throw new \LogicException(
        'The Request factory must return an instance
         of Symfony\Component\HttpFoundation\Request.'
      );
    }
  
    return $request;
  }
  
  return new static(
    $query,
    $request,
    $attributes,
    $cookies,
    $files,
    $server,
    $content
  );
}
```

A> This is from `vendor/symfony/http-foundation/Symfony/Component/HttpFoundation/Request.php`

This method allows a consumer (or even a subclass) to define a method for creating new requests. If the `$requestFactory` property is set, it will be used. Should it return anything other than an instance of the `Request` class (or subclass thereof) an exception will be thrown.

A> This is a fantastic example of adherence to the Open-Closed Principle. The class is protecting itself against faulty factory methods, but allowing their use from anywhere.

Should no factory method be specified, a new instance of this same class will be created.

## And Then…

Laravel adds a number of helper methods on top of this class. There are methods like `isAjax()` and `wantsJson()` which help to tailor requests without the slog-work of inspecting the headers to find these things out.

There are methods like `segment()` and `secure()` which help to read the URL without using a bunch of string-manipulation methods yourself.

At the end of the day, anything you can do with the underlying Symfony `Request` class is available to you. With the added benefit of some handy shortcut methods, and a consistent namespace (`Illuminate\Http`).
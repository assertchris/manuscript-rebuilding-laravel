# Start

Following the storing of paths in the `Container`/`Application` instance, we see a bit of logic to work out where the application bootstrapping file is. Another bootstrapping file. One from the same folder as the `Application.php` class file:

{lang=php}
```
$framework = $app['path.base'].
  '/vendor/laravel/framework/src';
 
require $framework.'/Illuminate/Foundation/start.php';
```

A> This comes from bootstrap/start.php

This seems a little redundant considering the `getBoostrapFile()` method, defined on the `Application` class:

{lang=php}
```
public static function getBootstrapFile()
{
  return __DIR__.'/start.php';
} 
```

A> This comes from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

## Mcrypt

Still, this file is mostly procedural code (needed to get the application running). Let’s take a look:

{lang=php}
```
if ( ! extension_loaded('mcrypt'))
{
  echo 'Mcrypt PHP extension required.'.PHP_EOL;
 
  exit(1);
}
```

A> This comes from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

Aside from turning default error reporting off, this new bootstrap file ensures that the **Mcrypt** PHP module is loaded. The **Mcrypt** module helps Laravel to hash and encrypt various data points throughout the system. These kinds of libraries take loads of work to make well, and Laravel uses encryption and hashing a lot.

A> Some of the ways Laravel uses encryption are with cookies/sessions and queue messages. Some of the ways Laravel uses hashes are with password salting and authentication remember tokens. We’ll look at all of these later!

## Facade Testing

Something strange happens next…

{lang=php}
```
$app->instance('app', $app);
```

A> This comes from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

The comment explains this as needed to facade test the application. I couldn’t find a case of this being done in the **[Laravel test suite](https://github.com/laravel/framework/tree/master/tests/Foundation)**, but the idea is sound.

Assuming `Foo` was implemented like this…

{lang=php}
```
class Foo
{
  public function getBar()
  {
    return App::make("bar");
  }
}
```

…you could effectively test that the call to the `App::make()` method was made, and that the value was returned:

{lang=php}
```
$app = Mockery::mock("stdClass");
$app->shouldReceive("make")->andReturn("mocked make");
 
App::swap($app);
 
$foo = new Foo();
 
$this->assertEquals("mocked make", $foo->getBar());
```

A> The usefulness of such a test is debatable. The example demonstrates that it can be done, not that it should be.

Following that, the following code can be found:

{lang=php}
```
if (isset($unitTesting))
{
 $app['env'] = $env = $testEnvironment;
}
```

A> This comes from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

This is completely out of context unless we also look at the detail `TestCase` class, which ships with Laravel:

{lang=php}
```
public function createApplication()
{
  $unitTesting = true;
  
  $testEnvironment = 'testing';
  
  return require __DIR__.'/../../bootstrap/start.php';
}
```

A> This comes from `app/tests/TestCase.php`

The code (in `vendor/…/start.php`) is entirely for the benefit the `TestCase` class. It is there to set the environment of the application to `testing` so that the configuration files in `app/config/testing` will be used in addition to the global configuration files.

## Facades

The code, after that, is about facades:

{lang=php}
```
Facade::clearResolvedInstances();
 
Facade::setFacadeApplication($app);
```

A> This comes from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

Facade, in Laravel, is a term used to refer to static-like classes which proxy to regular classes. We’ve already seen one of these in the form of `App`. The `App` facade invokes methods on an instance of `Illuminate\Foundation\Application`. It’s just shorter to type `App` than it is to toe the fully qualified class name.

A> The use of the word Facade is a highly contentious issue. I have no desire to talk about it. If you know the history of the discussion, please refrain from brining it up. If you don’t, you’re probably better off…

The methods look like this:

{lang=php}
```
public static function clearResolvedInstance($name)
{
  unset(static::$resolvedInstance[$name]);
}

public static function setFacadeApplication($app)
{
  static::$app = $app;
}
```

A> These are from `vendor/laravel/framework/src/Illuminate/Support/Facade/Facade.php`

The `Facade` class is the base class for all of the facades which ship with Laravel. It has a protected, static array of facade instances matched to keys of the `Container`.

A> Registering a facade is a two-step process. A service provider will add a resolver (either via the `bind()` method or the `instance()` method). A `Facade` class will then define a method which pulls a class instance out of the `Container`. We’ll see how this is done later.

The first method clears this array of resolved instances out. The second sets a reference to the `Application` on the `Facade` class…
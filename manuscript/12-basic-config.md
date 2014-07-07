# Basic Config

I thought a good place to continue from would be the config package. Laravel manages configuration variables in a few ways, but the most prevalent is through this package.

## How Do These Work?

Most of the configuration settings are defined, in a number of `*.php` files, in `app/config`. They have a general structure resembling:

{lang=php}
```
<?php
  
return array(
	'paths' => array(__DIR__.'/../views'),
	'pagination' => 'pagination::slider-3',
);
```

A> This file should be saved as `app/config/view.php`

This config file contains a couple of view-specific configuration settings. In fact, all of the config files that ship with Laravel are in this format. 

The code which employs them loads the arrays (via the PHP `include` construct) into the local scope. Before we get to the code which does that, let's look at how config is constructed and defined.

## File-based Configuration

Recall that the `public/index.php` file loads a series of bootstrapping files, which result in an `Application` instance being created. In `vendor/.../Foundation/start.php` we see the following code:

{lang=php}
```
$app->instance('config', $config = new Config(
	$app->getConfigLoader(), $env
));
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

Config is an instance of `Illuminate\Config\Repository`. That `getConfigLoader()` function does the following:

{lang=php}
```
public function getConfigLoader()
{
  return new FileLoader(
    new Filesystem,
    $this['path'].'/config'
  );
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

The specifics of the `Filesystem` class are a topic for another time, but you can just think of it as an Object-oriented abstraction of the built-in Posix filesystem functions/objects.

A> We'll study the Filesystem classes soon!

You may also recall that the `Application` is an instance of the `Container`, and that the bootstrapping files stored a number of filesystem paths to the common `Application` instance. In this case; `path` is an alias to `app/`, which means this Filesystem instance is working from a root of `app/config`.

The main functionality behind the `FileLoader` class is in the `load()` method:

{lang=php}
```
public function load($environment, $group, $namespace = null)
{
  $items = array();
  
  $path = $this->getPath($namespace);
  
  if (is_null($path))
  {
    return $items;
  }
  
  $file = "{$path}/{$group}.php";
  
  if ($this->files->exists($file))
  {
    $items = $this->files->getRequire($file);
  }
  
  $file = "{$path}/{$environment}/{$group}.php";
  
  if ($this->files->exists($file))
  {
    $items = $this->mergeEnvironment($items, $file);
  }
  
  return $items;
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Config/FileLoader.php`

The `getPath()` method returns the root config path (app/config), without a namespace. Namespaces are just a way for Laravel to be able to support loading config settings which can be grouped with a common prefix. The most common namespace you'll see is `package::`, which finds config settings in vendor/workshop packages.

If the provided namespace doesn't match an existing directory, an empty array is returned. For valid directories, a filename pattern is constructed. That file (if it exists) is merged with the already loaded config data. 

## Getting Config Settings

The `Iluminate\Config\Repository` class extends `Illuminate\Support\NamespacedItemResolver`, which provides methods to strip the namespace (e.g. `package::`) from the cache key, and then calls the `FileLoader::load()` method.

This means you can access the application config settings as easily as:

{lang=php}
```
Config::get("view.paths");
// where Config is an alias to Illuminate\Support\Facades\Config
```

...and where package config settings can be accessed as easily as:

{lang=php}
```
Config::get("foo::bar.baz");
// where "foo" is the namespace of a custom package
```



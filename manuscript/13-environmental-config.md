# Environmental Config

Last time we looked at how basic configuration works, and this is further augmented by the idea of environmental configuration. That is, depending on the environment; different sets of configuration files are used.

## Trip Down Memory Lane…

We touched on this idea, last time. There’s a specific spot in which we saw the application of environmental configuration:

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

This part of the `FileLoader` class checks for the existence of a set of environment-specific configuration files and loads the appropriate one. These tend to be located as subfolders of `app/config`, so you may already have seen folders like `app/config/production` and `app/config/testing`.

## Specifying Environments

Environments are toggled in one of the start files:

{lang=php}
```
$env = $app->detectEnvironment(array(
  'local' => array('*.local'),
));
```

A> This is from `bootstrap/start.php`

{lang=php}
```
public function detectEnvironment($envs)
{
  $args = isset($_SERVER['argv']) ? $_SERVER['argv'] : null;
   
  return $this['env'] = with(
    new EnvironmentDetector()
  )->detect($envs, $args);
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

The `EnvironmentDetector` class handles instances created through console commands and web requests. In the case of console commands; an env option can be used to set the environment under which the command is run.

In the case of web requests, this is determined by looking at the array of environments in `bootstrap/start.php`. The keys of this array are the name of the environment you want. So `"production" => [...]` and `"testing" => [...]` relate to `app/config/production/*` and `app/config/testing/*` respectively.

The values of this array are used to find a match. You could give an array of hostnames. You could also give a closure, which returns true or false depending on criteria you decide.

The array used to be for domain names, but that was later changed to use hostnames as a matter of security. You can find this name by typing hostname in the terminal.
It’s common practise to supply a closure which returns based on an environment variable:

{lang=php}
```
$env = $app->detectEnvironment(function() {
  if (isset(getenv("environment")) {
    return getenv("environment");
  }
  
  return "production";
});
```

A> This is from `bootstrap/start.php`

Laravel also supports the use of `.env.*.php` files, which return one-dimensional arrays of configuration variables. These are accessed through the `getenv()` function and the `$_ENV[...] ` array.

A> These files are useful for keeping configuration outside of the usual files, but we’ll not go into more detail than that. You can learn more about them at **[http://laravel.com/docs/configuration#](http://laravel.com/docs/configuration#protecting-sensitive-configuration.
Merging Configuration)**
A> **[protecting-sensitive-configuration.
Merging Configuration](http://laravel.com/docs/configuration#protecting-sensitive-configuration.
Merging Configuration)**.

Configurations are merged somewhat recursively, from specific to general. If there is no applicable environmental configuration, the general configuration will be used. Stated differently, if `app/config/production/cache.php` is missing, `app/config/cache.php` will be entirely used. If a single key (inside `app/config/production/cache.php` is changed) that key will override the general config.

A> I say "somewhat" because arrays of items will not be combined, but rather overridden. If you define just one new app.providers property, all of the others will be erased. For a full, recursive merge you’ll need to use the `append_config()` helper function: **[http://laravel.com/docs/configuration#](http://laravel.com/docs/configuration#environment-configuration)**
A> **[environment-configuration](http://laravel.com/docs/configuration#environment-configuration)**.

## Testing

The only exception to how this naturally works, is testing:

{lang=php}
```
public function createApplication()
{
  $unitTesting = true;
  $testEnvironment = 'testing';
  return require __DIR__.'/../../bootstrap/start.php';
}
```

A> This is from `app/tests/TestCase.php`

This code (particularly `$unitTesting` and `$testEnvironment`) are applied just while the application is booting:

{lang=php}
```
if (isset($unitTesting))
{
  $app['env'] = $env = $testEnvironment;
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

This means the environment will be overridden based on the presence and values of variables `$unitTesting` and `$testEnvironment`.
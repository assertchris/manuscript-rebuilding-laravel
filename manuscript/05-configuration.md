# Configuration

Environment configuration can easily pollute application and system code. Fortunately, Laravel keeps this nicely separated.

## Sensitive Configuration

The next couple of lines, in the `vendor/…/Foundation/start.php` file, isolates some of this configuration data:

{lang=php}
```
with($envVariables = new EnvironmentVariables(
  $app->getEnvironmentVariablesLoader()))->load($env);
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

The comment preceding this explains that this is to keep the `$_ENV` and `$_SERVER` variables away from the rest of the application code. Looking back insider the `Application` class, we see the following method:

{lang=php}
```
public function getEnvironmentVariablesLoader()
{
  return new FileEnvironmentVariablesLoader(
    new Filesystem,
    $this['path.base']
  );
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

This is the low-level code used to read `.env.php` files. These allow sensitive configuration variables to be excluded from the app/config files. The `load()` method looks like this:

{lang=php}
```
public function load($environment = null)
{
  foreach ($this->loader->load($environment) as $key => $value)
  {
    $_ENV[$key] = $value;
  
    $_SERVER[$key] = $value;
    
    putenv("{$key}={$value}");
  }
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Configuration/EnvironmentVariables.php`

This means that the application environment variable is used to determine the configuration variables loaded. For instance, if the environment is set to local, the file .env.local.php is loaded.

A> If the environment is set to `production` the `FileEnvironmentVariablesLoader` class will attempt to load `.env.php` instead. We’ll look at how this happens, when we dissect the `illuminate/config` package.

These configuration files define arrays of keys and values. Multi-dimensional arrays are not supported, but that’s reasonable when you consider that neither `$_ENV` nor `$_SERVER` support multi-dimensional arrays either.

A> The `putenv()` call allows for the same variables to be pulled out with calls to the `getenv()` function.

A> You can learn more about using these kinds of configuration files at **[http://laravel.com/docs/configuration#protecting-sensitive-configuration](http://laravel.com/docs/configuration#protecting-sensitive-configuration)**.

## Plain Old Configuration

Back in `vendor/../Foundation/start.php`, we see more configuration happening, with:

{lang=php}
```
$app->instance('config', $config = new Config(
 
 $app->getConfigLoader(), $env
 
));
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

This is to facilitate the loading of the regular configuration files (those in `app/config`) with a loader provided by the `Application` instance. That happens to be:

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

This code is very similar to that used to load the .env.php files. Part of me thinks that could be abstracted in a useful way, so that the code isn’t repeated across packages…

A> You can learn more about using configuration files at **[http://laravel.com/docs/configuration#environment-configuration](http://laravel.com/docs/configuration#environment-configuration)**.

## Exception Handling

The next few lines set the exception handling and error reporting:

{lang=php}
```
$app->startExceptionHandling();
 
if ($env != 'testing') ini_set('display_errors', 'Off');
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

We see this method, over in the Application class:

{lang=php}
```
public function startExceptionHandling()
{
  $this['exception']->register($this->environment());
  
  $this['exception']->setDebug($this['config']['app.debug']);
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

The `register()` method registers exception, error and shutdown handler functions (though the last one is excluded for the `testing` environment). The `setDebug()` method helps the exception class determine which kind of error messages should be displayed.

To illustrate this; when `debug` is set to true (in `app/config/app.php`), comprehensive debugging information is displayed. If `debug` is set to false, then the generic _“Whoops, looks like something went wrong.”_ message is displayed.

A> You should never have debug set to true, in the production environment. It leaks sensitive system information, and can lead to all sort of security nonsense.

We’ll review the specifics of the exception handler in another article.

A> You can learn more about handling errors at **[http://laravel.com/docs/errors#handling-errors](http://laravel.com/docs/errors#handling-errors)**.

## Timezone

Following the exception handling, the timezone is configured:

{lang=php}
```
$config = $app['config']['app'];
 
date_default_timezone_set($config['timezone']);
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

The timezone affects the difference between the time on your computer/in your location and that of the server. It’s not usually a huge problem, but **PHP 5.3+** will display a warning if you aren’t using this method or setting date.timezone in your `php.ini` configuration file.

A> Laravel would naturally suppress these errors, based on environment and configuration, but you don’t really want them in your log files either.

## Class Aliases

The next couple lines of code do something magical:

{lang=php}
```
$aliases = $config['aliases'];
 
AliasLoader::getInstance($aliases)->register();
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/start.php`

The `aliases` key referred to here is that in `app/config/app.php`. It basically uses the built-in `class_alias()` function:

{lang=php}
```
public function load($alias)
{
  if (isset($this->aliases[$alias]))
  {
    return class_alias($this->aliases[$alias], $alias);
  }
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/AliasLoader.php`

I spent a great deal of time trying to understand why some of the things in this class were done. Here are my thoughts:

1. The `AliasLoader` class is built as a Singleton class, but it is used only after the `Container` is initialised. I can only assume this is to avoid overhead when aliases are used, especially when considering…

2. The `AliasLoader` class prepends itself as an `spl_class_autoloader`. This is because the `class_alias()` function will attempt to load the class being aliased before applying the alias. For example; if we want to alias `App` to `Illuminate\Support\Facades\App`, the `class_alias()` method will first try to load the full class path before making the alias. Deferring the the call to `class_alias()` until the point at which the class first required avoids this premature I/O work.

The program flow then becomes:

1. An alias is used with `App::make()`.

2. `AliasLoader`, having prepended itself to the list of autoloaders, runs `load()`.

3. This uses `class_alias()` to link `App` to `Illuminate\Support\
Facades\App`.

4. The full class path is then loaded, using one of the remaining class autoloaders.

Interesting stuff!
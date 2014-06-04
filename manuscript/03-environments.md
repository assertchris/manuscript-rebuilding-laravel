# Environments

After initialising the Application instance, `bootstrap/start.php` invokes a method (on `$app`) called `detectEnvironment()`:

{lang=php}
```
$env = $app->detectEnvironment(array(
  'local' => array('your-machine-name'),
));
```

A> This is from `bootstrap/start.php`

You see, Laravel supports a thing called environments. Environments are triggers to load different configuration and execution parameters, based on the classification of server the application is running on.

A> Environments are Laravel’s way of dealing with changes in servers, so that you can classify certain servers as local development environments, or staging environments or production environments etc.

## Detecting The Environment

The `detectEnvironment()` method looks like:

{lang=php}
```
public function detectEnvironment($envs)
{
  $args = isset($_SERVER['argv']) ? $_SERVER['argv'] : null;
  
  return $this['env'] = with(new EnvironmentDetector())
    ->detect($envs, $args);
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

`$argv` is an associative array of arguments given to the script as it is run from the CLI (Command Line Interface). This array will be set when running things like Artisan (the Laravel command line utility), but it will not be set if the script is being executed as part of the web server request cycle.

A> The `with()` function is a compatibility function which returns whatever is passed to it so that (in the case of `new EnvironmentDetector()`) method calls can be chained. It isn’t needed beyond PHP 5.3 but it doesn’t cause any harm in modern versions.

So let’s take a look at the `EnvironmentDetector` class. It doesn’t have a constructor, but the `detect()` method looks like this:

{lang=php}
```
public function detect($environments, $consoleArgs = null)
{
  if ($consoleArgs)
  {
    return $this->detectConsoleEnvironment(
      $environments,
      $consoleArgs
    );
  }
  else
  {
    return $this->detectWebEnvironment(
      $environments
    );
  }
}
```

A> This is from
A> `vendor/laravel/framework/src/Illuminate/Foundation/EnvironmentDetector.php`

Here we see the effects of the `$argv` array not being set. If the script is being run on the command line, the method that gets executed is:

{lang=php}
```
protected function detectConsoleEnvironment(
  $environments,
  array $args
)
{
  if (!is_null($value = $this->getEnvironmentArgument($args)))
  {
    return head(array_slice(explode('=', $value), 1));
  }
  else
  {
    return $this->detectWebEnvironment($environments);
  }
}
```

A> This is from
A> `vendor/laravel/framework/src/Illuminate/Foundation/EnvironmentDetector.php`

This specifically checks for the `env` command-line argument. If it is not provided, `EnvironmentDetector` defaults to the method used to determine the web environment…

{lang=php}
```
protected function detectWebEnvironment($environments)
{
  if ($environments instanceof Closure)
  {
    return call_user_func($environments);
  }
  
  foreach ($environments as $environment => $hosts)
  {
    foreach ((array) $hosts as $host)
    {
      if ($this->isMachine($host)) return $environment;
    }
  }
  
  return 'production';
}
```

A> This is from
A> `vendor/laravel/framework/src/Illuminate/Foundation/EnvironmentDetector.php`

Looking ahead to the end of the method; we see that the default environment (if no other environment is matched) is `production`.

This can be both good and bad. Good because error reporting is turned off in production (at least by default) and bad because running things like migrations and seeders, when you’ve forgotten to override the environment, can have disastrous effects on your production database.

A> As a rule of thumb — always declare your environment. This, combined with the odd environment check (before doing something destructive in your codebase) will save you a ton of problems.

This calculated environment string is stored back in in the `Container` storage which means it will be available by calling `App::make("env")` or `$this->app->make("env")`, if you are in a service provider. There’s also a shortcut method called `App::environment()` which returns the current environment, or returns a boolean if the current environment is within a list of arguments to the method, as in the following example:

{lang=php}
```
$isSafeForAction = App::environment("local", "staging");
```

A> You can learn more about environments at **[http://laravel.com/docs/configuration#
environment-configuration](http://laravel.com/docs/configuration#environment-configuration)**.

## Setting Paths

The next thing we see, in `bootstrap/start.php`, is a call to `bindInstallPaths()`. This method looks like:

{lang=php}
```
public function bindInstallPaths(array $paths)
{
  $this->instance('path', realpath($paths['app']));
  
  foreach (array_except($paths, array('app')) as $key => $value)
  {
    $this->instance("path.{$key}", realpath($value));
  }
}
```

A> This is from `vendor/laravel/framework/src/Illuminate/Foundation/Application.php`

This method is given the contents of `bootstrap/paths.php`, an array of file paths, and stores each one in the `Container` storage. This results in each being available through calls to `App::make()`, according to the following list:

{lang=php}
```
"app"     - App::make("path")
"public"  - App::make("path.public")
"base"    - App::make("path.base")
"storage" - App::make("path.storage")
```

A> The class `App` is made available through a series of class aliases. We’ll look into this another time. For now it is enough to know that `App` is a reference to the single `Container`/`Application` instance, which is where most other classes should be resolved out of.
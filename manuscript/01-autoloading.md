# Autoloading

I thought the best place for us to start would be where requests start — that is `public/index.php`…

## Rewrite Rules

We know that’s where requests first come into contact with our application, because Laravel ships with a `public/.htaccess` file which offers some `mod_rewrite` logic to make that happen:

```
<IfModule mod_rewrite.c>
  <IfModule mod_negotiation.c>
    Options -MultiViews
  </IfModule>
  
  RewriteEngine On
  
  # Redirect Trailing Slashes...
  RewriteRule ^(.*)/$ /$1 [L,R=301]
  
  # Handle Front Controller...
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteRule ^ index.php [L]
</IfModule>
```

A> This is from `public/.htaccess`

The lines that interest us are `RewriteRule` and `RewriteCond`. The first `RewriteRule` line tells Apache that URLs with trailing slashes should be redirected to the same URL without the trailing slash.

The `RewriteCond` lines tell Apache to ignore further rewrite rules for URLs that point to existing directories or files. In other words, if a URL is requested, and that URL points to a file or a directory then no further redirection will happen.

The last `RewriteRule` line says that everything else should be redirected to `public/index.php`. While the first `RewriteRule` line actually performs a redirect (of the type `301`), this `RewriteRule` line will redirect transparently.

A> This `public/.htaccess` file makes me think that Laravel is tailored to being used under Apache. The only other mainstream web server, that is commonly used to run PHP applications is Nginx, and it ignores `.htaccess` files.

A> There is documentation for setting Laravel up to rewrite URLs in Nginx. You can find it at **[http://laravel.com/docs/installation#pretty-urls](http://laravel.com/docs/installation#pretty-urls)**.

A> **[Lovro Papež](https://twitter.com/slovenianGooner)** has put together **[a guide for setting these pretty URLs up in IIS](http://sloveniangooner.com/post/laravel-and-iis)**.

## Autoloading Files

Laravel is built to work with Composer. All of the dependencies it uses are loaded via Composer, and all the class autoloading is handled by the standard Composer autoloader. Well, maybe not all the autoloading…

The first significant line, of `public/index.php` is:

{lang=php}
```
require __DIR__.'/../bootstrap/autoload.php';
```

A> This is from `public/index.php`

Heading over to that file, we see a bit of profiling code, followed immediately by the loading of Composer’s class autoloader:

{lang=php}
```
require __DIR__.'/../vendor/autoload.php';
```

A> This is from `bootstrap/autoload.php`

This tells us that all classes mapped in `composer.json` or via dependencies will be available in a Laravel application. This is followed by a check for a file named `bootstrap/compiled.php`. If this file looks familiar to you that probably because it’s mentioned in the `.gitignore` file that ships with new Laravel applications.

A> `bootstrap/compiled.php` is a massive concatenation of classes Laravel identifies as being important. It concatenates them to offset the overhead of autoloading them in separate IO request. You can load additional classes into this “compile” step by adding them to `app/config/compile.php`. This also explains why it should be ignored — it can be application-specific and always duplicates code that can otherwise be generated.

Yet another class autoloader is specified by:

{lang=php}
```
Illuminate\Support\ClassLoader::register();
```

A> This is from `bootstrap/autoload.php`

As the comment above it states; this class helps to load models (and presumably other classes) which haven’t been introduced through a call to `composer dump-autoload`.

And lastly, workbenches are included in the whole class autoloading cycle with:

{lang=php}
```
if (is_dir($workbench = __DIR__.'/../workbench'))
{
  Illuminate\Workbench\Starter::start($workbench);
}
```

A> This is from `bootstrap/autoload.php`

Peeking inside this class we see the that it looks for files matching `autoload.php` which is a reference to Composer autoloader files in individual workbench folders. This is confirmed by **[http://laravel.com/docs/packages#development-workflow](http://laravel.com/docs/packages#development-workflow)**.

A> Workbenches are in-development packages which are isolated from a Laravel application files while being included in the running of the application. You can create a new workbench with the `php artisan workbench` command. You can learn more about it at **[http://laravel.com/docs/packages#creating-a-package](http://laravel.com/docs/packages#creating-a-package)**.

## UTF-what?

I skipped the part of `bootstrap/autoload.php` that loads the **Patchwork UTF-8** library. It’s related to the shortcomings of PHP’s UTF-8 support, but we’ll not go into that right now.
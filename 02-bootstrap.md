# Drupal 7 Deconstructed: Part 2 - The Bootstrap Process

So George's request for `/about-us` been handed to Drupal, and `index.php` is ready too bootstrap Drupal. What does that mean?

At a code level, we're talking about the [`drupal_bootstrap`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_bootstrap/7) function, which lets you pass in a parameter to tell it which level of bootstrap you need. In almost all cases, we want a "full" bootstrap, which usually means "this is a regular page request, nothing weird, so just give me everything."

What is "everything"? I'm glad you asked. All of the possible values for the parameter for `drupal_bootstrap()` are listed below. Note that they are run sequentially, meaning if you call it with `DRUPAL_BOOTSTRAP_CONFIGURATION` then it will only do that one (#1), but if you call it with `DRUPAL_BOOTSTRAP_SESSION` then it will do that one (#5) and all of the ones before it (#1-4). And since `DRUPAL_BOOTSTRAP_FULL` is last, calling it gives you everything in this list.

1. `DRUPAL_BOOTSTRAP_CONFIGURATION`: Set up some configuration
2. `DRUPAL_BOOTSTRAP_PAGE_CACHE`: Try to serve the page from the cache (in which case the rest of these steps don't run)
3. `DRUPAL_BOOTSTRAP_DATABASE`: Initialize the database connection
4. `DRUPAL_BOOTSTRAP_VARIABLES`: Load variables from the `variables` table
5. `DRUPAL_BOOTSTRAP_SESSION`: Initialize the user's session
6. `DRUPAL_BOOTSTRAP_PAGE_HEADER`: Set HTTP headers to prepare for a page response
7. `DRUPAL_BOOTSTRAP_LANGUAGE`: Initialize language types for multilingual sites
8. `DRUPAL_BOOTSTRAP_FULL`: Includes a bunch of other files and does some other miscellaneous setup.

Each of these are defined in more detail below.

## `DRUPAL_BOOTSTRAP_CONFIGURATION`

This guy just calls [`_drupal_bootstrap_configuration()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/_drupal_bootstrap_configuration/7), which in turn does the following:

### Sets error and exception handlers.

```php
set_error_handler('_drupal_error_handler');
set_exception_handler('_drupal_exception_handler');
```

These lines set a custom error handler ([`_drupal_error_handler()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/_drupal_error_handler/7)) and a custom exception handler ([`_drupal_exception_handler`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/_drupal_error_handler/7)) respectively. That means that those functions are called when Drupal encounters a PHP error or exception.

These functions each go a few levels deep, but all they're really doing is attempting to log any errors or exceptions that may occur, and then throw a `500 Service unavailable` response.

### Initializes the PHP environment

```php
drupal_environment_initialize()
```

The [`drupal_environment_initialize()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_environment_initialize/7) function called here doesn't do much that's worth writing home for. 

- It tinkers with the global `$_SERVER` array a little bit.
- It sets the configuration for error reporting
- It sets some session configuration using `ini_set()`

Boring.

### Starts a timer

```php
timer_start('page');
```

This is actually pretty nifty. Drupal has a global `$timers` variable that many people don't know about. 

Here, a timer is started so that the time it takes to render the page can be measured.

### Initializes some critical settings

```php
drupal_settings_initialize();
```

The [`drupal_settings_initialize()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_settings_initialize/7) function is super important, for exactly 2 reasons:

1. It includes the all-important `settings.php` file which contains our databas connection info (which isn't used yet), among other things.
2. It creates many of our favorite global variables, such as `$cookie_domain`, `$conf`, `$is_https`, and more!

And that's the end of the CONFIGURATION bootstrap. 1 down, 7 to go!

## `DRUPAL_BOOTSTRAP_PAGE_CACHE`

When bootstrapping the page cache, everything happens inside [`_drupal_bootstrap_page_cache()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/_drupal_bootstrap_page_cache/7) which does a lot of work.

### Includes cache.inc and any custom cache backends

```php
require_once DRUPAL_ROOT . '/includes/cache.inc';
foreach (variable_get('cache_backends', array()) as $include) {
  require_once DRUPAL_ROOT . '/' . $include;
}
```

This bit of fancyness allows us to specify our own cache backend(s) instead of using Drupal's database cache. 

This is most commonly used to support memcache, but someone could really go to town with this if they wanted, just by specifying (in the `$conf` array in `settings.php` an include file to use (such as `memcache.inc`) for whatever cache backend they're wanting to use.

### Checks to see if cache is enabled

```php
// Check for a cache mode force from settings.php.
if (variable_get('page_cache_without_database')) {
  $cache_enabled = TRUE;
}
else {
  drupal_bootstrap(DRUPAL_BOOTSTRAP_VARIABLES, FALSE);
  $cache_enabled = variable_get('cache');
}
```

You'll note that the first line there gives you a way to enable cache from `settings.php` directly. This speeds things up because that way it doesn't need to bootstrap `DRUPAL_BOOTSTRAP_VARIABLES` (i.e., load all of the variables from the DB table) which would also force it to bootstrap `DRUPAL_BOOTSTRAP_DATABASE`, which is a requirement for fetching the variables from the database, all just to see if the cache is enabled.

So assuming you don't have `$conf['page_cache_without_database'] = TRUE` in your `settings.php` file, then we will be bootstrapping the variables here, which in turn bootstraps the database. Both of those will be talked about in more info in a minute.

### Blocks any IP blacklisted users

```php
drupal_block_denied(ip_address());
```

This does exactly what you'd expect - checks to see if the user's IP address is in the list of blacklisted addresses, and if so, returns a `403 Forbidden` response. This doesn't strictly have anything to do with caching, except for the fact that it needs to block cached responses from blacklisted users and this is its last chance to do that.

An interesting thing to note here is that the [`ip_address()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/ip_address/7) function is super useful. On a normal site it just returns regular old `$_SERVER['REMOTE_ADDR']`, but if you're using some sort of reverse proxy in front of Drupal (meaning `$_SERVER['REMOTE_ADDR']` will always be the same), then it fetches the user's IP from the (configurable) HTTP header. Pretty awesome. 

But beware that if you have `$conf['page_cache_without_database'] = TRUE` in `settings.php` then it won't fetch blocked IPs from the database, because it wouldn't have bootstrapped `DRUPAL_BOOTSTRAP_VARIABLES` yet (re-read the previous section to see what I mean). Tricky, tricky!

### Checks to see if there's a session cookie

```php
if (!isset($_COOKIE [session_name()]) && $cache_enabled) {
  ...fetch and return cached response if there is one...
}
```

It only returns a cached response (assuming one exists to return) if the user doesn't have a valid session cookie. This is a way of ensuring that only anonymous users see cached pages, and authenticated users don't. (What's that? [You want authenticated users to see cached pages too](https://ohthehugemanatee.org/blog/2014/06/09/authenticated-user-caching-in-drupal/)?)

What's inside that "fetch and return cached response" block? Lots of stuff!

### Populates the global $user object

```php
$user = drupal_anonymous_user();
```

The [`drupal_anonymous_user()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_anonymous_user/7) function just creates an empty user object with a `uid` of 0.  We're creating it here just because it may need to be used later on down the line, such as in some `hook_boot()` implementation, and also because its timestamp will be checked and possibly logged.

### Checks to see if the page is already cached

```php
$cache = drupal_page_get_cache();
```

The [`drupal_page_get_cache()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_page_get_cache/7) function is actually simpler than you'd think. It just checks to see if the page is cacheable (i.e., if the request method is either `GET` or `HEAD`, as told in [`drupal_page_is_cacheable()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_page_is_cacheable/7)), and if so, it runs `cache_get()` with the current URL against the `cache_page` database table, to fetch the cache, if there is one.

### Serves the response from that cache

If `$cache` from the previous section isn't empty, then we have officially found ourselves a valid page cache for the current page, and we can return it and shut down. This block of code does a few things:

- Sets the page title using [`drupal_set_title()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_set_title/7)
- Sets a HTTP header: `X-Drupal-Cache: HIT`
- Sets PHP's default timezone to the site's default timezone (from `variable_get('date_default_timezone')`)
- Runs all implementations of `hook_boot()`, if the `page_cache_invoke_hooks` variable isn't set to FALSE.
- Serves the page from cache, using [`drupal_serve_page_from_cache($cache)`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_serve_page_from_cache/7), which is scary looking but basically just adds some headers and prints the cache data (i.e., the page body).
- Runs all implementations of `hook_exit()`, if the `page_cache_invoke_hooks` variable isn't set to FALSE.

And FINALLY, once all of that is complete, it runs `exit;` and we're done, assuming we got this far. 

Otherwise, it doesn't do any of the above, and just sets the `X-Drupal-Cache: MISS` header.

Whew. That's a lot of stuff. Luckily, the next section is easier.

## `DRUPAL_BOOTSTRAP_DATABASE`

We're not going to get super in the weeds with everything Drupal does with the database here, since that deserves its own chapter, but here's an overview of the parts that happen while bootstrapping, within the [`_drupal_bootstrap_database ()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/_drupal_bootstrap_database/7) function.

### Checks to see if we have a database configured

```php
if (empty($GLOBALS ['databases']) && !drupal_installation_attempted()) {
  include_once DRUPAL_ROOT . '/includes/install.inc';
  install_goto('install.php');
}
```

Nothing fancy. If we don't have anything in `$GLOBALS ['databases']` and we haven't already started the installation process, then we get booted to `/install.php` since Drupal is assuming we need to install the site.

### Includes the `database.inc` file

This beautiful beautiful [`database.inc`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/7) file includes all of the database abstraction functions that we know and love, such as `db_query()` and `db_select()` and `db_update()`. 

It also has holds the base `Database` and `DatabaseConnection` and `DatabaseTransaction` classes (among a bunch of others).

It's a 3000+ line file, so it's out of scope for a discussion on bootstrapping, and we'll get back to "How Drupal Does Databases" in a later chapter.

### Registers autoload functions for classes and interfaces

```php
spl_autoload_register('drupal_autoload_class');
spl_autoload_register('drupal_autoload_interface');
```

This is just a tricky way of ensuring that a class or interface actually exists, when we try to autoload one. Both [`drupal_autoload_class()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_autoload_class/7) and [`drupal_autoload_interface()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_autoload_interface/7) just call [`registry_check_code()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/_registry_check_code/7), which looks for the given class or interface first in the `cache_bootstrap` table, then `registry` table if no cache is found. 

If it finds the class or interface, it will `require_once` the file that contains that class or interface. Otherwise, it just returns FALSE so Drupal knows that somebody screwed the pooch and we're looking for a class or interface that doesn't exist.
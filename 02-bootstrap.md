# Drupal 7 Deconstructed: Part 2 - The Bootstrap Process

So George's request for `/about-us` been handed to Drupal, and `index.php` is ready to bootstrap Drupal. What does that mean?

## A quick summary

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

## 1. `DRUPAL_BOOTSTRAP_CONFIGURATION`

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

1. It includes the all-important `settings.php` file which contains our database connection info (which isn't used yet), among other things.
2. It creates many of our favorite global variables, such as `$cookie_domain`, `$conf`, `$is_https`, and more!

And that's the end of the CONFIGURATION bootstrap. 1 down, 7 to go!

## 2. `DRUPAL_BOOTSTRAP_PAGE_CACHE`

When bootstrapping the page cache, everything happens inside [`_drupal_bootstrap_page_cache()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/_drupal_bootstrap_page_cache/7) which does a lot of work.

### Includes cache.inc and any custom cache backends

```php
require_once DRUPAL_ROOT . '/includes/cache.inc';
foreach (variable_get('cache_backends', array()) as $include) {
  require_once DRUPAL_ROOT . '/' . $include;
}
```

This bit of fanciness allows us to specify our own cache backend(s) instead of using Drupal's database cache. 

This is most commonly used to support memcache, but someone could really go to town with this if they wanted, just by specifying (in the `$conf` array in `settings.php`) an include file to use (such as `memcache.inc`) for whatever cache backend they're wanting to use.

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

## 3. `DRUPAL_BOOTSTRAP_DATABASE`

We're not going to get super in the weeds with everything Drupal does with the database here, since that deserves its own chapter, but here's an overview of the parts that happen while bootstrapping, within the [`_drupal_bootstrap_database()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/_drupal_bootstrap_database/7) function.

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

If it finds the class or interface, it will `require_once` the file that contains that class or interface and return `TRUE`. Otherwise, it just returns `FALSE` so Drupal knows that somebody screwed the pooch and we're looking for a class or interface that doesn't exist.

So, in English, it's saying "*Ok, it looks like you're trying to autoload a class or an interface, so I'll figure out which file it's in by checking the cache or the registry database table, and then include that file, if I can find it.*"

## 4. `DRUPAL_BOOTSTRAP_VARIABLES`

This one just calls [`_drupal_bootstrap_variables()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/_drupal_bootstrap_variables/7), which actually does a good bit more than just including the variables from the variables table. 

Here's what it does:

### Initializes the locking system

```php
require_once DRUPAL_ROOT . '/' . variable_get('lock_inc', 'includes/lock.inc');
lock_initialize();
```

Drupal's locking system allows us to create arbitrary locks on certain operations, to prevent race conditions and other bad things. If you're interested to read more about this, there is a very good [API page about it](https://api.drupal.org/api/drupal/includes!lock.inc/group/lock/7).

The two lines of code here don't actually acquire any locks, they just initialize the locking system so that later code can use it. In fact, it's actually used in the very next section, which is why it's initialized in this seemingly unrelated phase of the bootstrap process.

### Load variables from the database

```php
global $conf;
$conf = variable_initialize(isset($conf) ? $conf : array());
```

The [`variable_initialize()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/variable_initialize/7) function basically just returns everything from the `variables` database table, which in this case adds it all to the global `$conf` array, so that we can `variable_get()` things from it later.

But there are a few important details:

1. It tries to load from the cache first, by looking for the `variables` cache ID in the `cache_bootstrap` table.
2. Assuming the cache failed, it tries to acquire a lock to avoid a stampede if a ton of requests are all trying to grab the `variables` table at the same time. 
3. Once it has the lock acquired, it grabs everything from the `variables` table.
4. Then it caches all of that, so that step 1 won't fail next time.
5. Finally, it releases the lock.

### Load all "bootstrap mode" modules

```php
require_once DRUPAL_ROOT . '/includes/module.inc';
module_load_all(TRUE);
```

Note that this may seem scary (OH MY GOD we're loading every single module just to bootstrap the variables!) but it's not. That `TRUE` is a big deal, because that tells Drupal to only load the "bootstrap" modules. A "bootstrap" module is a module that has the `bootstrap` column in the `system` table set to 1 for it. 

On the typical Drupal site, this will only be a handful of modules that are specifically required this early in the bootstrap, like the Syslog module or the System module, or some contrib modules like Redirect or Variable.

### Sanitize the `destination` URL parameter

Here's another one that you wouldn't expect to happen as part of bootstrapping variables. 

The `$_GET['destination']` parameter needs to be protected against open redirect attacks leading to other domains. So what we do here is to check to see if it's set to an external URL, and if so, we unset it. 

The reason we have to wait for the variables bootstrap for this is that we need to call the [`url_is_external()`](https://api.drupal.org/api/drupal/includes%21common.inc/function/url_is_external/7) function to do it, and that function a variable to store the list of allowed protocols.

## 5. `DRUPAL_BOOTSTRAP_SESSION`

Bootstrapping the session means requiring the `session.inc` file and then running [`drupal_session_initialize()`](https://api.drupal.org/api/drupal/includes%21session.inc/function/drupal_session_initialize/7), which is a pretty fun function.

### Register custom session handlers

The first thing that happens here is that Drupal registers custom session handlers with PHP:

```php
session_set_save_handler('_drupal_session_open', '_drupal_session_close', 
  '_drupal_session_read', '_drupal_session_write', 
  '_drupal_session_destroy', '_drupal_session_garbage_collection');
```

If you've never seen the [`session_set_save_handler()`](http://php.net/session_set_save_handler) PHP function before, it just allows you to set your own custom session storage functions, so that you can have full control over what happens when sessions are opened, closed, read, written, destroyed, or garbage collected. As you can see above, Drupal implements its own handlers for all 6 of those.

What does Drupal actually do in those 6 handler functions? 

- `_drupal_session_open()` and `_drupal_session_close()` both literally just `return TRUE;`.
- `_drupal_session_read()` fetchs the session from the `sessions` table, and does a join on the `users` table to include the user data along with it.
- `_drupal_session_write()` checks to see if the session has been updated in the current page request or more than 180 seconds have passed since the last update, and if so, it gathers up session data and drops it into the `sessions` table with a `db_merge()`.
- `_drupal_session_destroy()` just deletes the appropriate row from the `sessions` DB table, sets the global `$user` object to be the anonymous user, and deletes cookies.
- `_drupal_session_garbage_collection()` deletes all sessions from the `sessions` table that are older than whatever the max lifetime is set to in PHP (i.e., whatever `session.gc_maxlifetime` is set to).

### If we already have a session cookie, then start the session

We then check to see if there's a valid session cookie in `$_COOKIE[session_name()]`, and if so, we run the [`drupal_session_start()`](https://api.drupal.org/api/drupal/includes%21session.inc/function/drupal_session_start/7). If you're a PHP developer and you just want to know where `session_start()` happens, then you've found it.

That's basically all that `drupal_session_start()` does, besides making sure that we're not a command line client and we haven't already started the session.

### Disable page cache for this request

Remember back in the `DRUPAL_BOOTSTRAP_PAGE_CACHE` section where I said that authenticated users don't get cached pages (unless you use something outside of Drupal core)? This is the part that makes that happen.

```php
if (!empty($user->uid) || !empty($_SESSION)) {
  drupal_page_is_cacheable(FALSE);
}
```

So if we have a session or a nonzero user ID, then we mark this page as uncacheable, because we may be seeing user-specific data on it which we don't want anyone else to see.

### If we don't already have a session cookie, lazily start one

This part's tricky. Drupal lazily starts sessions at the end of the request, so all the bootstrap process has to do is create a session ID and tell $_COOKIE about it, so that it can get picked up at the end.

```php
session_id(drupal_random_key());
$insecure_session_name = substr(session_name(), 1);
$session_id = drupal_random_key();
$_COOKIE[$insecure_session_name] = $session_id;
```

I won't go in detail here since we're talking about the bootstrap, but at the end of the request, `drupal_page_footer()` or `drupal_exit()` (depending on which one is responsible for closing this particular request) will call [`drupal_session_commit()`](https://api.drupal.org/api/drupal/includes%21session.inc/function/drupal_session_commit/7), which checks to see if there's anything in $_SESSION that we need to save to the database, and will run `drupal_session_start()` if so.

### Sets PHP's default timezone from the user's timezone

```php
date_default_timezone_set(drupal_get_user_timezone());
```

You may remember that the cache bootstrap above was responsible for setting the timezone for cached pages. This is where the timezone gets set for uncached pages. 

The [`drupal_get_user_timezone()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_get_user_timezone/7) is very simple. It just checks to see if user-configurable timezones are enabled and the user has one set, and uses that if so, otherwise it falls back to the site's default timezone setting.

## 6. `DRUPAL_BOOTSTRAP_PAGE_HEADER`

This is probably the simplest of the bootstrap levels. It does 2 very simple things in the [`_drupal_bootstrap_page_header()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/_drupal_bootstrap_page_header/7) function.

### Invokes hook_boot()

```php
bootstrap_invoke_all('boot');
```

If you've ever wondered how much of the bootstrap process has to complete before you can be guaranteed that hook_boot will run, there's your answer. Remember that for cached pages, it will have already run back in the cache bootstrap, but for uncached pages, this is where it happens.

### Sends initial HTTP headers 

There's a little bit of a call stack here. `drupal_page_header()` calls `drupal_send_headers()` which calls `drupal_get_http_header()` to finally fetch the headers that it needs to send.

Note that in this run, it just sends a couple default headers (`Expires` and `Cache-Control`), but the interesting part is that static caches are used throughout, and anything can call [`drupal_add_http_header()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_add_http_header/7) later on down the line, which will also call `drupal_send_headers()`. This allows you to append or replace existing headers before they actually get sent anywhere.


## 7. `DRUPAL_BOOTSTRAP_LANGUAGE`

In this level, the [`drupal_language_initialize()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_language_initialize/7) function is called. This function only really does anything if we're talking about a multilingual site. It checks `drupal_multilingual()` which just returns `TRUE` if the list of languages is greater than 1, and false otherwise.

If it's not a multilingual site, it cuts out then.

If it is a multilingual site, then it initializes the system using `language_initialize()` for each of the language types that been configured, and then runs all `hook_language_init()` implementations. 

This is a good time to note that the language system is complicated and confusing, with a web of "language types" (such as `LANGUAGE_TYPE_INTERFACE` and `LANGUAGE_TYPE_CONTENT`) and "language providers", and of course actual languages. It deserves a chapter of its own, so I'm not going to go into any more detail here.

## 8. `DRUPAL_BOOTSTRAP_FULL`

And we have landed. Now that we already have the building blocks like a database and a session and configuration, we can add All Of The Other Things. And the [`_drupal_bootstrap_function()`](https://api.drupal.org/api/drupal/includes%21common.inc/function/_drupal_bootstrap_full/7) does just that.

### Requires a ton of files

```php
require_once DRUPAL_ROOT . '/' . variable_get('path_inc', 'includes/path.inc');
require_once DRUPAL_ROOT . '/includes/theme.inc';
require_once DRUPAL_ROOT . '/includes/pager.inc';
require_once DRUPAL_ROOT . '/' . variable_get('menu_inc', 'includes/menu.inc');
require_once DRUPAL_ROOT . '/includes/tablesort.inc';
require_once DRUPAL_ROOT . '/includes/file.inc';
require_once DRUPAL_ROOT . '/includes/unicode.inc';
require_once DRUPAL_ROOT . '/includes/image.inc';
require_once DRUPAL_ROOT . '/includes/form.inc';
require_once DRUPAL_ROOT . '/includes/mail.inc';
require_once DRUPAL_ROOT . '/includes/actions.inc';
require_once DRUPAL_ROOT . '/includes/ajax.inc';
require_once DRUPAL_ROOT . '/includes/token.inc';
require_once DRUPAL_ROOT . '/includes/errors.inc';
```

All that stuff that we haven't needed yet but may need after this, we require here, just in case. That way, we're not having to load `ajax.inc` on the fly if we happen to be using AJAX later, or `mail.inc` on the fly if we happen to be sending an email.


### Load all enabled modules

The [`module_load_all(TRUE)`](https://api.drupal.org/api/drupal/includes%21module.inc/function/module_load_all/7) does exactly what you'd expect - grabs the name of every enabled module using `module_list()` and then runs `drupal_load()` on it to load it. There's also a static cache in this function so that it only runs once per request.

### Registers stream wrappers

The [`file_get_stream_wrappers()`](https://api.drupal.org/api/drupal/includes%21file.inc/function/file_get_stream_wrappers/7) has a lot of meat to it, but it's all details around a fairly simple task.

At a high level, it's grabbing all stream wrappers using `hook_stream_wrappers()`, allowing the chance to alter them using `hook_stream_wrappers_alter()`, and then registering (or overriding) each of them using `stream_wrapper_register()`, which is a plain old PHP function. It then sticks the result in a static cache so that it only runs all of this once per request.

### Initializes the path

The [`drupal_path_initialize()`](https://api.drupal.org/api/drupal/includes%21path.inc/function/drupal_path_initialize/7) function is called which just makes sure that `$_GET['q']` is setup (if it's not, then it sets it to the frontpage), and then runs it through [`drupal_get_normal_path()`](https://api.drupal.org/api/drupal/includes%21path.inc/function/drupal_get_normal_path/7) to see if it's a path alias, and if so, replace it with the internal path. 

This also gives modules a chance to alter the inbound URL. Before `drupal_get_normal_path()` returns the path, it calls all implementations of `hook_url_inbound_alter()` to do just that.

### Sets and initializes the site theme

```php
menu_set_custom_theme();
drupal_theme_initialize();
```

These two fairly innocent looking functions are NOT messing around. 

The purpose of [`menu_set_custom_theme()`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/menu_set_custom_theme/7) is to allow modules or theme callbacks to dynamically set the theme that should be used to render the current page. To do this, it  calls [`menu_get_custom_theme(TRUE)`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/menu_get_custom_theme/7), which is a bit scary looking, but doesn't do anything important besides that and saving the result to a static cache.

After that, the [`drupal_theme_initialize()`](https://api.drupal.org/api/drupal/includes%21theme.inc/function/drupal_theme_initialize/7) comes along and goes to town.

First, it just loads all themes using [`list_themes()`](https://api.drupal.org/api/drupal/includes%21theme.inc/function/list_themes/7), which is where the `.info` file for each theme gets parsed and the lists of CSS files, JS files, regions, etc., get populated.

Secondly, it tries to find the theme to use by checking to see if the user has a custom theme set, and if not, falling back to the `theme_default` variable.

```php
$theme = !empty($user->theme) && drupal_theme_access($user->theme) ? 
  $user->theme : variable_get('theme_default', 'bartik');
```

Then it checks to see if a different custom theme was chosen on the fly in the previous step (the `menu_set_custom_theme()` function), by running `menu_get_custom_theme()` (remember that static cache). If there was a custom theme returned, then it uses that, otherwise it keeps the default theme.

```php
$custom_theme = menu_get_custom_theme();
$theme = !empty($custom_theme) ? $custom_theme : $theme;
```

Once it has firmly decided on what dang theme is going to render the dang page, it can move on to building a list of base themes or ancestor themes.

```php
$base_theme = array();
$ancestor = $theme;
while ($ancestor && isset($themes [$ancestor]->base_theme)) {
  $ancestor = $themes [$ancestor]->base_theme;
  $base_theme [] = $themes [$ancestor];
}
```

It needs that list because it needs to initialize any ancestor themes along with the main theme, so that theme inheritance can work. So then it runs [`_drupal_theme_initialize`](https://api.drupal.org/api/drupal/includes%21theme.inc/function/_drupal_theme_initialize/7) on each of them, which adds the necessary CSS and JS, and then initializes the correct theme engine, if needed.

After that, it resets the `drupal_alter` cache, because themes can have alter hooks, and we wouldn't want to ignore them becuase we had already built the cache by now.

```
drupal_static_reset('drupal_alter');
```

And finally, it adds some info to JS about the theme that's being used, so that if an AJAX request comes along later, it will know to use the same theme.

```php
$setting ['ajaxPageState'] = array(
  'theme' => $theme_key,
  'theme_token' => drupal_get_token($theme_key),
);
drupal_add_js($setting, 'setting');
```

### A couple other miscellaneous setup tasks

- Detects string handling method using `unicode_check()`.
- Undoes magic quotes using `fix_gpc_magic()`.
- Ensures `mt_rand` is reseeded for security.
- Runs all implementations of `hook_init()` at the very end.

## Conclusion

That's it. That's the entire bootstrap process. There are a lot of places that deserve some more depth, and we'll get there, but you should be feeling like you have a fairly good understanding of where and when things get set up while bootstrapping.

Keep in mind this is only a small part of the page load process. Most of the really heavy lifting happens after this, so keep reading!

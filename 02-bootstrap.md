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

<h1>MORE TO COME HERE</h1>

# Cron

Who hasn't battled with cron at one point or another? It's a rite of passage for a Drupal developer. 

In this chapter, we'll see what exactly triggers a cron run, what it does, and why it does it.

## A quick summary

First, cron gets triggered. This can happen via a request from an external cron system to [`cron.php`](https://api.drupal.org/api/drupal/cron.php/7), or using poor man's cron, or really any other custom code that decides it's necessary. Either way, we end up running [`drupal_cron_run()`](https://api.drupal.org/api/drupal/includes%21common.inc/function/drupal_cron_run/7).

Once called, that function gathers up all implementations of [`hook_cron()`](https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_cron/7) and runs them each.

That's it. Quick enough for you?

## Kicking things off

First, cron has to be triggered. There are a bunch of ways to go about this.

### Poking `cron.php`

The recommended way is to have some external system such as Jenkins or an OS crontab send a request to `yoursite.com/cron.php`. 

The `cron.php` file is so simple that I can get away with just posting the entire thing here:

```php
define('DRUPAL_ROOT', getcwd());

include_once DRUPAL_ROOT . '/includes/bootstrap.inc';
drupal_bootstrap(DRUPAL_BOOTSTRAP_FULL);

if (!isset($_GET['cron_key']) || variable_get('cron_key', 'drupal') != $_GET['cron_key']) {
  watchdog('cron', 'Cron could not run because an invalid key was used.', array(), WATCHDOG_NOTICE);
  drupal_access_denied();
}
elseif (variable_get('maintenance_mode', 0)) {
  watchdog('cron', 'Cron could not run because the site is in maintenance mode.', array(), WATCHDOG_NOTICE);
  drupal_access_denied();
}
else {
  drupal_cron_run();
}
```

If you're smart (which you are, and good looking), then you may have noticed that the first 3 lines of cron.php are exactly the same as `index.php`. Look back on the first chapter for more on that. This makes sense, because it's such a similar idea - both `cron.php` and `index.php` are the first file that gets hit for their particular requests (regular page requests vs. cron runs), so they have similar setup work to do.

Anyways, `cron.php` runs a full bootstrap to get things moving. First, it checks to make sure that a valid cron key exists in `?key=KEYGOESHERE` in the URL. Your cron key can be retrieved from `admin/config/system/cron` and it's literally just a random string that gets generated when the System module is first installed (i.e., in [`system_install`](https://api.drupal.org/api/drupal/modules%21system%21system.install/function/system_install/7)), like so:

```php
$cron_key = drupal_random_key();
variable_set('cron_key', $cron_key);
``` 

Requiring that key servers as a layer of protection against some wahoos pounding your cron or running it when you don't want it to be run, just by requesting `cron.php`. 

It then checks to make sure the site isn't in maintenance mode. 

If either of those two checks fail, meaning the site is in maintenance mode or the cron key is invalid, then it returns the *access denied* page and logs a `watchdog` notice.

Assuming they succeed, then we run `drupal_cron_run()`, and we're cooking with gas. 

### Poor man's cron

The second is by using Drupal's built-in method, commonly known as "poor man's cron". This allows you to fire a cron run every X hours/days which gets triggered by the first request to land after X hours/days pass.

On the `admin/config/system/cron`, you're presented with a fancy "Run cron every ____" dropdown, with options from 1 hour to 1 week. This field saves its result to the `cron_safe_threshold` variable.

Then, on every page load, once everything else is done, [`system_run_automated_cron()`](https://api.drupal.org/api/drupal/modules%21system%21system.module/function/system_run_automated_cron/7) is called at the very end of [`drupal_page_footer()`](https://api.drupal.org/api/drupal/includes%21common.inc/function/drupal_page_footer/7), which itself is called at the very end of [`drupal_deliver_html_page()`](https://api.drupal.org/api/drupal/includes%21common.inc/function/drupal_deliver_html_page/7), 

The `system_run_automated_cron()` function just grabs the timestamp of the most recent crun run ("a"), and the value of the `cron_safe_threshold` variable ("b"), and compares a + b to the current time to see if it's time to run cron again.

If it is, then our good friend `drupal_run_crun()` runs.

### Triggering cron manually

The third option is by manually calling `drupal_cron_run()` somewhere, which is basically the same thing as saying "everything else". 

This might happen by clicking the cron button from the `admin_menu` button, or by clicking the "run cron" button from the status report, or in custom code, or whatever.

The end result is the same - you're just running `drupal_cron_run()` directly instead of having cron.php or poor man's cron run it for you.

## Actually doing something

By now, we've reached `drupal_cron_run()` one way or another, so we're ready to actually run the dang cron.
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

Requiring that key serves as a layer of protection against some wahoos pounding your cron or running it when you don't want it to be run, just by requesting `cron.php`. 

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

By now, we've reached [`drupal_cron_run()`](https://api.drupal.org/api/drupal/includes%21common.inc/function/drupal_cron_run/7) one way or another, so we're ready to actually run the dang cron.

Here's everything that function does, from start to finish.

### Ignore request cancellations

```php
@ignore_user_abort(TRUE);
```

We don't want to stop the cron run if the request gets canceled, because cron really needs to run start to finish to be safe. That line does exactly that; it tells PHP that if the user aborts, then we will keep on trucking. [ignore_user_abort()](http://php.net/manual/en/function.ignore-user-abort.php) is a native PHP function that prevents PHP from aborting if the client disconnects. Prepending it with the [@ symbol](http://php.net/manual/en/language.operators.errorcontrol.php) causes any error messages raised by the ignore function itself to be ignored.

### Temporarily destroy the session

```php
$original_session_saving = drupal_save_session();
drupal_save_session(FALSE);
$original_user = $GLOBALS['user'];
$GLOBALS['user'] = drupal_anonymous_user();
```

Here, we remove the user's session and make him or her anonymous, while saving the original session and user to an object so that we can restore it in a bit.

This allows us to prevent any session information from being saved as cron runs. Otherwise, as a random example, a node that gets created during cron run could be auto-assigned to the current user, which doesn't make any sense.

It also allows us to enforce consistent permissions by ensuring that the user will always be an anonymous user. Otherwise, if users in different roles sometimes ran cron, they could theoretically see different results depending on their permissions, which are supposed to be irrelevant to cron.

### Set a time limit

```php
drupal_set_time_limit(240);
```

The [`drupal_set_time_limit()`](https://api.drupal.org/api/drupal/includes%21common.inc/function/drupal_set_time_limit/7) is basically just a wrapper around PHP's [`set_time_limit()`](http://php.net/set_time_limit) function to ensure that that function exists before attempting to run it.

Setting the time limit to 240 here means cron will run for up to 4 minutes before deciding that things are taking too long and throwing a fatal error. If you need to run tasks that might exceed the four minute limit, those should be added via the Queue API instead. 

### Acquire a lock

We obviously don't want to run cron if it's already running, so we do this:

```php
if (!lock_acquire('cron', 240.0)) {
  watchdog('cron', 'Attempting to re-run cron while it is already running.', array(), WATCHDOG_WARNING);
}
else {
  // Code for running cron is here
}
```

### Look for regular cron jobs and start them

This is the part that actually does the work. We just need to look for all implementations of [`hook_cron()`](https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_cron/7) and run them all, but it's important to be careful that if one of them throws an exception, the rest of them should still be allowed to run.

Here's the code for that:

```php
foreach (module_implements('cron') as $module) {
  try {
    module_invoke($module, 'cron');
  }
  catch (Exception $e) {
    watchdog_exception('cron', $e);
  }
}
```

### Wrap things up

With all `hook_cron()` implementations successfully having run, we can close things out. This means setting a variable showing when cron last completed, adding a debug message to watchdog, and releasing the aforementioned lock.

```php
variable_set('cron_last', REQUEST_TIME);
watchdog('cron', 'Cron run completed.', array(), WATCHDOG_NOTICE);
lock_release('cron');
```

So that is the end of what you might call the "regular" cron system, but Drupal has a neat feature for longer running cron jobs as well, and now is the time to kick that off.

### Look for cron queues and run them

It's a perhaps little known feature of Drupal that you can tell the cron system about a custom queue callback for long running tasks that can't be trusted to complete in the 240 seconds allowed by regular implementations of `hook_cron()`. "Queue" here refers to [`DrupalQueue`](https://api.drupal.org/api/drupal/modules%21system%21system.queue.inc/group/queue/7), a system for queueing items for later processing, which we will talk about in another chapter.

This is the code that sets that up:

```php
$queues = module_invoke_all('cron_queue_info');
drupal_alter('cron_queue_info', $queues);
```

So we're just running all implementations of [`hook_cron_queue_info()`](https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_cron_queue_info/7) and then running all implementations of [`hook_cron_queue_info_alter()`](https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_cron_queue_info_alter/7) to gather up any registered cron queues and allow altering them before running.

And then a little bit later, this part creates the queues:

```php
foreach ($queues as $queue_name => $info) {
   DrupalQueue::get($queue_name)->createQueue();
}
```

And then finally, a bit later (but still in [`drupal_cron_run()`](https://api.drupal.org/api/drupal/includes%21common.inc/function/drupal_cron_run/7)), we process the queues, making sure to skip any that have asked not to run on cron.

```php
foreach ($queues as $queue_name => $info) {
  if (!empty($info['skip on cron'])) {
    continue;
  }
  $function = $info['worker callback'];
  $end = time() + (isset($info['time']) ? $info['time'] : 15);
  $queue = DrupalQueue::get($queue_name);
  while (time() < $end && ($item = $queue->claimItem())) {
    try {
      $function($item->data);
      $queue->deleteItem($item);
    }
    catch (Exception $e) {
      watchdog_exception('cron', $e);
    }
  }
}
```

All that's doing is looping through the queue and running each item until it runs out of time, which defaults to 15 seconds but can be set to anything.

The aggregator module makes use of this for pulling in feeds, which can be a very time consuming process, but that's the only part of core that uses it.

### Restore the user session

Now that regular cron and cron queues have run, we're completely done in [`drupal_cron_run()`](https://api.drupal.org/api/drupal/includes%21common.inc/function/drupal_cron_run/7). But before we can close it out, we need to put back the user session that we took away at the beginning.

```php
$GLOBALS['user'] = $original_user;
drupal_save_session($original_session_saving);
```

And we're done! Yay, cron! A very useful and surprisingly simple system.

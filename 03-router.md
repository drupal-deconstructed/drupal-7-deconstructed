# Drupal 7 Deconstructed: Part 3 - The Menu Router

Alright! Now we're getting somewhere! Here's what we have so far. We have our request to `/about-us` and we've already run through a full Drupal bootstrap, thanks to `index.php`.

Now comes the fun part.

## A quick summary

If you want the nickel tour of the Drupal menu router (or you don't want to look at any code), then you're in the right place. Here's what you can expect.

First, the very last line of the `index.php` file kicks things off by telling the menu system to figure out what the request is asking for, and serve it. This happens by looking at the `menu_router` database table to see if there are any rows that match the current URL.

Assuming we find a match (in this case, our `about-us` path has been converted during the bootstrap process to something like `node/1234`, which makes with `node/%` in the menu_router table), we call whatever function the `menu_router` table tells us to call.

That function will be responsible for building the page content, and the rest is history (or, will be covered in other chapters).

Now, to dig a little deeper.

## Step 0. Fetching the system URL for a path alias

This step 0 because it has already happened. Remember in the Bootstrap chapter that, during the `DRUPAL_BOOTSTRAP_FULL` phase, the [`drupal_path_initialize()`](https://api.drupal.org/api/drupal/includes%21path.inc/function/drupal_path_initialize/7) function is called. That function just makes sure that `$_GET['q']` is set (if it's not, then it sets it to the frontpage URL). 

Then, more importantly, it runs through [`drupal_get_normal_path()`](https://api.drupal.org/api/drupal/includes%21path.inc/function/drupal_get_normal_path/7) to see if we are looking at a path alias rather than an internal path, and if so, replaces it with the internal path that it's aliasing.

All that is to say that by the time the bootstrap is done, `$_GET['q']` is set to `node/123` even though the initial request was for `about-us`, because it converts alias paths to system paths.

## Step 1. Kick things off from index.php

Our `index.php` file calls the grand [`menu_execute_active_handler()`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/menu_execute_active_handler/7) function.

First, it checks to see if the site is offline and bypasses a lot of logic if that's the case. It even gives modules a chance to have their say as to whether the site is offline, using [`hook_menu_site_status_alter()`](https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_menu_site_status_alter/7).

```php
$page_callback_result = _menu_site_is_offline() ? MENU_SITE_OFFLINE : MENU_SITE_ONLINE;
$read_only_path = !empty($path) ? $path : $_GET ['q'];
drupal_alter('menu_site_status', $page_callback_result, $read_only_path);
```

Assuming it's online, then things finally start to get interesting.

## Step 2: Figure out what we need to do with this path

From here, we call the [`menu_get_item($path)`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/menu_get_item/7) function, which does many things, all of which basically amount to "tell us everything we need to know about what to do with this path".

The first interesting thing it does is checks to see if we need to do a menu rebuild, and kicks one off if so.

```php
if (variable_get('menu_rebuild_needed', FALSE) || !variable_get('menu_masks', array())) {
  if (_menu_check_rebuild()) {
    menu_rebuild();
  }
}
```

We'll assume for now that we don't need to rebuild the menu table, but later in this chapter we'll talk about what happens there.

ANCESTORS

QUERY MENU ROUTER TABLE

ALTER HOOK

BLAH BLAH BLAH https://api.drupal.org/api/drupal/includes%21menu.inc/function/menu_get_item/7

## Step 3: Call the appropriate function for this path

## Step 4: Deliver the page

## What about building the menu router?
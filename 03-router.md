# The Menu Router

Alright! Now we're getting somewhere! Here's what we have so far. We have our request to `/about-us` and we've already run through a full Drupal bootstrap, thanks to `index.php`.

Now comes the fun part.

## A quick summary

If you want the nickel tour of the Drupal menu router (or you don't want to look at any code), then you're in the right place. Here's what you can expect.

First, the very last line of the `index.php` file kicks things off by telling the menu system to figure out what the request is asking for, and serve it. This happens by looking at the `menu_router` database table to see if there are any rows that match the current URL.

Assuming we find a match (in this case, our `about-us` path has been converted during the bootstrap process to something like `node/1234`, which matches `node/%` in the menu_router table), we call whatever function the `menu_router` table tells us to call.

That function will be responsible for building the page content, and the rest is history (or, will be covered in other chapters).

Now, to dig a little deeper.

## Step 0. Fetching the system URL for a path alias

We start with step 0 because it has already happened. Remember in the Bootstrap chapter that, during the `DRUPAL_BOOTSTRAP_FULL` phase, the [`drupal_path_initialize()`](https://api.drupal.org/api/drupal/includes%21path.inc/function/drupal_path_initialize/7) function is called. That function just makes sure that `$_GET['q']` is set (if it's not, then it sets it to the frontpage URL). 

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

From here, we call the [`menu_get_item($path)`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/menu_get_item/7) function, which does many things, all of which basically amount to "tell us everything we need to know about what to do with this path".

## Step 2: Possibly rebuild the menu system

The first interesting thing that function does is check to see if we need to do a menu rebuild, and kicks one off if so.

```php
if (variable_get('menu_rebuild_needed', FALSE) || !variable_get('menu_masks', array())) {
  if (_menu_check_rebuild()) {
    menu_rebuild();
  }
}
```

We'll assume for now that we don't need to rebuild the menu table, but later in this chapter we'll talk about what happens there.

## Step 3: Find the most relevant `menu_router` row for this path

Still inside `menu_get_item()`.

So with the assumption that we don't need to rebuild the menu system, we can continue on to actually finding what to do for the path we're on. To do that, we query the `menu_router` table. This table is the source of truth for "what do I need to do for this path?" 

Here's the structure of it:

```
mysql> explain menu_router;
+-------------------+--------------+------+-----+---------+-------+
| Field             | Type         | Null | Key | Default | Extra |
+-------------------+--------------+------+-----+---------+-------+
| path              | varchar(255) | NO   | PRI |         |       |
| load_functions    | blob         | NO   |     | NULL    |       |
| to_arg_functions  | blob         | NO   |     | NULL    |       |
| access_callback   | varchar(255) | NO   |     |         |       |
| access_arguments  | blob         | YES  |     | NULL    |       |
| page_callback     | varchar(255) | NO   |     |         |       |
| page_arguments    | blob         | YES  |     | NULL    |       |
| delivery_callback | varchar(255) | NO   |     |         |       |
| fit               | int(11)      | NO   | MUL | 0       |       |
| number_parts      | smallint(6)  | NO   |     | 0       |       |
| context           | int(11)      | NO   |     | 0       |       |
| tab_parent        | varchar(255) | NO   | MUL |         |       |
| tab_root          | varchar(255) | NO   | MUL |         |       |
| title             | varchar(255) | NO   |     |         |       |
| title_callback    | varchar(255) | NO   |     |         |       |
| title_arguments   | varchar(255) | NO   |     |         |       |
| theme_callback    | varchar(255) | NO   |     |         |       |
| theme_arguments   | varchar(255) | NO   |     |         |       |
| type              | int(11)      | NO   |     | 0       |       |
| description       | text         | NO   |     | NULL    |       |
| position          | varchar(255) | NO   |     |         |       |
| weight            | int(11)      | NO   |     | 0       |       |
| include_file      | mediumtext   | YES  |     | NULL    |       |
+-------------------+--------------+------+-----+---------+-------+
23 rows in set (0.00 sec)
```

That's a lot of stuff right? The primary key is the `path` field, and everything else is just data about the given path. Here, we can find access arguments/callbacks, page arguments/callbacks, whether the page is a local task (tab) or a menu item or just a plain old callback, any theming alterations to consider, and a ton of other stuff.

So we have a path, and we need to fetch all that other stuff about it. But unfortunately we can't just query for the path, because we care about ancestors. That is to say that, given a URL of `node/1234/edit`, all of the following need to be returned as matches (in order of best match to worst match):

- node/1234/edit
- node/1234/%
- node/%/edit
- node/%/%
- node/1234
- node/%
- node

Why? Well, because there probably isn't a row in `menu_router` with a path of exactly `node/1234/edit`. It's `node/%/edit` that will be our match, because that's the first one in that list that will actually exist in `menu_router`. 

But remember, this is a simple example. What about a custom path of `people/george/bio/picture/profile` which ends up matching with our custom path of `people/%/bio/picture/%`? Can you see why it's important to ask for all possible ancestor paths, and that way we can find the one that's the most specific match, and use that?

So before we query `menu_router`, we need to call [`menu_get_ancestors($path)`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/menu_get_ancestors/7) to fetch the possible ancestor paths for the given path. Only then can we query `menu_router` to find the path that fits the best. Here's how that all looks at a code level.

```php
$original_map = arg(NULL, $path);
$parts = array_slice($original_map, 0, MENU_MAX_PARTS);
$ancestors = menu_get_ancestors($parts);
$router_item = db_query_range('SELECT * FROM {menu_router} WHERE path IN (:ancestors) ORDER BY fit DESC', 0, 1, array(':ancestors' => $ancestors))->fetchAssoc();
```

## Step 4. Allow altering the router item

Still inside `menu_get_item()`.

Before we can continue on, we need to give modules the ability to alter the menu item right away, before anything else happens, using [`hook_menu_get_item_alter()`](https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_menu_get_item_alter/7). 

```php
drupal_alter('menu_get_item', $router_item, $path, $original_map);
```

This differs from the more commonly used [`hook_menu_alter()`](https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_menu_alter/7) in that this is at run-time, as opposed to `hook_menu_alter()` which only runs when the menu system is being built, and doesn't run as part of a page request.

## Step 5: Check for access to the router item

You guessed it! Still inside `menu_get_item()`.

This part is more confusing than you may expect. 

It's confusing first of all because it happens in a function called [`_menu_translate()`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/_menu_translate/7) which means "translate" as in "translate placeholders in the URL to loaded entities (or other things), when needed". (It also translates the menu title to the appropriate language later in this same function, confusing things further).

To start, it needs to translate placeholders (i.e., "%" signs) in paths to the loaded entity if a load function exists for it. For example, if our path in `menu_router` is `node/%node` then this chunk of code will say "ok, I see `%node` as opposed to just `%`, therefore, I know I need to run `node_load()` on whatever is in that part of the URL, which in this case is `1234`. 

To do this, it runs [`_menu_load_objects()`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/_menu_load_objects/7) which calls the appropriate load function if there is one and, assuming we don't get an access error, a missing entity, or anything else which would cause a problem, put the result (i.e., the fully loaded entity) back into the menu router item for later use. 

As a side note, this is a very common place to hit an _access denied_ page, in cases where the user doesn't have access to whichever entity we're trying to load. For example, an anonymous user trying to view an unpublished node would fail at this step.

So we've made it this far, and we have a loaded entity (remember that the `/about-us` page we're talking about is a node), but we're not in the clear yet. We still haven't run the `access_callback` function given to us from the `menu_router` table. 

That part happens via a call to [`_menu_check_access()`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/_menu_check_access/7), which basically runs the function set in `access_callback` if one exists, otherwise it falls back to [`user_access()`](https://api.drupal.org/api/drupal/modules%21user%21user.module/function/user_access/7), and includes any defined `access_arguments`. The fallback to `user_access()` means that we can just pass in a permission as `access_arguments` completely and leave out `access_callback` in our `hook_menu()` and it works out just peachy, which is a nice little shortcut.

**And we have finally reached the end of `menu_get_item()`! Hooray!**

## Step 6: Call the appropriate function for this path

Back inside `menu_execute_active_handler()`, and here's what we know at this point:

- We know that whether the user has access to this page or not
- We know what function to run if the user does have access
- We know which arguments to pass into that function, if any

We also know a lot of other stuff, but that's all details. Those 3 bullets are all we need at this point.

This little chunk of code is so nice and readable that I'm going to include it directly:

```php
if ($router_item['access']) {
  if ($router_item['include_file']) {
    require_once DRUPAL_ROOT . '/' . $router_item['include_file'];
  }
  $page_callback_result = call_user_func_array($router_item['page_callback'], $router_item['page_arguments']);
}
else {
  $page_callback_result = MENU_ACCESS_DENIED;
}
```

Nice and simple. If we have access, include any given include file if needed, call whatever the `page_callback` is, and include any given `page_arguments`. Or, if we don't have access, then just return the _access denied_ page.

## Step 7: Deliver the page

And finally, the work of the router is done, and we can just hand the work off to the callback assigned to this path.

```php
$default_delivery_callback = (isset($router_item) && $router_item) ? $router_item['delivery_callback'] : NULL;
drupal_deliver_page($page_callback_result, $default_delivery_callback);
```

Pretty, right? Yeah, not so much, but it definitely gets the job done.

## What about (re)building the menu router?

I mentioned earlier that we'd talk about how the menu router gets assembled in the first place. In other words, what exactly happens in the oh-so-scary [`menu_rebuild()`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/menu_rebuild/7) function that takes forever to run?

Well, it all happens in 3 simple steps.

```php
list($menu, $masks) = menu_router_build();
_menu_router_save($menu, $masks);
_menu_navigation_links_rebuild($menu);
```

Let's take it one by one.

### Gather the router items

The [`menu_router_build()`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/menu_router_build/7) function basically calls all `hook_menu()` implementations and builds a giant array out of them. 

This is actually an incredibly scary process. Don't be fooled by how simple `menu_router_build()` looks; the magic really happens in [`_menu_router_build()`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/_menu_router_build/7) (underscore functions strike again!). Take a look at that one to see what I mean. 

I won't go into much detail about what happens there, but the gist is that it takes the info provided to it by various `hook_menu()` implementations and figures out everything else that the `menu_router` table wants to store. That includes things like figuring out what the `load_function` should be based on named placeholders, figuring out if any given link exists in a custom navigation menu, inheriting access callbacks and page callbacks from parent paths if not explicitly defined, etc.

The end result is that we have all of the data given to us by all of the `hook_menu()` implementations, and it's in a format that `menu_router` will play nicely with.

### Save the router items

Finally, an easy one. The [`_menu_router_save()`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/_menu_router_save/7) function basically just does a giant `db_insert()` into the menu_router table for each menu item returned by `menu_router_build()`. 

It executes in batches of 20 to find a balance between performance and memory overhead, but other than that, there's nothing really tricky happening.

### Update the navigation menus

Since `hook_menu()` lets you place menu items in actual user facing navigation menus using the `MENU_NORMAL_ITEM` type and the optional `menu_name` key, we need to actually do the work of putting those items into navigation menus. This happens in [`_menu_navigation_links_rebuild`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/_menu_navigation_links_rebuild/7). 

This little guy handles a lot of tough work. For each menu router entry, it'll add it to the appropriate menu, or update it if it's already there, or move it from one menu to another if that changed, or delete it if needed. 

It's also smart enough to remove any items from menus that don't have matching router items anymore (so you don't have orphan menu items), or which have changed types from `MENU_NORMAL_ITEM` to another type. And to top it all off, it deletes them from the bottom up, meaning it starts with the ones with the greatest depth, so that it doesn't have to do as much re-parenting as it would if it deleted them in the opposite or a random order.

### When does this happen?

That's all well and good, but what would trigger this to happen? How does Drupal know when it's time to run rebuild the menu router?

Turns out that lots of things trigger that, but it's a bit complicated because there are two ways it can be triggered.

1. The `menu_rebuild()` function can be called directly, of course.
2. The `menu_rebuild_needed` variable can be set to `TRUE` which will trigger a call to `menu_rebuild()` the next time the `menu_get_item()` is called (see the *Possibly rebuild the menu system* section of this chapter to see how that happens).

There are a couple obvious things that trigger menu rebuilds. One of the most common is  [`drupal_flush_all_caches()`](https://api.drupal.org/api/drupal/includes%21common.inc/function/drupal_flush_all_caches/7) function. Besides being called whenever you manually flush caches (which is about 9000 times a day if you're anything like me), it also gets called by [`system_modules_submit()`](https://api.drupal.org/api/drupal/modules%21system%21system.admin.inc/function/system_modules_submit/7) when enabling a new module, so that the module's `hook_menu()` or `hook_menu_alter()` implementations (or other menu related hooks) can be respected.

Besides that one, there are a few other places that call `menu_rebuild()` directly. Most notably:

- [`node_type_form_submit()`](https://api.drupal.org/api/drupal/modules%21node%21content_types.inc/function/node_type_form_submit/7) - for adding tabs and paths for a content type that is being created or updated
- [`node_type_delete_confirm_submit()`](https://api.drupal.org/api/drupal/modules%21node%21content_types.inc/function/node_type_delete_confirm_submit/7) - for doing the exact opposite: deleting all that stuff.
- [`theme_enable()`](https://api.drupal.org/api/drupal/includes%21theme.inc/function/theme_enable/7) and [`theme_disable()`](https://api.drupal.org/api/drupal/includes%21theme.inc/function/theme_disable/7) - for adding or removing tabs and paths related to a theme that is being enabled or disabled

As far as places which set the `menu_rebuild_needed` variable to `TRUE`, achieving the same effect, we have a couple notable ones:

- [`field_ui_field_attach_create_bundle()`](https://api.drupal.org/api/drupal/modules%21field_ui%21field_ui.module/function/field_ui_field_attach_create_bundle/7) - if we're adding a new bundle to a fieldable entity type, then we need to add menu item tabs for it
- [`image_system_file_system_settings_submit()`](https://api.drupal.org/api/drupal/modules%21image%21image.module/function/image_system_file_system_settings_submit/7) - if we're saving the File System configuration form and update the public files path setting, then we need to rebuild the menu to reflect the new path.

Those are the big ones as far as core is concerned. If you search the standard Drupal site's codebase for `menu_rebuild` then you'll find a bunch more, but most of them belong to either contrib modules or automated test cases. Runtime core code does a relatively good job of only doing a full menu rebuild when it's absolutely necessary.
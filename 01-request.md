# Drupal 7 Deconstructed: Part 1 - The Request

Have you ever wondered how Drupal does what it does? Good, me too. In this series of posts, I'm going to explain what Drupal is doing behind the scenes to perform its magic.

In Part 1, we'll keep it fairly high level and walk through the path a request takes as it moves through Drupal. In later parts, we'll take deeper dives into individual pieces of this process.

## Step 0: Some background information

For this example, let's pretend that George, a user of your site, wants to visit your About Us page, which lives at `http://oursite/about-us`. 

Let's also pretend that this page is a node (specifically, the node with `nid` of `1`) with a URL alias of `about-us`.

And to keep things simple, we'll say that we're using Apache as our webserver, and there's nothing fancy sitting in front of Drupal, such as a caching layer or the like. 

**So basically, we're talking about a plain old Drupal site on a plain old webserver.**

## Step 1: The request hits the server

There's some pretty hot action that happens before Drupal even sees the request. George's browser sends a request to `http://oursite.com/about-us`, and this thing we call the internet figures out that that should go to our server. If you're not well versed on how that part happens, you may benefit from reading [this infographic on how the internet works](http://i.imgur.com/fzY1Arg.jpg).

Once our server has received the request, that's when the fun begins. This request will land on port `80`, and Apache just so happens to be listening on that port, so Apache will immediately see the request and decide what to do with it. 

Since the request is going to `oursite.com` then Apache can look into its configuration files to see what the root directory is for `oursite.com` (along with lots of other info about it which is out of scope for this post). We'll say that the root directory for our site is `/var/www/oursite`, which is where Drupal lives. So Apache is going to send the request there.

## Step 2: The .htaccess file 

But Drupal hasn't taken over just yet. Drupal ships with a `.htaccess` file which is a way of telling Apache some things. In fact, Drupal's `.htaccess` tells Apache a whole lot of things. The most important things it does are:

1. Protects files that could contain source code from being viewable, such as `.module` files or `.inc` files, both of which contain PHP.
2. Allows requests that point directly to files in the filesystem to pass through without touching Drupal.
3. Redirects all other requests to Drupal's `index.php` file.

It also does some other fancy things such as disabling directory indexes and allowing Drupal to serve gzipped CSS and JS, but those are the biggies.

So, in our case, Apache will see that we're looking for `/about-us`, and will say:

1. *Is this request trying to access a file that it shouldn't? No.*
2. *Is this request directly pointing to a file in the filesystem? No.*
3. *Then this request should be sent to `index.php`!*

And away we go...

## Step 3: Drupal's index.php file

Finally, we have reached Drupal, and we're looking at PHP. Drupal's index.php is super clean, and in fact only has 4 lines of code, each of which are easy to understand.

### Line 1: Define DRUPAL_ROOT

```php
define('DRUPAL_ROOT', getcwd());
```

This just sets a constant called `DRUPAL_ROOT` to be the value of the current directory that `index.php` is in. This constant is [used all over the place](https://api.drupal.org/api/drupal/index.php/constant/constants/DRUPAL_ROOT/7) in the Drupal core codebase.

In our case, this means that `DRUPAL_ROOT` gets set to `/var/www/oursite`.

### Lines 2 and 3: Bootstrap Drupal

```php
require_once DRUPAL_ROOT . '/includes/bootstrap.inc';
drupal_bootstrap(DRUPAL_BOOTSTRAP_FULL);
```

These lines run a full Drupal bootstrap, which basically means that they tell Drupal "Hey, grab all of the stuff you're going to need before we can get to actually handling this request."

**For more information about the bootstrap process, see Part 2 of this series.**

### Line 4: Do everything else

```php
menu_execute_active_handler();
```

This simple looking function is where the heart and soul of Drupal lives. 

**For more information about what happens in this ball of wax, visit Part 3 of this series.**
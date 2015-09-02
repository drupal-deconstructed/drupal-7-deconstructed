# The Form System

**Note: this chapter is not complete!**

Ah yes. The form system. Who doesn't love the Drupal 7 form system, right? Right?

The typical Drupal developer can probably tell you how to build a form and what the big array of fields should look like, but it's very doubtful that that developer has any idea how Drupal takes that big array and converts it into an actual HTML form.

Fear not! By the end of this chapter you will be that developer no longer!

## A quick summary

The form system is very easy to understand.

![Form system call stack](http://i.imgur.com/lhcS4LX.png)

Hope that helps!

## Kick things off

In the beginning, there is (usually) a call to [`drupal_get_form()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/drupal_get_form/7). This could be the function defined as a page callback in `hook_menu()`, or it could be called from a `hook_block_view()`, or it could be included in the `content_type` plugin for a Panels pane, or any number of other things. But whatever you're talking about, if it involves displaying a form, then you're probably going to be calling `drupal_get_form()` to fetch it, and you're probably going to be passing in the name of your custom form builder function as the `$form_id` parameter.

So let's start there. Here's what the function looks like:

```php
function drupal_get_form($form_id) {
  $form_state = array();

  $args = func_get_args();
  // Remove $form_id from the arguments.
  array_shift($args);
  $form_state ['build_info']['args'] = $args;

  return drupal_build_form($form_id, $form_state);
}
```

This is already a little tricky because of the argument handling. The `drupal_get_form()` function is kind of neat in that it expects the first argument to be `$form_id` (which will typically be the name of the function that builds the form array and returns it), but you can pass in any arbitrary arguments after that first one to provide your form builder function with any data that it might need. You can see that anything you pass in after `$form_id` gets added to `$form_state ['build_info']['args']` so that in your form builder function, you can fetch it out of there. 

Anyways, we end up calling [`drupal_build_form()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/drupal_build_form/7), which is a workhorse of a function. It does a lot of things, so let's take them one step at a time.

## Set some defaults

The first thing that `drupal_build_form()` does is adds some default values to the `$form_state` array, like so:

```php
$form_state += form_state_defaults();
```

Looking at [`form_state_defaults()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/form_state_defaults/7) shows us this: 

```php
function form_state_defaults() {
  return array(
    'rebuild' => FALSE,
    'rebuild_info' => array(),
    'redirect' => NULL,
    'build_info' => array(
      'args' => array(),
      'files' => array(),
    ),
    'temporary' => array(),
    'submitted' => FALSE,
    'executed' => FALSE,
    'programmed' => FALSE,
    'programmed_bypass_access_check' => TRUE,
    'cache' => FALSE,
    'method' => 'post',
    'groups' => array(),
    'buttons' => array(),
  );
}
```

Any given `$form_state` should generally have all of those keys included, although the values for them may be different, since we are talking about "defaults" after all.

## Fetch the form from cache if possible

The second important function of `drupal_build_form()` is to try to grab the form from the forms cache, if it exists there.

```php
$check_cache = isset($form_state ['input']['form_id']) && $form_state ['input']['form_id'] == $form_id && !empty($form_state ['input']['form_build_id']);
if ($check_cache) {
    $form = form_get_cache($form_state ['input']['form_build_id'], $form_state);
}
```

Note that it only tries to grab a cached form if `$form_state['input']['form_build_id']` is not empty. Otherwise, we already know that this is a freshly built form and we can't rely on the cache.

So what happens in [`form_get_cache()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/form_get_cache/7)? Let's find out.

First, it tries to fetch a cached item with a `cid` (cache ID) that matches ``form_` . $form_build_id` from the `cache_form` database table. This will hopefully return a fully built form array, if one has been cached.

```php
if ($cached = cache_get('form_' . $form_build_id, 'cache_form')) {
  $form = $cached->data;
    
  ...
}
```

Then, it makes sure that the `$form` it found in that step has a valid `#cache_token`, or if a cache token doesn't exist, it makes sure the user is logged out. The reasoning here is that unauthenticated users aren't getting user-specific info in their form, so in that case, a cache token isn't necessarily needed.

If we pass that check, then we can try to grab the form state from the cache, by looking for a `cid` matching `'form_state_' . $form_build_id`. Note that this differs from the previous `cache_get()` in that we're looking for `form_state_` here, not `form_`. In other words, we're trying to see if there are previously entered values that we need to prepopulate into the form.

If so, meaning we *do* have previously entered values to stick into the form, we just populate the `$form_state` variable with them. Since that variable was passed into `form_get_cache()` by reference, then that's all we have to do.

And that's it. At the end of it all, we just return `$form` if it was set back at the beginning. Otherwise, we return nothing. Then, back in [`drupal_build_form()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/drupal_build_form/7), all we have left to do is process the form using `drupal_process_form()` (more on that in a bit), and return it.

## If not cached, build the form from scratch

Assuming the cache didn't give us anything to work with, we need to piece it together ourselves. Still inside [`drupal_build_form()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/drupal_build_form/7), we use two very important functions to do that.

```php
$form = drupal_retrieve_form($form_id, $form_state);
drupal_prepare_form($form_id, $form, $form_state);
```

### Retrieve the form

The [`drupal_retrieve_form()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/drupal_retrieve_form/7) is basically a big fancy guy that, at the end of the day, just calls the original form function to populate the `$form` array..

Before it can do that, it has some details and housekeeping to attend to:

- If the form function is a menu callback, and it's in an include file (not a `.module` file), then we manually set `$form_state ['build_info']['files']['menu']` to the filename of the include file, so that if we load the form on a different path (like `system/ajax` for processing AJAX requests), we can still find it.
- If there's no function with a name matching the value of `$form_id`, then it runs implementations of [`hook_forms`](https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_forms/7) to try to find a matching form builder function for that `$form_id`.
- If `$form_state['wrapper_callback']` was defined, and the function matching that value exists, then it calls that function first, and then adds anything returned by it to `$args` which is passed into the form builder function.

Once all of that is done, we can finally do what we came here to do. We call the form builder function and pass `$args` into it:

```php
$form = call_user_func_array(isset($callback) ? $callback : $form_id, $args);
$form ['#form_id'] = $form_id;
return $form;
```

In that snippet, `$callback` will only be set if there is no function called `$form_id`, and it had to resort to `hook_forms()` to find the callback, as mentioned above.

Now, we have a fully built `$form` array that we can return for preparation.

### Prepare the form

We now have `$form`, which is a giant array of (mostly) fields and their properties. Now we need to prepare it a bit, which means a lot of things:

#### Set some required elements

We need the following to exist, so we set them here if they're not already there:

- `$form['form_build_id']` is set to hidden element with a value of a randomly generated key
- `$form['form_token']` is set to a special `token` type element, if the current user is authenticated, to protect against [cross site request forgery](https://en.wikipedia.org/wiki/Cross-site_request_forgery).
- `$form['form_id']` is set to a hidden element which contains the form ID, obviously
- `$form['#id']` is set as a CSS friendly version of `$form_id`, by running it through [`drupal_html_id()`](https://api.drupal.org/api/drupal/includes%21common.inc/function/drupal_html_id/7)

#### Set up a form validation callback

The source of truth for "how to validate this form" lives in `$form['#validate']`, which is an array of validation functions. But that may need to be created. 

We do that here by first checking to see if there's a function whose name matches `$form_id . '_validate'` and if so, add that to the `$form['#validate']` array.

Otherwise, if this is a shared form, we can grab the `base_form_id` for it and see if a `_validate` function exists for that.

#### Set up a form submission callback

Here, we do exactly the same thing we just did for validation, except with `_submit` functions and `$form['#submit']`.

#### Apply theme suggestions

If `$form['#theme']` isn't set, we just make it array with one value: the `$form_id`.

If this happens to be a shared form, we add the base form ID to the array as well.

#### Allow form alterations

It's time to run the famous `hook_form_alter()` hook implementations.

```php
$hooks = array('form');
if (isset($form_state ['build_info']['base_form_id'])) {
  $hooks [] = 'form_' . $form_state ['build_info']['base_form_id'];
}
$hooks [] = 'form_' . $form_id;
drupal_alter($hooks, $form, $form_state, $form_id);
```

So we start with [`hook_form_alter()`](https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_form_alter/7), then we add on [`hook_form_BASE_FORM_ID_alter()`](https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_form_BASE_FORM_ID_alter/7), and finally we add [`hook_form_FORM_ID_alter()`](https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_form_FORM_ID_alter/7), in that order. 

Note that order matters here, because this gives us a way to control priority. We can use `hook_form_FORM_ID_alter()` if we want to make sure we have the last say.

## Process the form

Whether we grabbed the form from cache or built it ourselves, we need to process it once we have it. 

```php
drupal_process_form($form_id, $form, $form_state);
```

The [`drupal_process_form()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/drupal_process_form/7) function is really the heart of the form API. This is where the form gets assembled, validated, and submitted.

Let's take each of those 3 phases one by one.

### Build the form elements

Yes, I know this is confusing, because we've already "built" the form by calling its form builder function to grab the form array. What's happening here is different. The `drupal_process_form()` function calls [`form_builder()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/form_builder/7). This recursive function cycles through the form tree from top to bottom, and for each element, does the following:

#### Populate `#value` with user input

https://api.drupal.org/api/drupal/includes%21form.inc/function/_form_builder_handle_input_element/7

#### Call element `#process` functions

Elements can contain `#process` callback functions, which get called after user input has been added to `#value`, but before `form_builder()` processes the element it's working on. This allows for code to dynamically add child elements, set additional properties, or implement special logic.

It's important to note that `#process` calls are made in "preorder traversal", meaning they are called for the parents before their children.

If a form element has a `#process` function defined, then it gets called here, like so: 

```php
if (isset($element ['#process']) && !$element ['#processed']) {
  foreach ($element ['#process'] as $process) {
    $element = $process($element, $form_state, $form_state ['complete form']);
  }
  $element ['#processed'] = TRUE;
}
```

#### Recursively build child elements

Now that `#process`	has run for an element, we're ready to process that element, and that means doing a bunch of things, such as:

- Denying access to child elements if the parent has `#access` set to FALSE
- Inherit `#disabled` and `#allow_focus` from parent elements
- Assign a decimal placeholder weight to child elements to preserve order
- A few other less important things

And then finally, and most importantly, we end up running `form_builder()` on the current element. This is obviously a recursive function call, since we're already inside of `form_builder()`, so this is how child elements get built and processed.

#### Call element `#after_build` functions

Any defined `#after_build` callbacks are run after `form_builder()` is done processing the element it's working on.

In contract with `#process`, `#after_build` calls are made in "postorder traversal", meaning they're called for the child elements first, then the parent elements.

```php
if (isset($element ['#after_build']) && !isset($element ['#after_build_done'])) {
  foreach ($element ['#after_build'] as $function) {
    $element = $function($element, $form_state);
  }
  $element ['#after_build_done'] = TRUE;
}
```

### Add validation and submission handlers
### Deal with multistep forms
### Cache form and form state if possible
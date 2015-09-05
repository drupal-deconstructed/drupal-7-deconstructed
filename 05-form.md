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

For each element of the form, if the element is an input (as opposed to a hidden or markup element), then we need to populate it with a `#value` if one exists. For example, if we're reloading the form with validation errors, we need to prepopulate the previously entered values in the process.

```php
// Handle input elements.
if (!empty($element ['#input'])) {
  _form_builder_handle_input_element($form_id, $element, $form_state);
}
```

The [`_form_builder_handle_input_element()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/_form_builder_handle_input_element/7) function is long (due to the comments) and somewhat scary looking, but its main purposes are assigning a `#name` to nested fields, and assigning a `#value` if one exists in `$form_state['input']`. 

It also does a few other things, such as setting `$form_state['triggering_element']` which tells us which submit button was clicked if the form was submitted, and running any defined `#value_callback` properties for dynamically setting values.

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

#### Set the triggering element

It's useful to know which button was clicked to submit the form, as this button can carry custom submission or validation handlers.

```php
if (!$form_state['programmed'] && !isset($form_state['triggering_element']) && !empty($form_state['buttons'])) {
  $form_state['triggering_element'] = $form_state['buttons'][0];
}
```

Pretty simple. The only gotcha is that it makes sure that the form wasn't submitted programmatically via the [`drupal_form_submit()`](http://api.drupal.org/api/search/7/drupal_form_submit) function (which sets `$form_state['programmed']` to `TRUE`), because if it was, then there would have been no button clicked at all.

#### Look for validate/submit handlers on the triggering element

If a button was clicked to submit the form (i.e., the "triggering element"), and that button had custom `#validate` or `#submit` properties, then those need to be used instead of the form-level ones. To do this, `form_builder()` runs this little snippet:

```php
foreach (array('validate', 'submit') as $type) {
  if (isset($form_state['triggering_element']['#' . $type])) {
    $form_state[$type . '_handlers'] = $form_state['triggering_element']['#' . $type];
  }
}
```

Later on, when we're validating the form, we'll check for that.

And that wraps things up for `form_builder()`. Again, it handles a few other details as well, but that's the gist.

### Validate the form input (if it exists)

Back in [`drupal_process_form()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/drupal_process_form/7), it's time to validate the form, if there's a submission to validate. 

```php
if ($form_state ['process_input']) {
  drupal_validate_form($form_id, $form, $form_state);
  
  // HANDLING SUBMISSIONS AND OTHER STUFF GOES HERE (SEE BELOW)
}
```

Where does `$form_state['process_input']` get set? That happens back in the `form_builder()` function, if either `$form_state['input']` or `$form_state['programmed']` are non-empty. In other words, if there is user input, or we're submitting the form programmatically, then we set `$form_state['process_input']` to TRUE, which flags it for validation and submission.

Let's see what actually happens in [`drupal_validate_form()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/drupal_validate_form/7). Turns out, that function calls [`_form_validate()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/_form_validate/7) which is where the heavy lifting happens. 

The `_form_validate()` function is another recursive form function which calls itself on each child element of the given element. So if we're starting with the form itself, then `_form_validate()` will call `_form_validate()` for each of the form's children, which will call `_form_validate()` for each of those children's children, and so on until it has run on everything.

But what actually happens there?

#### Validate `#maxlength`

First of all, it validates the `#maxlength` property. 

```php
if (isset($elements ['#maxlength']) && drupal_strlen($elements ['#value']) > $elements ['#maxlength']) {
  form_error(...);
}
```

#### Validate that `#value` exists in `#options`

Next, it validates that if `#options` exists, then `#value` is one of the possible options. The code for this is a bit too long and detailed to be posted as a snippet here, but the idea is simple. For select fields, checkboxes, and radio buttons, it just verifies that the user isn't trying to be sneaky and submit a value that wasn't an option to begin with.

#### Validate `#required`

After that, we need to make sure that if `#required` is `TRUE` on the given element, then a valid `#value` exists for it. 

This is a little tricky since `#value` can have a few different formats. It can be a string, or an array, or an integer. So we need to support each of those. The fact that an unchecked checkbox has a value of 0 (the integer), which means it would fail validation, combined with the fact that a text input could have a value of "0" (the string), which means it would pass validation since "0" is not an empty string, makes it especially tricky.

```php
if (isset($elements ['#needs_validation']) && $elements ['#required']) {
  $is_empty_multiple = (!count($elements ['#value']));
  $is_empty_string = (is_string($elements ['#value']) && drupal_strlen(trim($elements ['#value'])) == 0);
  $is_empty_value = ($elements ['#value'] === 0);
  if ($is_empty_multiple || $is_empty_string || $is_empty_value) {
    if (isset($elements ['#title'])) {
      form_error($elements, $t('!name field is required.', array('!name' => $elements ['#title'])));
    }
    else {
      form_error($elements);
    }
  }
}
```

See that? It just has a separate check for each of the 3 possible formats. Not so bad.

#### Execute user-defined validation handlers

We've reached the part where custom validation callbacks run.

```php
if (isset($form_id)) {
  form_execute_handlers('validate', $elements, $form_state);
}
```

That simple call to [`form_execute_handlers()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/form_execute_handlers/7) tells Drupal to run any custom validation callbacks. These are defined as things `$form_state['validate_handlers']` array if populated (this is described in the "Look for validate/submit handlers on the triggering element" section earlier in this chapter), otherwise it checks the `form['#validate']` array.

Assuming it finds some (or at least one) validation callbacks, it cycles through them and runs them:

```php
foreach ($handlers as $function) {
  $function($form, $form_state);
}
```

Simple enough. Then the callbacks themselves have the ability to call `form_set_error()` as needed to flag validation failures.

#### Execute element-specific validation handlers

In addition to form-level validation, individual elements can also provide their own validation callbacks using the [`#element_validate`](https://api.drupal.org/api/drupal/developer!topics!forms_api_reference.html/7#element_validate) property, which should an array of validation functions for the given element.

These are run like so:

```php
elseif (isset($elements ['#element_validate'])) {
  foreach ($elements ['#element_validate'] as $function) {
    $function($elements, $form_state, $form_state ['complete form']);
  }
}
```

And with that, we have reached the end of the `_form_validate()` function.

#### Account for `#limit_validation_errors`

We're back in `drupal_validate_form()` now, and all validation has had a chance to run at this point. 

But before moving onto either submitting the form if it validated or reloading it with errors if not, we have one more step. We need to check the [`#limit_validation_errors`](https://api.drupal.org/api/drupal/developer!topics!forms_api_reference.html/7#limit_validation_errors) property on the triggering element (i.e., the button used to submit the form).

This property can be set on form buttons to tell Drupal to ignore any failed validation on elements that aren't specifically listed. This is most commonly used for things like AJAX calls that aren't meant to validate and submit the entire form, such as "Add More" for adding new empty inputs to a form without submitting it.

The code for this is a bit more detailed than we need to get here. All it's really doing is removing any non-validated form values from `$form_state['values']` so that only values that passed validation are left around for the submit callbacks. This way, we can reach submit callbacks without throwing any unwanted errors, without the possibility of submitting data that doesn't validate in the process.

#### What does `form_set_error()` do?

Before we move onto submission, it's worth looking at how errors actually get set. When a validation function calls [`form_set_error()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/form_set_error/7) to flag a validation failure, what happens?

The `form_set_error()` function keeps a static cache of form errors for the current form that it confusingly calls `$form`.

```php
$form = &drupal_static(__FUNCTION__, array());
```

Note that this `$form` is NOT the actual `$form` that's being validated. It's just an array of error messages, keyed by the name of the elements that contain them. When called, it sets an error like so:

```php
$form [$name] = $message;
if ($message) {
  drupal_set_message($message, 'error');
}
```

In that snippet, `$form` is the static variable that holds validation messages, `$name` is the name of the element, and `$message` is the error message being set. Note that there's a little more to `form_set_error()` than that because it has to account for children elements which have strange names like `foo][bar][baz` as opposed to just `baz`, but those are just details.

So that's all well and good, but how can we use this to decide if the form passed or failed validation? Well, the [`form_get_errors()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/form_get_errors/7) function is used for that. This simple function just calls `form_set_error()` with no arguments to retrieve the static variable that holds validation messages, and returns them if they exist, otherwise it doesn't return at all.

So before submitting the form, we can just ensure that `form_get_errors()` returns nothing to be sure that we have passed validation.

### Submit the form input (if it exists)

Back in [`drupal_process_form()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/drupal_process_form/7), it's time to submit the form. 

```php
if ($form_state ['submitted'] && !form_get_errors() && !$form_state ['rebuild']) {
  form_execute_handlers('submit', $form, $form_state);
```

That's really all there is to it. We've already talked about `form_get_errors()` and `form_execute_handlers()` above. All we're doing here is running the submit handlers defined for the form in the `#submit` property of the triggering element if set, or in `$form['#submit']` if not, just like we did for `#validate`.

After actually submitting the form by running the submit callbacks as above, there are a few other things to do.

#### Clear out the form cache

Now that the form has been submitted, we have no more use for its session-specific caches, so we can clear them out. 

```php
if (!variable_get('cache', 0) && !empty($form_state ['values']['form_build_id'])) {
  cache_clear_all('form_' . $form_state ['values']['form_build_id'], 'cache_form');
  cache_clear_all('form_state_' . $form_state ['values']['form_build_id'], 'cache_form');
}
```

Note that the check for `!empty($form_state ['values']['form_build_id'])` makes sure that we only clear out the form cache built specifically for the current user.

#### Process batches defined during submission

The Form API has built in support for batch operations. Submit handlers can call `batch_set()` to set up batch operations, for long running things that might time out if handled all in a single request.

Any defined batch operations are gathered up using `batch_get()` and processed here. The innards of how the batch system works are discussed in a later chapter, so I won't go into details here.

#### Redirect the form

The [`drupal_redirect_form()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/drupal_redirect_form/7) function is called at this point.

This function handles redirecting after a valid form submission is completed. It kicks the user to whatever page is defined in `$form_state['redirect']`, if one is at all:

```php
if (isset($form_state ['redirect'])) {
  if (is_array($form_state ['redirect'])) {
    call_user_func_array('drupal_goto', $form_state ['redirect']);
  }
  else {
    drupal_goto($form_state ['redirect']);
  }
}
```

Note that this supports passing an array into `$form_state['redirect']` instead of just a string path. Doing so will pass the array pieces into `drupal_goto()` as arguments, which means that you can pass along options, which end up being passed to the `url()` function.

And finally, if we haven't redirected yet, meaning `$form_state['redirect']` wasn't defined, then we just reload the current page.

```php
drupal_goto(current_path(), array('query' => drupal_get_query_parameters()));
```

And just like that, our submission is done, and off we go.

### Cache form and form state if possible

With everything else done, and assuming we haven't redirected away from the form after a submission, we can cache the form if appropriate. Here's the code:

```php
if (!$form_state ['rebuild'] && $form_state ['cache'] && empty($form_state ['no_cache'])) {
  form_set_cache($form ['#build_id'], $unprocessed_form, $form_state);
}
```

Nothing to it! Note that we only cache if we haven't specifically told the form to rebuild or to avoid caching.

## Rendering the form

I know what you're thinking. How does the form actually get displayed? How does this giant array get converted to a set of `<input>` and `<form>` and `<select>` tags? 

Well, the end result of all of this form building work is a [render array](https://www.drupal.org/node/930760). Converting render arrays to markup to be displayed to end users is whole new ball of wax, and we'll talk about that in the Render chapter.
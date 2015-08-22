# The Form System

**Note: this chapter is not complete!**

Ah yes. The form system. Who doesn't love the Drupal 7 form system, right? Right?

The typical Drupal developer can probably tell you how to build a form and what the big array of fields should look like, but it's very doubtful that that developer has any idea how Drupal takes that big array and converts it into an actual HTML form.

Fear not! By the end of this chapter you will be that developer no longer!

## A quick summary

![Form system call stack](http://i.imgur.com/lhcS4LX.png)

Hope that helps!

## Fetching the form function

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

Anyways, we end up calling [`drupal_build_form()`](https://api.drupal.org/api/drupal/includes%21form.inc/function/drupal_build_form/7), which is a workhorse of a function. 
# Module Support

The major difference between Drupal and most other CMSes is its modular
construction. Almost every aspect of the system is exposed in such a
way that developers can extend it or override by implementing
particular hooks in their own modules. They can also create hooks in
their module so that other developers can further extend the system.

Aside from extensibility, a second advantage is that it is possible to
disable unneeded modules to reduce the amount of code that is executed,
which improves Drupal's performance and reduces its exposure to attacks.

## How modules get registered
Before we can use the functionality provided by a module, we must first
download it, install it, and enable it. The act of downloading the
module, whether one uses drush dl, FTPs the files, or uses the Update
module (don't do that), is simply the act of placing the files in a
directory within your Drupal installation where Drupal will know to scan
for the code. Prior to this point, all of the functionality of Drupal we
have looked at has been in non-optional include files. However, even
within core there is functionality provided by 40 separate modules, many
of which can be disabled.

*Need to look at the minimal install profile to figure out what the
absolute bare minimum is*

Installing a module is the act of enabling it for the first time. During
install, the system may need to create database tables for the module or
run other install hooks. When a module is enabled, we check to see if it
has dependencies which must be met, install its schema and update the
cache, invoke the install hooks (if this is our first installation or a
reinstallation) and then invoke hook\_modules\_installed() and
hook\_modules\_enabled().

Let's look at the code:
```
function module_enable($module_list, $enable_dependencies = TRUE) {
  if ($enable_dependencies) {
    // Get all module data so we can find dependencies and sort.
    $module_data = system_rebuild_module_data();
    // Create an associative array with weights as values.
    $module_list = array_flip(array_values($module_list));
```
$enable_dependencies is true by default. Most of the situations in which
it is overridden are in tests or when running other installs which can
be trusted to manage their own dependencies.

system_rebuild_module_data() is itself implemented in a module
(system.module) to be precise.

## The module lifecycle
All of the functions that define the module subsystem live in
[module.inc](https://api.drupal.org/api/drupal/includes!module.inc/7).
You may remember from chapter 2 that this file is first included during the DRUPAL\_BOOTSTRAP\_VARIABLES
phase of the bootstrap process.

Modules are loaded by the [module\_load\_all](module_load_all) function.
This function takes a single boolean parameter, $bootstrap. If it is
true, only the bootstrap modules are loaded (those modules that
implement hook\_boot, hook\_exit, hook\_watchdog, or
hook\_language\_init). During DRUPAL\_BOOTSTRAP\_VARIABLES, it is called
with a value of true.

The rest of the modules are not loaded until DRUPAL\_BOOTSTRAP\_FULL.

The function itself is very simple:
```
function module_load_all($bootstrap = FALSE) {
  static $has_run = FALSE;

  if (isset($bootstrap)) {
    foreach (module_list(TRUE, $bootstrap) as $module) {
      drupal_load('module', $module);
    }
    // $has_run will be TRUE if $bootstrap is FALSE.
    $has_run = !$bootstrap;
  }
  return $has_run;
}
```
The isset test on bootstrap is because the theme include specifically
calls module_load_all with NULL. *Need to figure out why that is*

module_list returns the list of enabled modules. If bootstrap is TRUE,
it will only return the bootstrap modules.

module_list() is a fairly thin wrapper around system_list().

## Responding to hooks

## Exposing hooks of our own

## Module updates

## Disabling a module

## Uninstalling a module


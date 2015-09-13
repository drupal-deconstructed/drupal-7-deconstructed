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
*Talk about module_enable and module_disable here*

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


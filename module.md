# Module Support

The major difference between Drupal and most other CMSes is its modular
construction. Almost every aspect of the system is exposed in such a
way that developers can extend it or override by implementing
particular hooks in their own modules. They can also create hooks in
their module so that other developers can further extend the system.

Aside from extensibility, a second advantage is that it is possible to
disable unneeded modules to reduce the amount of code that is executed,
which improves Drupal's performance and reduces its exposure to attacks.

## The underlying database
All of the module information is stored in the system table.
```
+----------------+--------------+------+-----+---------+-------+
| Field          | Type         | Null | Key | Default | Extra |
+----------------+--------------+------+-----+---------+-------+
| filename       | varchar(255) | NO   | PRI |         |       |
| name           | varchar(255) | NO   |     |         |       |
| type           | varchar(255) | NO   | MUL |         |       |
| owner          | varchar(255) | NO   |     |         |       |
| status         | int(11)      | NO   |     | 0       |       |
| throttle       | tinyint(4)   | NO   |     | 0       |       |
| bootstrap      | int(11)      | NO   |     | 0       |       |
| schema_version | smallint(6)  | NO   |     | -1      |       |
| weight         | int(11)      | NO   |     | 0       |       |
| info           | text         | YES  |     | NULL    |       |
+----------------+--------------+------+-----+---------+-------+
```


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

[system_rebuild_module_data()](https://api.drupal.org/api/drupal/modules!system!system.module/function/system_rebuild_module_data/7) is itself implemented in a module
(system.module to be precise). It's a short function, so let's look at it in its entirety:
```
function system_rebuild_module_data() {
  $modules_cache = &drupal_static(__FUNCTION__);
  // Only rebuild once per request. $modules and $modules_cache cannot be
  // combined into one variable, because the $modules_cache variable is reset by
  // reference from system_list_reset() during the rebuild.
  if (!isset($modules_cache)) {
    $modules = _system_rebuild_module_data();
    ksort($modules);
    system_get_files_database($modules, 'module');
    system_update_files_database($modules, 'module');
    $modules = _module_build_dependencies($modules);
    $modules_cache = $modules;
  }
  return $modules_cache;
}
```
Rebuilding the cache is a relatively expensive operation, so we first
check to see if we already have it cached. If not, we call
_system_rebuild_module_data() which does all the heavy lifting.

It's too long to reproduce, but the general gist of what it does is:
1. Scan the modules directories under the current installation profile
   and under sites/all to find every file that ends with a ".module" 
   extension.
2. If the directory it is in also has a .info file, call
   drupal_parse_info_file() on it. That function is basically just a
   cache wrapper around drupal_parse_info_format(), which is largely
   an ugly regular expression that turns a .info file into a nested
   array. For details on the .info format itself, see [Writing module
   .info files (Drupal 7.x)](https://www.drupal.org/node/542202)

*TODO: Pick back up at the point where we include the profile itself
as a module.*


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

module_list() is a fairly thin wrapper around [system_list()](https://api.drupal.org/api/drupal/includes!module.inc/function/system_list/7). system_list() can return either:
1. All enabled modules
2. All bootstrap modules
3. All themes

The value for the cache bucket keyed by this function's name is an
array. There are separate sub-lists of the cache for each of the
scenarios listed above.

## Responding to hooks

## Exposing hooks of our own

## Module updates

## Disabling a module

## Uninstalling a module

## Odds and ends to work in somewhere
- One option in the .info file that you may not be familiar with is
  *required*. If set to TRUE, the module cannot be disabled. This should
  only be set in core modules. At the time of this writing, the
  following modules are always required: field_sql_storage, field, text,
  filter, node, system, and user.

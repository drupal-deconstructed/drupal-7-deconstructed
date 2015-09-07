# The Database System

**Note: This chapter isn't complete yet!**

Brace yourselves, this one's a doozy. Drupal's database system is probably one of the least understood parts. While a decent Drupal developer might have a loose idea of how the menu router works or what happens during the bootstrap process, nobody seems to have any idea of how Drupal talks to the database.

Lucky for us, it's not actually that bad. Promise!

## Some background information

Before we jump into the code, let's take a quick step back.

If we're trying to explain how Drupal talks to databases, we first need to define the major building blocks in general terms. Here are the three big pieces that need to exist before we have a useful system.

- The connection. How is a connection to the database first established?
- The query. How is a query built and sent to the database?
- The result. How are query results read and understood?

Thankfully, [PDO](http://php.net/manual/en/intro.pdo.php) ("PHP Data Objects") exists for PHP, which has stock solutions for each of those three.

- The `PDO` class handles creating, maintaining, and closing the connection between PHP and whatever database you're using. 
- The `PDOStatement` class handles querying the database (using "prepared statements" which is where the name comes from) as well as returning results.

That's it. Those are the two major pieces of PDO. There is also a `PDOException` class and some drivers for different databases, but those are just details.

Congratulations, you know enough about PDO to understand what Drupal is doing. In fact, a lot of PDO will look familiar to you if you've done some querying with Drupal. For example, `fetchAll()` and `rowCount()` are core functions of the `PDOStatement` class.

## A (not so) quick summary

With that out of the way, let's talk about what Drupal's doing. This one's a bit tough to summarize, because at a high level it's extremely simple, and at a low level, it's pretty dang complex.

The bird's eye view is that Drupal's database system is not much more than some extensions on top of PDO. Drupal has some classes which extend from the core PDO classes to add support for advanced features (table prefixing is the most obvious example) and fancy helper functions, but at the end of the day it's all just dressing on top of PDO.

Let's start with a quick rundown of the major classes we'll see. Note that this isn't an exhaustive list of all DB-related classes in Drupal, but it's enough to get you the big picture.

Here are the big three, all of which live along with a few others in `includes/database/database.inc`:

* [`DatabaseConnection`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/class/DatabaseConnection/7) - extends the `PDO` class to manage the connection
* [`DatabaseStatementBase`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/class/DatabaseStatementBase/7) - extends the `PDOStatement` class to send queries and fetch results
* [`Database`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/class/Database/7) - a standalone class that isn't meant to be instantiated or extended, which contains some functions used to manage the connection with `DatabaseConnection` without having to resort to global variables (this will make more sense later).

And then we have the classes for specific query types, which live in `includes/database/query.inc`:

* [`Query`](https://api.drupal.org/api/drupal/includes%21database%21query.inc/class/Query/7) - base class that doesn't do much on its own
* [`InsertQuery`](https://api.drupal.org/api/drupal/includes%21database%21query.inc/class/InsertQuery/7) - extends the `Query` class for `INSERT` queries
* [`UpdateQuery`](https://api.drupal.org/api/drupal/includes%21database%21query.inc/class/UpdateQuery/7) - extends the `Query` class for `UPDATE` queries
* [`DeleteQuery`](https://api.drupal.org/api/drupal/includes%21database%21query.inc/class/DeleteQuery/7) - extends the `Query` class for `DELETE` queries
* [`MergeQuery`](https://api.drupal.org/api/drupal/includes%21database%21query.inc/class/MergeQuery/7) - extends the `Query` class for `MERGE` queries
* [`TruncateQuery`](https://api.drupal.org/api/drupal/includes%21database%21query.inc/class/TruncateQuery/7) - extends the `Query` class for `TRUNCATE` queries
* [`SelectQuery`](https://api.drupal.org/api/drupal/includes%21database%21select.inc/class/SelectQuery/7) - extends the `Query` class for `SELECT` queries. Note that this one is special enough to exist in its own file (`includes/database/select.inc`) with its own `interface` to go along with it.

And to close the loop, since this is supposed to be a summary after all, here's the general process (at a VERY high level, even though it may not seem like it):

1. A query function such as [`db_select()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/db_select/7) is called when a query is being requested.
2. That function will call the [`getConnection()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/Database%3A%3AgetConnection/7) method of the `Database` class which will fetch the DB driver and connection info.
3. Once we have the connection info, we can instantiate the driver-specific DB class, such as [`DatabaseConnection_mysql`](https://api.drupal.org/api/drupal/includes%21database%21mysql%21database.inc/class/DatabaseConnection_mysql/7) and pass in that info.
4. Our `DatabaseConnection_mysql` DB class's constructor will call its parent (i.e., `DatabaseConnection`) constructor which calls its own parent (i.e., `PDO`) constructor, which creates and returns the connection
5. Back in the `db_select()` function with a connection in hand, we can use chaining to instantiate the `SelectQuery` class on top of it and build the query object
6. The [`execute()`](https://api.drupal.org/api/drupal/includes%21database%21select.inc/function/SelectQuery%3A%3Aexecute/7) method of the `SelectQuery` class runs, which ends up running the [`query()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseConnection%3A%3Aquery/7) method of the `DatabaseConnection` class
7. That ends up running the [`execute()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseStatementBase%3A%3Aexecute/7) method of the `DatabaseStatementBase` class
8. Finally, that ends up running the `execute()` method if its parent class, which is `PDOStatement`, which actually does run the query against the target database.
9. Our query has run, and we can fetch results using, for example, `fetchAll()` (which comes straight from `PDOStatement` or [`fetchAllAssoc()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseStatementBase%3A%3AfetchAllAssoc/7) (which is a nicety provided by `DatabaseStatementBase`).

Too detailed? Ok fine, here it is at an EVEN HIGHER level:

1. A query function like `db_select()` runs.
2. It creates a connection (thanks `PDO`!).
3. Then it creates a query object.
4. Then it executes the query (thanks again `PDO!`).
5. Then it fetches results however you ask it to.

That's about as helpful a summary as we're going to get without diving into the details a bit. Speaking of details...

## The lifecycle of a query

Rather than going function by function on the hundreds of database-related functions, I think it's more helpful (or perhaps less boring) to trace the path of a query from start to finish.

### We kick off a query

For George's request to `about-us`, a bunch of queries are going to run, and the vast majority of them are going to be `SELECT` queries, so we'll use one of those as our example. Let's see what happens when this runs (assuming `$node` is the node currently being viewed).

```php
$result = db_select('node', 'n')
  ->fields('n')
  ->condition('nid', $node->nid, '=')
  ->execute()
  ->fetchAssoc();
```

Pretty basic query, right? Let's walk through it, line by line, just to get our bearings. Bear with me if you're a veteran; we'll get there.

`$result = db_select('node', 'n')`

We're calling `db_select()` on the `node` table, and aliasing it to `n`.

`->fields('n')`

We're telling it that instead of fetching specific fields in the `node` table, we want all of them.

`->condition('nid', $node->nid, '=')`

We're saying we want to filter out any results where the `nid` column isn't equal to `$node->nid`.

`->execute()`

We're done building the query, and we're ready to run it.

`->fetchAssoc();`

We want the results to be formatted into an associative array. And we're done!

So you can see that even this very basic query has a lot of information baked in. Specifically, we told it:

1. What kind of query we want to run
2. What our base table is
3. What fields we want to retrive from it
4. What conditions we want to include
5. How we want the results formatted

And all of that in one convenient little chained method call!

### A connection is established 

Let's dig in! Starting at the beginning, we're calling [`db_select`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/db_select/7), which is a very simple function.

```php
function db_select($table, $alias = NULL, array $options = array()) {
  if (empty($options['target'])) {
    $options['target'] = 'default';
  }
  return Database::getConnection($options['target'])->select($table, $alias, $options);
}
```

So basically, use the target database unless otherwise specified, then grab a connection using `getConnection()`, and run `select()` on it.

First things first, we need to connect to a database before we can do anything useful. Notice that Drupal doesn't just open a connection for the heck of it during the database bootstrap. No connections are opened until they're actually needed for a query. Here's what happens when we're ready for that.

Firstly, each possible query function (in our case, `db_select()`, but just as true for `db_update()`, `db_insert()`, etc.) includes a call to `Database::getConnection()` before calling the actual query. We're calling `db_select()` so we get this:

```php
return Database::getConnection($options['target'])->select($table, $alias, $options);
```

If you dive into that [`Database::getConnection`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/Database%3A%3AgetConnection/7) call, it'll take you to [`Database::openConnection()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/Database%3A%3AopenConnection/7). 

But wait, there's more! If you dive into that `openConnection()` call, you're taken to [`Database::parseConnectionInfo()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/Database%3A%3AparseConnectionInfo/7). This very important function populates the `Database::$databaseInfo` variable with the information from the `global $databases` variable. 

You may (or may not) remember that that global variable was created in the `DRUPAL_BOOTSTRAP_CONFIGURATION` phase of the bootstrap process, courtesy of the [`drupal_settings_initialize()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_settings_initialize/7) function.

Anyways, as a result of the `Database::openConnection()` function, we have a fully populated `$databaseInfo` variable, and we pass that into the constructor for the driver class we're using. 

```php
if (!$driver = self::$databaseInfo[$key][$target]['driver']) {
  throw new DatabaseDriverNotSpecifiedException('Driver not specified for this database connection: ' . $key);
}

$driver_class = 'DatabaseConnection_' . $driver;
require_once DRUPAL_ROOT . '/includes/database/' . $driver . '/database.inc';
$new_connection = new $driver_class(self::$databaseInfo[$key][$target]);
```

From here, the driver class that we've just instantiated will do some driver-specific stuff (more to come on that later) along with calling its parent class's constructor. Its parent class happens to be `DatabaseConnection`, which we've already talked about as being extended from the core PHP `PDO` class, which creates our connection when instantiated.

And we have our connection. To summarize: any given query will call `getConnection()` which calls `openConnection()` which, after finding our DB connection info from the `global $databases` variable, calls the constructor for our DB driver's class. That constructor calls `DatabaseConnection`'s constructor which calls `PDO`'s constructor, which creates our connection. 

And with an open connection ripe and ready for the picking, we can move on and do something useful.

### A query object is created

### A condition is added

### The query is executed

### The results are returned

## Schema and structure

`DatabaseSchema` class.

## Database drivers

What do they do? Walk through MySQL? Or possibly SQLite to be simpler?

# TODO:

- Talk about DatabaseStatementPrefetch
- Talk about all the db_* functions in database.inc
- Talk about logging (setLogger and getLogger)
- Talk about table prefixing
- Talk about query errors/exceptions

# The Database System

Uh oh. Drupal's database system is probably one of the least understood parts. While a decent Drupal developer might have a loose idea of how the menu router works or what happens during the bootstrap process, nobody seems to have any idea of how Drupal talks to the database.

Lucky for us, it's not actually that bad. Promise!

## A quick summary

This one's a bit tough to summarize, because at a high level it's extremely simple, and at a low level, it's pretty dang complex.

The bird's eye view is that Drupal's database system is built around classes which extend [PHP's PDO](http://php.net/manual/en/intro.pdo.php) (PHP Data Objects) database API, and the syntax and behavior is actually pretty similar. 

Drupal has some classes which extend from the core PDO classes to add support for advanced features and fancy helper functions, but at the end of the day it's all just dressing on top of PDO.

I think that's about as helpful a summary as I can come up with without getting into the details. Speaking of details...

## Some background information

Before we jump into the code, let's take a quick step back.

If we're trying to explain how Drupal talks to databases, we first need to define the major building blocks in general terms. Here are the three big pieces that need to exist before we have a useful system.

- The connection. How is a connection to the database first established?
- The query. How is a query built and sent to the database?
- The result. How are query results read and understood?

Thankfully, PDO exists for PHP, which has stock solutions for each of those three.

- The `PDO` class handles creating, maintaining, and closing the connection between PHP and whatever database you're using. 
- The `PDOStatement` class handles querying the database (using "prepared statements" which is where the name comes from) as well as returning results.

That's it. Those are the two major pieces of PDO. There is also a `PDOException` class and some drivers for different databases, but those are just details.

Congratulations, you know enough about PDO to understand what Drupal is doing. In fact, a lot of PDO will look familiar to you if you've done some querying with Drupal. For example, `fetchAll()` and `rowCount()` are core functions of the `PDOStatement` class.

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

Pretty basic query, right? Basically, just give me everything from the `node` table for the `nid` of the node we're currently viewing.

The first thing to notice is that we're calling [`db_select`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/db_select/7), which is a very simple function.

```php
function db_select($table, $alias = NULL, array $options = array()) {
  if (empty($options['target'])) {
    $options['target'] = 'default';
  }
  return Database::getConnection($options['target'])->select($table, $alias, $options);
}
```

So basically, use the target database unless otherwise specified, then grab a connection, and run `select()` on it.

Let's start with the `getConnection()` part of it.

### A connection is established 

First things first, we need to connect to a database before we can do anything useful. Notice that Drupal doesn't just open a connection for the heck of it during the database bootstrap. No connections are opened until they're actually needed for a query. Here's what happens when we're ready for that.

Firstly, each possible query function (in our case, `db_select()`, but just as true for `db_update()`, `db_insert()`, etc.) includes a call to `Database::getConnection()` before calling the actual query. We're calling `db_select()` so we get this:

```php
return Database::getConnection($options['target'])->select($table, $alias, $options);
```

If you dive into that [`Database::getConnection`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/Database%3A%3AgetConnection/7) call, it'll take you to [`Database::openConnection()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/Database%3A%3AopenConnection/7). 

But wait, there's more! If you dive into that `openConnection()` call, you're taken to [`Database::parseConnectionInfo()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/Database%3A%3AparseConnectionInfo/7). This very important function populates the `Database::$databaseInfo` variable with the information from the `global $databases` variable. 

You may (or may not) remember that that global variable was created in the `DRUPAL_BOOTSTRAP_CONFIGURATION` phase of the bootstrap process, courtesy of the [`drupal_settings_initialize`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_settings_initialize/7) function.

Anyways, as a result of the `Database::openConnection()` function, we have a fully populated `$databaseInfo` variable, and we pass that into the constructor for the driver class we're using. 

```php
if (!$driver = self::$databaseInfo[$key][$target]['driver']) {
  throw new DatabaseDriverNotSpecifiedException('Driver not specified for this database connection: ' . $key);
}

$driver_class = 'DatabaseConnection_' . $driver;
require_once DRUPAL_ROOT . '/includes/database/' . $driver . '/database.inc';
$new_connection = new $driver_class(self::$databaseInfo[$key][$target]);
```

From here, the driver class that we've specified will do some driver-specific stuff along with calling its parent class's constructor. Its parent class happens to be `DatabaseConnection`, which we've already talked about, and we know that `DatabaseConnection`'s constructor calls its own parent's constructor, which is `PDO`, which initiates the connection.

And we have our connection. To summarize: any given query will call `getConnection()` which calls `openConnection()` which, after finding our DB connection info from the `global $databases` variable, calls the constructor for our DB driver's class. That constructor calls `DatabaseConnection`'s constructor which calls `PDO`'s constructor, which creates our connection.

### A query object is created

### A condition is added

### The query is executed

### The results are returned

# NOTES: IGNORE

Here are some temporary notes that may or may not be used when the chapter is finalized.

### The `DatabaseConnection` constructor

The class's constructor (i.e., its [`__construct`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseConnection%3A%3A__construct/7) function) is thankfully very simple. Besides just calling `PDO`'s `__construct` function, it adds support for table prefixes:

```php
$this->setPrefix(isset($this->connectionOptions['prefix']) ? $this->connectionOptions['prefix'] : '');
```

If you're especially astute, you'll notice that `$this->connectionOptions` isn't actually set yet. That's because `$this->connectionOptions` is meant to be overridden by driver-specific classes. For example, `DatabaseConnection_mysql` and `DatabaseConnection_sqlite` set it.

It also sets the class we want to use for prepared statements, so that we can use Drupal's extended class for querying and fetching results, rather than defaulting to the core PHP `PDOStatement` class.

```php
$this->setAttribute(PDO::ATTR_STATEMENT_CLASS, array($this->statementClass, array($this)));
```

The class we use here will be `DatabaseStatementBase`, because that variable gets set just a few lines before the `__construct` function:
 
```php
protected $statementClass = 'DatabaseStatementBase';
```


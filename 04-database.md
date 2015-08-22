# The Database System

Brace yourselves, this one's a doozy. Drupal's database system is probably one of the least understood parts, and perhaps one of the toughest to understand.

## Some background information

Before we jump into the code, let's take a quick step back.

If we're trying to explain how Drupal talks to databases, we first need to define the major building blocks in general terms. Here are the three big pieces that need to exist before we have a useful database abstraction layer.

- The connection. How is a connection established, maintained, and closed?
- The query. How is a query built and sent to the database?
- The result. How are query results returned in a format which we can use?

Thankfully, [PDO](http://php.net/manual/en/intro.pdo.php) ("PHP Data Objects") exists for PHP, which has stock solutions for each of those three. There are two classes here that we need to be aware of.

- The `PDO` class handles creating, maintaining, and closing **the connection** between PHP and whatever database you're using. 
- The `PDOStatement` class handles **querying the database** (using "prepared statements" which is where the name comes from) as well as **returning results**.

That's it. Those are the two major pieces of PDO, and you can see that each of our three major pieces in the list above above are addressed in one of those two classes.

*Note: there are more `PDO` classes than just those two. There is also a `PDOException` class and some driver classes for different databases, for example, but those aren't important for the purposes of our understanding right now.*

Congratulations, you know enough about PDO to understand how Drupal is using it! In fact, a lot of PDO will look familiar to you if you've done some querying with Drupal. For example, `fetchAll()` and `rowCount()` are core functions of the `PDOStatement` class.

## A (not so) quick summary

With that out of the way, let's talk about what Drupal's doing. This one's a bit tough to summarize, because at a high level it's extremely simple, and at a low level, it's pretty dang complex.

The bird's eye view is that Drupal's database system is not much more than some extensions on top of PDO. Drupal has some classes which extend from the core PDO classes to add support for advanced features (table prefixing is the most obvious example) and fancy helper functions, but at the end of the day it's all just dressing on top of PDO.

Let's start with a quick rundown of the major classes in use. Note that this isn't an exhaustive list of all DB-related classes in Drupal, but it's enough to get you the big picture.

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
* [`SelectQuery`](https://api.drupal.org/api/drupal/includes%21database%21select.inc/class/SelectQuery/7) - extends the `Query` class for `SELECT` queries. *Note that this one is special enough to exist in its own file (`includes/database/select.inc`) with its own `interface` to go along with it.*

And to close the loop, since this is supposed to be a summary after all, here's the general process (at a VERY high level, even though it may not seem like it):

1. A query function such as [`db_select()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/db_select/7) is called when a query is being requested.
2. That function will call the [`getConnection()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/Database%3A%3AgetConnection/7) method of the `Database` class which will fetch the DB driver and connection info from `settings.php`.
3. Once we have the connection info, we can instantiate the database driver class for our chosen database, such as [`DatabaseConnection_mysql`](https://api.drupal.org/api/drupal/includes%21database%21mysql%21database.inc/class/DatabaseConnection_mysql/7), and hand it our connection info.
4. The driver class constructor function will call the class's parent constructor (i.e., `DatabaseConnection`'s constructor) which calls its own parent's constructor (i.e., `PDO`'s constructor), which creates and returns the connection.
5. Back in the `db_select()` function with a connection in hand, we can use function chaining to instantiate the `SelectQuery` class on top of it and build the query object.
6. Any other functions chained after our original `db_select()`, such as `condition()` or `range()` or `orderBy()`, will be functions inside our query class (i.e., `SelectQuery`) and will alter the instantiated query object's attributes.
6. The [`execute()`](https://api.drupal.org/api/drupal/includes%21database%21select.inc/function/SelectQuery%3A%3Aexecute/7) method of the `SelectQuery` class runs, which converts our query object to a `SQL` string, and runs it through [`query()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseConnection%3A%3Aquery/7) method of the `DatabaseConnection` class
7. That ends up running the [`execute()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseStatementBase%3A%3Aexecute/7) method of the `DatabaseStatementBase` class
8. Finally, *that* function runs the `execute()` method if its parent class, which is `PDOStatement`, which actually executes the query against the target database.
9. Our query has run, and we can fetch results using one of many `fetch*()` functions provided by `PDOStatement` and `DatabaseStatementBase`.

To detailed? Ok fine, here it is at an EVEN HIGHER level:

1. A query function like `db_select()` runs.
2. It creates a connection (thanks `PDO`!).
3. Then it creates a query object.
4. Then it executes the query (thanks again `PDO!`).
5. Then it fetches results however you ask it to.

That's about as helpful a summary as we're going to get without diving into the details a bit. Speaking of details...

## The lifecycle of a query

Rather than going function by function on the hundreds of database-related functions, I think it's more helpful (or perhaps less boring) to trace the path of a query from start to finish.

### We kick off a query

For George's request to `about-us`, a bunch of queries are going to run, and the vast majority of them are going to be `SELECT` queries, so we'll use one of those as our example. Let's see what happens when this very typical looking query runs (assuming `$node` is the node currently being viewed).

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

Let's dig in! Starting at the beginning, we're calling [`db_select()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/db_select/7), which is a very simple function.

```php
function db_select($table, $alias = NULL, array $options = array()) {
  if (empty($options['target'])) {
    $options['target'] = 'default';
  }
  return Database::getConnection($options['target'])->select($table, $alias, $options);
}
```

So basically, use the target database unless otherwise specified, then grab a connection using [`getConnection`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/Database%3A%3AgetConnection/7), and run [`select()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseConnection%3A%3Aselect/7) on it.

First things first, we need to connect to a database before we can do anything useful. Notice that Drupal doesn't just open a connection for the heck of it during the database bootstrap. No connections are opened until they're actually needed for a query. Here's what happens when we're ready for that.

Firstly, each possible query function (in our case, `db_select()`, but just as true for `db_update()`, `db_insert()`, etc.) includes a call to the `getConnection()` function of the `Database` class before calling the actual query. We're calling `db_select()` so we get this (copied from above):

```php
return Database::getConnection($options['target'])->select($table, $alias, $options);
```

If you dive into that [`Database::getConnection`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/Database%3A%3AgetConnection/7) call, it'll take you to [`Database::openConnection()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/Database%3A%3AopenConnection/7). 

But wait, there's more! If you dive into that `openConnection()` call, you're taken to [`Database::parseConnectionInfo()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/Database%3A%3AparseConnectionInfo/7). This very important function populates the `$databaseInfo` variable with the information from the `global $databases` variable. 

You may or may not remember that that global variable was created in the `DRUPAL_BOOTSTRAP_CONFIGURATION` phase of the bootstrap process, courtesy of the [`drupal_settings_initialize()`](https://api.drupal.org/api/drupal/includes%21bootstrap.inc/function/drupal_settings_initialize/7) function.

Anyways, as a result of the `Database::openConnection()` function, we have a fully populated `$databaseInfo` variable, and we pass that into the constructor for the driver class we're using. 

```php
if (!$driver = self::$databaseInfo[$key][$target]['driver']) {
  throw new DatabaseDriverNotSpecifiedException('Driver not specified for this database connection: ' . $key);
}

$driver_class = 'DatabaseConnection_' . $driver;
require_once DRUPAL_ROOT . '/includes/database/' . $driver . '/database.inc';
$new_connection = new $driver_class(self::$databaseInfo[$key][$target]);
```

All we're really doing here is grabbing the driver name from `$databaseInfo`, and using that to try to instantiate a class for it, which should be named `DatabaseConnection_<driver>` and should be located in '/includes/database/<driver>/database.inc'. If found, we pass in the database info so that the driver class can make use of it.

From here, the driver class that we've just instantiated will do some driver-specific stuff (whatever is needed to prepare a connection for whatever database driver you're using) along with calling its parent class's constructor. Its parent class happens to be `DatabaseConnection`, which we've already talked about as being extended from the core PHP `PDO` class, which creates our connection when instantiated.

And we have our connection. To summarize: any given query will call `getConnection()` which calls `openConnection()` which, after finding our DB connection info from the `global $databases` variable, calls the constructor for our DB driver's class. That constructor calls `DatabaseConnection`'s constructor which calls `PDO`'s constructor, which creates our connection. 

And with an open connection ripe and ready for the picking, we can move on and do something useful.

### A query object is created

Remember that this is the line of code we're looking at, from the `db_select()` function.

```php
return Database::getConnection($options['target'])->select($table, $alias, $options);
```

Now that `getConnection()` is complete and has given us a DB connection to use, we can move on to the second part of that statement.

The [`select()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseConnection%3A%3Aselect/7) function is also very simple:

```php
$class = $this->getDriverClass('SelectQuery', array('query.inc', 'select.inc'));
return new $class($table, $alias, $this, $options);
```

This just gives the DB drivers a chance to override the default query classes to add or alter their behavior so that they play nice with the database in use. For example, PostgreSQL has to have its own version of SelectQuery to change the way `orderBy()` and `orderRandom()` behave.

All this is doing behind the scenes is checking to see if there is a driver-specific `SelectQuery` class, and returning that if so, otherwise it just returns the default. To do this, it just looks for a class named `SelectQuery_<drivername>` such as `SelectQuery_mysql`. In our case, it won't find one, because `SelectQuery_mysql` doesn't exist. But if it were an `INSERT` query, it *would* find `InsertQuery_mysql`, which does exist. Or if we were using SQLite, then it would find `SelectQuery_sqlite`, which also exists.

Query objects, such as our `SelectQuery`, are really complicated little things. They extend the `Query` class but they don't really get much from it. Most of the logic belongs to the individual query classes themselves. Our `SelectQuery` has functions for all of the things you might want to do to it, such as:

- `SelectQuery::having()`
- `SelectQuery::isNotNull()`
- `SelectQuery::where()`
- Anything else you might chain onto a call to `db_select()`

All of the things you can chain onto `db_select()` functions to write complicated queries end up running functions defined in the `SelectQuery` class. And all of those functions basically just alter our current instantiated `SelectQuery` object by updating its attributes.

I know this all seems very vague and confusing. As a specific example, let's see what happens when we walk through our original query, given what we know about Query objects.

```php
$result = db_select('node', 'n')
  ->fields('n')
  ->condition('nid', $node->nid, '=')
  ->execute()
  ->fetchAssoc();
```

Remember that the first line creates the connection and the `SelectQuery` object. We've already covered that.

The second line calls the [`SelectQuery::fields()`](https://api.drupal.org/api/drupal/includes%21database%21select.inc/function/SelectQuery%3A%3Afields/7) (remember, all chained functions are defined in the class for the query object) function which looks like this: 

```php
public function fields($table_alias, array $fields = array()) {

  if ($fields) {
    foreach ($fields as $field) {
      // We don't care what alias was assigned.
      $this->addField($table_alias, $field);
    }
  }
  else {
    // We want all fields from this table.
    $this->tables[$table_alias]['all_fields'] = TRUE;
  }

  return $this;
}
```

See what I mean about just altering the `SelectQuery` object? It either runs [`SelectQuery::addField()`](https://api.drupal.org/api/drupal/includes%21database%21select.inc/function/SelectQuery%3A%3AaddField/7) on each field if there are specific fields listed, or it sets `all_fields` to `TRUE` if not. Remember, all of this is just so that by the time we're ready to execute the query, we have a fully built query object that we can convert into a SQL statement. More on that in a bit.

In the meantime, the 3rd line (the one with `condition()` happens to be important enough that it deserves its own section.

### A condition is added

The third line calls `condition()` which is yet another function belonging to `SelectQuery`, but it's a little more involved than most of the others. Here's what it looks like:

```php
public function condition($field, $value = NULL, $operator = NULL) {
  $this->where->condition($field, $value, $operator);
  return $this;
}
```

The tricky part is that `$this->where` is actually an instance of the `DatabaseCondition` class (yes, conditions get their own class - are you overwhelmed yet?). This instantiation happened in the [`__construct()`](https://api.drupal.org/api/drupal/includes%21database%21select.inc/function/SelectQuery%3A%3A__construct/7) function of `SelectQuery`, like so:

```php
$this->where = new DatabaseCondition('AND');
```

As a result, `$this->where->conditions()` becomes an array of conditions in the following format:

```php
array(
  'field' => $field,
  'value' => $value,
  'operator' => $operator,
);
```

Then, at execution time, that gets compiled along with the rest of the query into raw SQL.

So, by adding a condition to our query using the chained `condition()` function, what it's really doing is calling the `condition()` function of the `DatabaseCondition` class, which adds an array to the list of conditions on our query object.

### The query is executed

Don't forget the original code that kicked all of this off:

```php
$result = db_select('node', 'n')
  ->fields('n')
  ->condition('nid', $node->nid, '=')
  ->execute()
  ->fetchAssoc();
```

We're done with those first 3 lines, so now it's time to run `execute()`, which is yet another function of the `SelectQuery` class. It looks like this:

```php
public function execute() {
  // If validation fails, simply return NULL.
  // Note that validation routines in preExecute() may throw exceptions instead.
  if (!$this->preExecute()) {
    return NULL;
  }

  $args = $this->getArguments();
  return $this->connection->query((string) $this, $args, $this->queryOptions);
}
```

The [`SelectQuery::preExecute()`](https://api.drupal.org/api/drupal/includes%21database%21select.inc/function/SelectQuery%3A%3ApreExecute/7) function does a few very important things:

- If the query has any tags added by the [`addTag()`](https://api.drupal.org/api/drupal/includes%21database%21select.inc/function/SelectQuery%3A%3AaddTag/7) function (which we don't in our example) then it runs the alter hook(s) specified by those tags.
- It also runs `preExecute()` on any defined subqueries
- Finally, it runs `preExecute()` on any defined unions

With `preExecute()` complete, we move onto the [`getArguments()`](https://api.drupal.org/api/drupal/includes%21database%21select.inc/function/SelectQuery%3A%3AgetArguments/7). This is the part where `WHERE` and `HAVING` clauses are compiled from objects to string SQL syntax, along with any `UNION`s or subqueries. 

And finally, we run the dang query! Note that that last line, the one that runs `query()`, casts `$this` (i.e., the query object) to a string, which by default PHP behavior means that it runs its `__toString()` function.

Lucky for us, `SelectQuery` has a great big [`__toString()`](https://api.drupal.org/api/drupal/includes%21database%21select.inc/function/SelectQuery%3A%3A__toString/7) function which converts that hefty object into a regular old prepared statement, which looks a lot more like the raw SQL that our database expects. That `__toString()` function doesn't really do anything magical - it's just a matter of asking our query object if it has this or that, and adding the appropriate SQL to the string. For example, here's how `GROUP BY` clauses are added:

```php
// GROUP BY
if ($this->group) {
  $query .= "\nGROUP BY " . implode(', ', $this->group);
}
```

See? Not so bad.

At the end of `__toString()` we have a fully built SQL string which can be passed on to run the query, by calling the [`query()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseConnection%3A%3Aquery/7) method of the `DatabaseConnection` class.

This is the code from that function that ends up doing the magic: 

```php
$this->expandArguments($query, $args);
$stmt = $this->prepareQuery($query);
$stmt->execute($args, $options);
```

That first [`expandArguments()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseConnection%3A%3AexpandArguments/7) line is mostly just housekeeping. For any arguments that are arrays, such as if we had something like `->condition('nid', $nids_array)`, it will convert the array to a comma delimited list.

The second line, the one that runs [`prepareQuery()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseConnection%3A%3AprepareQuery/7) isn't very fun either. It calls the [`prefixTables()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseConnection%3A%3AprefixTables/7) which literally just runs a `str_replace` to add in table prefixes if you're using them. If not, it just removes the curly brackets around table names in the query and moves on.

And we have arrived at the last line. Remember that we're working with the `DatabaseConnection` class which extends from the core `PDO` class. This last line just calls the `execute()` function of the core `PDO` class, which takes over and runs the query against our database.

### The results are returned

Our query has run, folks! We're ready to see what it returned! It's been a long journey but we're not done yet!

The next bit of code from the [`DatabaseConnection::query()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseConnection%3A%3Aquery/7) function is important:

```php
switch ($options['return']) {
  case Database::RETURN_STATEMENT:
    return $stmt;
  case Database::RETURN_AFFECTED:
    return $stmt->rowCount();
  case Database::RETURN_INSERT_ID:
    return $this->lastInsertId();
  case Database::RETURN_NULL:
    return;
  default:
    throw new PDOException('Invalid return directive: ' . $options['return']);
}
```

This is our Drupal decides what exactly to return. 

- `SELECT` queries just return the results object as given to us by PDO. 
- `UPDATE`, `DELETE`, `MERGE`, and `TRUNCATE` queries return the count of affected rows
- `INSERT` queries return the ID of the last inserted item
- You'll almost never see `RETURN_NULL` set - it happens as an edge case for `INSERT` queries running against PostgreSQL or if manually specified in the code that runs the query.

Let's take a peek back at our original code:

```php
$result = db_select('node', 'n')
  ->fields('n')
  ->condition('nid', $node->nid, '=')
  ->execute()
  ->fetchAssoc();
```

All of the stuff we just talked about happens as part of `execute()`. Now that that's all complete, we can fetch the results in whatever format we want them. 

We used `fetchAssoc()`, but we could have easily used any of the other methods defined in our [`DatabaseStatementBase`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/class/DatabaseStatementBase/7) class (which, remember, extends `PDOStatement`), such as:

- [`fetchAssoc()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseStatementBase%3A%3AfetchAssoc/7) - returns the next row as an associative array
- [`fetchAllAssoc()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseStatementBase%3A%3AfetchAllAssoc/7) - returns the entire result set as an associative array keyed by the given field
- [`fetchAllKeyed()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseStatementBase%3A%3AfetchAllKeyed/7) - returns a single associative array (only useful for two-column result sets)
- [`fetchCol()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseStatementBase%3A%3AfetchCol/7) - returns an entire single column of the result set as an indexed array
- [`fetchField()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/DatabaseStatementBase%3A%3AfetchField/7) - returns a single field from the next record of a result set

Or, we can also choose from these, which come straight from the `PDOStatement` class, so Drupal can use them out of the box:

- [`fetch()`](http://php.net/manual/en/pdostatement.fetch.php) - returns the next row from the result set
- [`fetchObject()`](http://php.net/manual/en/pdostatement.fetchobject.php) - returns the next row as an object
- [`fetchAll()`](http://php.net/manual/en/pdostatement.fetchall.php) - returns an array containing all rows of a result set
- [`fetchColumn()`](http://php.net/manual/en/pdostatement.fetchcolumn.php) - returns a single column (you probably should never use this since Drupal's `fetchField()` exists which calls this behind the scenes)

There's not much to say about how these `fetch*` functions do what they do. The implementation of the `PDO` functions is outside of the scope of this book, and the ones that exist in Drupal basically each just call the `PDO` functions with a certain set of parameters, or a little bit of processing.

As an example of "a little bit of processing", the `fetchAllAssoc()` function just looks like this:

```php
foreach ($this as $record) {
  $record_key = is_object($record) ? $record->$key : $record [$key];
  $return [$record_key] = $record;
}

return $return;
```

And as an example of "just calling a `PDO` function", we can look at what `fetchCol()` does internally:

```php
return $this->fetchAll(PDO::FETCH_COLUMN, $index);
```

See? Nothing very special there. Thankfully, this is one part of Drupal's database layer that isn't very tough to grasp.

And thus we have queried the database and fetched our results, and we're ready to do something useful with them. The database layer's work is done, for that particular query.

## Database schema

All that is well and good, but how does the database table structure get built in the first place? How does Drupal know which tables and fields and indexes to create? 

For those who may not be aware, this is a good time to note that modules primarily define their table structure inside a `hook_schema()` implementation in `modulename.install`. Let's walk through exactly how that gets called on a module (in our case, we'll say the `node` module) and what Drupal does with it.

When a module is enabled, the [`module_enable`](https://api.drupal.org/api/drupal/includes%21module.inc/function/module_enable/7) runs this little chunk of code (which I have removed comments from):

```php
if (drupal_get_installed_schema_version($module, TRUE) == SCHEMA_UNINSTALLED) {
  drupal_install_schema($module);
  $versions = drupal_get_schema_versions($module);
  $version = $versions ? max($versions) : SCHEMA_INSTALLED;
  drupal_set_installed_schema_version($module, $version);
  module_invoke($module, 'install');
}
```

In plain English, this code checks to see if the module has ever been installed before, and if not (meaning the `schema_version` column in the `system` table for this module is equal to `SCHEMA_UNINSTALLED` which is `-1`), it installs the schema, then saves the current schema version to the system table.

*Note that the above code only runs if the module has never been installed. If it was previously installed, then disabled, and is being re-enabled again, that code does not run at all. Therefore, any schema changes which need to be made after the module was previously installed will have to happen in `hook_update_N` and will run the next time database updates are applied.*

Obviously, the magic happens in [`drupal_install_schema($module)`](https://api.drupal.org/api/drupal/includes%21common.inc/function/drupal_install_schema/7). Let's take a look at what's going on there.

```php
function drupal_install_schema($module) {
  $schema = drupal_get_schema_unprocessed($module);
  _drupal_schema_initialize($schema, $module, FALSE);

  foreach ($schema as $name => $table) {
    db_create_table($name, $table);
  }
}
```

Let's take it line by line. 

First, we run [`drupal_get_schema_unprocessed($module)`](https://api.drupal.org/api/drupal/includes%21common.inc/function/drupal_get_schema_unprocessed/7). This extremely simple function literally just loads the module's install file, invokes its `hook_schema()` implementation, and returns that result, without really doing any other processing on it. Not bad.

This gives us `$schema`, which is an array of tables, which themselves are arrays of properties.

Next, we run [`_drupal_schema_initialize()`](https://api.drupal.org/api/drupal/includes%21common.inc/function/_drupal_schema_initialize/7) which is also fairly simple. It just loops through the `$schema` array and makes sure that each table has `$table['module']` and `$table['name']` set, and sets it if needed.

And finally, we just loop through the `$schema` array and run [`db_create_table()`](https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/db_create_table/7) on each table inside it. This ends up running [`DatabaseSchema::createTable()`](https://api.drupal.org/api/drupal/includes%21database%21schema.inc/function/DatabaseSchema%3A%3AcreateTable/7) on the table, which looks like this:

```php
public function createTable($name, $table) {
  if ($this->tableExists($name)) {
    throw new DatabaseSchemaObjectExistsException(t('Table @name already exists.', array('@name' => $name)));
  }
  $statements = $this->createTableSql($name, $table);
  foreach ($statements as $statement) {
    $this->connection->query($statement);
  }
}
```

Basically, we just check to make sure the table doesn't already exist, then create it if so.

The `createTableSql()` is probably the only interesting part of that snippet. It's a driver-specific function, meaning there is a separate version of it depending on which database driver you're using. Here's [the MySQL version](https://api.drupal.org/api/drupal/includes!database!mysql!schema.inc/function/DatabaseSchema_mysql%3A%3AcreateTableSql/7) of it, if you'd like to take a peek. 

No matter which driver you're using, however, the idea is the same. We use the `createTableSql()` function to create the raw SQL for the `CREATE TABLE` statement based on the table array, and then run the SQL to. The function creates a string for the `CREATE TABLE` statement, then loops through each of the defined fields and calls `createFieldSql()` (another driver-specific function) on each of them, adding the result to the SQL string. At the end of the function, we should have a fully built `CREATE TABLE` statement complete with all defined fields and their options, ready to run.

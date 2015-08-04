# Drupal 7 Deconstructed: Part 4 - The Database

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

- The `PDO` class handles the connection between PHP and the database.
- The `PDOStatement` class handles querying the database as well as returning results.

That's it. Those are the two major pieces of PDO. There is also a `PDOException` class and some drivers for different databases, but those are just details.

Congratulations, you know enough about PDO to understand what Drupal is doing.

## Level one: the database connection

The `DatabaseConnection` class extends the core `PDO` class.

## Level two: prepared statements

The `DatabaseStatementBase` class extends the core `PDOStatement` class. It adds some nice helpful functions that we know and love such as `fetchAllAssoc()` and `fetchField()`.

## Level three: database drivers

## Level four: query types

### InsertQuery

### SelectQuery

### UpdateQuery

### DeleteQuery

### MergeQuery

### TruncateQuery

## Level five: query conditions
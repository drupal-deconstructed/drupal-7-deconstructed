# Performance
*Under construction*

Drupal runs some of the largest and busiest sites on the internet,
including [weather.com](http://weather.com). Despite that, Drupal does
not have a great reputation for performance. Understanding the root
causes will help you to understand why some patterns occur in the Drupal
codebase and assist you in coming up with an architecture and a
configuration that will help you maximize Drupal's performance.
Essentially there are two things that make performance a challenge in
Drupal

1. The repository for almost everything is a relational database
2. PHP has a "share nothing" architecture. 

If you have ever turned on query logging in mysql and hit Drupal, you
will notice that a single page view generates A LOT of queries. I just
cleared cache and hit the home page of a Lightning demo site I had setup
and needed 794 queries to render the page. It's not uncommon for some of
these queries to be more complex than is ideal for performance.
Generally speaking, this is a consequence of trying to expose a lot of
functionality to site builders with maximum flexibility.

The share nothing architecture of PHP means that each PHP process
maintains all of its own data. This architecture greatly simplifies
learning PHP and makes it easy to build simple applications.
Unfortunately, it creates performance challenges and makes it tough to
get good performance on complex applications (like Drupal). It is why
every single Drupal request has to bootstrap Drupal (in contrast to an
architecture like Java or .NET where much of that could be done once and
stored in application scoped variables). 

The general pattern solution for this in PHP is to persist data to the
filesystem or the database. For a common, non-Drupal-specific example,
consider storing session data. By default, it goes to the filesystem and
PHP makes it available to every PHP process as necessary. By default,
Drupal caches things in the database. When you look at your Drupal
database, you'll see many tables whose name starts with "cache\_". On my
Lightning install, there are 22, but the exact set will differ
slightly depending on which modules you have installed.

Unfortunately, what this means is that even in scenarios where
everything is cached, there is still a lot of interaction with the
database.

##Page caching in Drupal


##Data caching in Drupal


=======
Coming soon...

>>>>>>> master:16-performance.md
*todo: talk about drupal_static()*
Need to describe the "advanced usage pattern" for drupal\_static. This
[blog post](http://jmuras.com/blog/index.html%3Fp=537.html) is the best
description I've found for that.

*todo: talk about drupal_static_reset and cache_clear_all*

##Improving views performance
*todo: Add this after we write the views chapter - or maybe put it
there*

##Alternative cache backends
*talk about memcache*

##Reverse proxies
*talk about varnish. Make this short because this is inherently not an
internal - it's about avoiding Drupal's internals*
=======

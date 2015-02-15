---
layout: post
title:  "MySQL and efficient URL lookups"
date:   2015-02-17 00:47:41
categories: application-architecture
---

There comes a time where it's necesssary to store url's in their full length format in a database. Whether this is for a url shortening service, some kind of bespoke analytics and the like.

When performing lookups on this data, which uses a standard B-Tree index to catalog locations of everything, the index can become huge, because URLs are usually quite long. You would normally query a table of URLs like this:

{% highlight sql %}
mysql> SELECT id FROM url WHERE url="http://www.myreallylongurl.com"
{% endhighlight %}

This is fine when the dataset is small, but when the dataset gets large, querying like this is incredibly inefficient. The InnoDB engine has a special feature called *adaptive hash indexes*. When InnoDB notices that some index values are being accessed frequently, it builds a hash index for the in memory on top of the usual B-Tree index. This gives the B-Tree properties of a hash index, such as very fast hashed lookups. This process is completely automatic, and you have zero control over it.

If you aren't using InnoDB as a storage engine, you can simply build your own hash index. This will give you access to some of the properties of a hash index, without ever being a real hash index. This could by done by removing any index on a url column and adding a column that contains a hash of the URL value, with an index on it.

### Hashes to the rescue!

To begin with this, I'll create a table with mysql, that has a primary key, url varchar column and a url_hash column.
 
{% highlight sql %}
   CREATE TABLE urls (
     id int unsigned NOT NULL auto_increment,
     url varchar(255) NOT NULL,
     url_hash int unsigned NOT NULL DEFAULT 0,
     PRIMARY KEY(id)
   );
{% endhighlight %}

Next, we need some way of creating a hash value for each url as it is inserted/updated. We can do this with triggers in a nice way.

{% highlight sql %}
DELIMITER //
CREATE TRIGGER urls_hash BEFORE INSERT ON urls FOR EACH ROW BEGIN
SET NEW.url_hash=crc32(NEW.url);
END;
//
CREATE TRIGGER urls_hash BEFORE UPDATE ON urls FOR EACH ROW BEGIN
SET NEW.url_hash=crc32(NEW.url);
END;
//
DELIMITER ;
{% endhighlight %}

You should use CRC32() here and not SHA1() or MD5() hash functions here. These return very long strings, which result in wasted space and slower comparisons. These are cryptographically strong functions designed to eliminate collisions.

If your table has many rows and CRC32() gives too many collisions, implement your own 64-bit hash function. Make sure you use a function that returns an integer, not a string. One way to implement a 64-bit hash function is to use just part of the value returned by MD5() like below.

{% highlight sql %}
mysql> SELECT CONV(RIGHT(MD5('http://www.mylongurl.com/'), 16), 16, 10) AS HASH64;
{% endhighlight %}

### Handling collisions

As stated above, CRC32() is not meant to be collision proof, so when querying the database with this method, it's necessary to also include the literal value in the `WHERE` clause, like thus:

{% highlight sql %}
mysql> SELECT id FROM url WHERE url_crc=CRC32("http://www.mylongurl.com") AND url="http://www.mylongurl.com";
{% endhighlight %}

This will ensure you avoid problems with collisions and it is wise to assume that there will be. The chance of collision is about 1% in a database with 90,000 rows. This is called the [Birthday Paradox][birthday].

[birthday]: http://betterexplained.com/articles/understanding-the-birthday-paradox







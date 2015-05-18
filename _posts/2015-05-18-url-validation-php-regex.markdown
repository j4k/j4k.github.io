---
layout: post
title:  "URL Validation in PHP for regex."
date:   2015-05-18 20:19:11
categories: php regex
---

# URL Validation

Processing URLs for validity according to the [living specification](https://url.spec.whatwg.org/#url-parsing) in PHP is non trivial. There are lots of things to consider, and lots of different things can be considered a url. `http://➡.ws/䨹'` is a valid domain according to the spec for instance, as is `http://223.255.255.254` and `http://userid:password@example.com:8080/`.

### DNS Lookups

Using built in methods like `checkdnsrr()` are fine for checking to see if a domain has any DNS entries. It returns a boolean, and can be passed a parameter to look up a certain type of record. This gives a yes/no answer about whether the url exists in one form or another on the internet. Checking for any records usually gives the best results.

{% highlight php %}
<?php
            // check for any type of record
            if (checkdnsrr($url, 'ANY')) {
                // the record exists!
            }
{% endhighlight %}

This works fine for standard ASCII URLs, but `checkdnsrr` fails when passed a URL like `ñandu.cl`. If you need to do lots of validation with urls that contain unicode characters in with `checkdnsrr`, then it's possible, but you need to install the pecl packages `intl` and `idn`.  With these packages, it's possible to convert the domain name to IDNA Ascii form before using `checkdnsrr`. ([IDNA is a mechanism defined for handling internationalized domain names containing non-ASCII characters.](http://en.wikipedia.org/wiki/Internationalized_domain_name))

{% highlight php %}
<?php
    // check for any type of record
    if (checkdnsrr(idn_to_ascii($url), 'ANY')) {
            // the record exists!
    }
{% endhighlight %}

This method has the obvious drawback that if the domain you are trying to validate doesn't have any DNS entries, it will appear invalid, though this may not be the case. Doing this kind of non-vital network stuff on user input runs the risk of a delayed response while this blocks the flow of execution, also. So it may not always be the best choice, depending on the use case.

### Regular Expressions

** Disclaimer: these Regex patterns are PCRE, you may need to adjust them somewhat if you aren't using PHP. *

Another way to validate a domain is to validate it using Regular Expressions. This doesn't ensure that the domain actually exists, but for a quick check that the structure of a URL is valid rather than it's presence on the internet, they often suffice.

If you need something that just validates western urls for form validation - something like this may do:

{% highlight php %}
<?php
      $regex = "#((https?|ftp)://(\S*?\.\S*?))([\s)\[\]{},;"\':<]|\.\s|$)#i";
      $result = preg_match($regex, 'http://google.co.uk'); // true
{% endhighlight %}

This is quite a greedy regex, however, and it matches incorrect urls like `http://.www.foo.bar/` and `http://1.1.1.1.1`.

I will step through this bit by bit to give you an idea of what is going on here. We start the regex string with a `#` and there is another near the end. This is called the delimiter. When using PCRE functions in PHP, it is required that you use a delimiter inside a regex string to show where the regular expression starts and ends. Any characters you see outside of the delimiter are indications to the regular expression parser to toggle certain modes. `i` for instance, let's the parser know that this regular expression is case insensitive. `u` enables unicode support.

This will validate a basic http/https/ftp url. It does not however, take into account things like IPs, internal IPs, or non ASCII urls. For those kind of edge cases, you need something a bit more robust, which invariably means the regular expression gets a bit more complicated.

### Excluding phrases in Regex.

{% highlight php %}
<?php
        $regex = "#^" .
        // protocol identifier
        "(?:(?:https?|ftp):\\/\\/)?" .
        // user:pass authentication
        "(?:\\S+(?::\\S*)?@)?" .
        "(?:" .
        // IP address exclusion
        // private & local networks
        "(?!(?:10|127)(?:\\.\\d{1,3}){3})" .
        "(?!(?:169\\.254|192\\.168)(?:\\.\\d{1,3}){2})" .
        "(?!172\\.(?:1[6-9]|2\\d|3[0-1])(?:\\.\\d{1,3}){2})" .
        // IP address dotted notation octets
        // excludes loopback network 0.0.0.0
        // excludes reserved space >= 224.0.0.0
        // excludes network & broacast addresses
        // (first & last IP address of each class)
        "(?:[1-9]\\d?|1\\d\\d|2[01]\\d|22[0-3])" .
        "(?:\\.(?:1?\\d{1,2}|2[0-4]\\d|25[0-5])){2}" .
        "(?:\\.(?:[1-9]\\d?|1\\d\\d|2[0-4]\\d|25[0-4]))" .
        "|" .
        // host name
        "(?:(?:[a-z\\x{00a1}-\\x{ffff}0-9]-*)*[a-z\\x{00a1}-\\x{ffff}0-9]+)" .
        // domain name
        "(?:\\.(?:[a-z\\x{00a1}-\\x{ffff}0-9]-*)*[a-z\\x{00a1}-\\x{ffff}0-9]+)*" .
        // TLD identifier
        "(?:\\.(?:[a-z\\x{00a1}-\\x{ffff}]{2,}))" .
        ")" .
        // port number
        "(?::\\d{2,5})?" .
        // resource path
        "(?:\\/\\S*)?" .
        "$#ui"; // unicode enabled + case insensitive
{% endhighlight %}

This is not exhaustive, for example - it will match anything as a top level domain. Something you could do to validate for all known current TLD's would be to look at IANA's [list](http://data.iana.org/TLD/tlds-alpha-by-domain.txt) of TLDs and use that to generate the TLD identifier part of this regular expression.

{% highlight php %}
<?php
    $urls = file_get_contents('http://data.iana.org/TLD/tlds-alpha-by-domain.txt');
    $regex = "#^" .
    // ...
    // TLD identifier
    "(?:\\.(?:[".implode('|', explode('\n', $urls))."]{2,}))";
    // ...
{% endhighlight %}

### Working with Regex

Working with regex is all about recognising what the symbols mean. To help you figure out what a regular expression is doing, you should copy it into something like [regex101](http://regex101.com) or [regexr](http://regexr.com), which will give you a break down of what the various parts of the regular expression denote.


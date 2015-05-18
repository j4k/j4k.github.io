---
layout: post
title:  "URL Validation in PHP with regex"
date:   2015-05-18 20:19:11
categories: php regex
---

# URL Validation

Processing URLs for validity according to the [living specification](https://url.spec.whatwg.org/#url-parsing) in PHP is non trivial. There are lots of things to consider, and lots of different things can be considered a url. `http://➡.ws/䨹'` is a valid domain according to the spec for instance, as is `http://223.255.255.254` and `http://userid:password@example.com:8080/`. This article is going to briefly go over two ways that I validate domains depending on my needs, runs over basic regular expressions, and gives a more complex example of a more robust regex that handles most cases in the living specification (though some cases fall through!). If any regular expressions wizards notice anything I could have done better here - send me a [tweet](http://twitter.com/jackw411).

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

Another way to validate a domain is to validate it using Regular Expressions. This doesn't ensure that the domain actually exists, but for a quick check that the structure of a URL is valid rather than it's presence on the internet, they often suffice.

If you need something that just validates western urls for form validation - something like this may do:

{% highlight php %}
<?php
      $regex = "#((?:https?|ftp)://(?:\S*?\.\S*?)(?:[\s)\[\]{},;"\':<]|\.\s|$))#i";
      $result = preg_match($regex, 'http://google.co.uk'); // true
{% endhighlight %}

This is quite a greedy regex, however, and it matches incorrect urls like `http://.www.foo.bar/` and `http://1.1.1.1.1`, but for a rough validation step to ensure a url seems roughly correct, then this might do the trick. Be sure to follow up with more stringent checks on the domain via DNS lookup, curl or some kind of request library.

I will step through this bit by bit to give you an idea of what is going on here. We start the regex string with a `#` and there is another near the end. This is called the delimiter. When using PCRE functions in PHP, it is required that you use a delimiter inside a regex string to show where the regular expression starts and ends. Any characters you see outside of the delimiter are indications to the regular expression parser to toggle certain modes. `i` for instance, let's the parser know that this regular expression is case insensitive. `u` enables unicode support.

This will validate a basic http/https/ftp url. It does not however, take into account things like IPs, internal IPs, or non ASCII urls. For those kind of edge cases, you need something a bit more robust, which invariably means the regular expression gets a bit more complicated.

To extract the domain name using a regular expression like this you would have to change the non-capturing groups we are using to define segments in the regex to capturing groups. Capturing groups allow you define portions of an overall pattern that you want to extract on their own. Non-capturing groups allow you to structure your regular expression, and write sub-patterns within a regular expression easily, without capturing anything from within the group, unless there is a capturing group nested inside.

A non-capturing group is defined as thus: `(?: )`, and a capturing group is defined simply as `()`. Sticking a `?` quantifier behind one of these groups makes these an optional part of the regex. So, in the above regex, we start after the delimiter `#` with the start of a capturing group, which allows us to grab the whole url out of a preg_match as the one and only match. Next we use a non capturing group `(?:https?|ftp)` to say "optionally, check for the presence of the characters http, https or ftp". This is done with the order of the `?` and the pipe character (`|`). The question mark indicates to the parser that the preceeding character should be matched 0 or 1 times, making it an optional character, and the pipe symbol pretty much translates into "or".

After this, the regex tries to match `://` exactly, because it is not inside any character class. After this, we come up against the next non-capturing group `(?:\S*?\.\S*?)`. This one contains something called a metacharacter `\S`. There are a few different metacharacters in regular expressions and they vary from implementation to implementation. This one here, means "match any non-whitespace character". Following this metacharacter up with a `*` quantifier changes the meaning slightly, as the `*` quantifier means "0 or more" so by changing these together we get "match any non-whitespace character 0 or more times". Chaining this again with `?` which me know to mean "optionally", changes the meaning again.. you get the idea.

So without explaining every part of it - the next section matches any non whitespace character 0 or more times optionally, then a `.` then any non whitespace character 0 or more times optionally, inside a non capturing group (that itself sits inside a capturing group!). It's important with regular expressions to be able to verbalise what the symbols mean when they are next to each other like above. Well, maybe not verbalise it, because you might sound mental, but at least have an inner monolog as to what is going on! There is a handy cheat sheet on [duckduckgo.com](https://duckduckgo.com/?q=regex&ia=answer).

As stated before this isn't a bulletproof regular expression. It is very greedy and even gives false positives on trickier invalid domains.

Let's make it a bit more robust.

### Excluding phrases in Regex.

So now that we know how to *include* certain patterns, wouldn't it be handy to exclude certain patterns in our regular expression to adhere to the living standard mentioned before. This can be done with something called a negative assertion. Looking at a [cheatsheet](https://duckduckgo.com/?q=regex&ia=answer) we can see, under assertions, that a Negative lookahead is defined with `?!`. Sticking this inside a capturing group, gives us a negative lookahead for anything within the group. This is pretty powerful, and what the lookahead part means during the comparison of a string and a regular expression is "from whereever we are in the comparison, take a peek ahead and see if the stuff inside this group matches anything coming up, if so, bail". You can see in the regular expression below we use these for a few things.

We check for local networks first and exclude those, followed by private networks. Then in the same group the regex checks if it is actually an IP, and match for IP ranges, and if it finds no match we use the pipe separator to check for a domain name. After either one of these two checks passes, port number and resource path are set.

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
    "(?:\\.(?:".implode('|', explode('\n', $urls))."))";
    // ...
{% endhighlight %}

### Working with Regex

Working with regex is all about recognising what the symbols mean. To help you figure out what a regular expression is doing, you should copy it into something like [regex101](http://regex101.com) or [regexr](http://regexr.com), which will give you a break down of what the various parts of the regular expression denote. For people trying to penetrate regular expressions, there is an excellent primer on Laracasts, thats about 10 minutes long, and will really give you a jump start in the world of regular expressions For people trying to penetrate regular expressions, there is an excellent primer on [Laracasts](http://laracasts.com), thats about 10 minutes long, and will really give you a jump start in the world of regular expressions.

If you have any questions or need a hand, tweet me [@jackw411](http://twitter.com/jackw411), I'm happy to help!

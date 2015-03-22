---
layout: post
title:  "Parsing Concrete5 Areas for Blocks and Stacks"
date:   2015-03-22 20:19:11
categories: concrete5
---


### Parsing Concrete5 Areas for Blocks and Stacks

Recently we were challenged to find an easy way to loop through a page layouts blocks and extract certain ones that contained geolocation information to be drawn onto a Mapbox Map.

This is something we had done before quite easily, but we hit a problem when we needed to descend into stacks in an Area. This isn't something that is well documented, and took a little while of delving into Concrete5's source to solve.

### Parsing Blocks

Parsing blocks in an Area is quite simple.

{% highlight php %}
    public function getBlocks()
    {
        $allBlocks = Page::getCurrentPage()->getBlocks('Main'); // get blocks from `Main` area
        $blocks = array();
        foreach ($allBlocks as $b) {
            if ($b->btHandle == "block_handle_you_want") {
                $block = $b->getInstance();
                // you can now call all the blocks controller methods
                $blocks[] = $block;
            }
        }
        return $blocks;
    }
{% endhighlight %}

Concrete5's Areas are hard coding strings in layout files, so there's no clean way to iterate over all of the current pages areas. Just a heads up if you want to do something like that, store all the Area names in an array as a kind of map and iterate over it.

### Getting concrete5 blocks out of stacks

The trickiest thing about getting blocks out of stacks is the lack of documentation on Concrete5's part. It's pretty easy to do once you know what you are doing. First you start off the same way - get all the blocks within a prescribed area. Stacks have a `btHandle` of `core_stack_display`, so we check for those and add them to an array.

{% highlight php %}
    $stackArray = array();
    $blocks = Page::getCurrentPage()->getBlocks('Main'); // get blocks from area

    foreach ($blocks as $block) {
        if ($block->btHandle == 'core_stack_display') // if is a stack
        {
            $stack = Stack::getByID($block->getInstance()->stID);
            $stackArray[$stackName] = Stack::getByID($block->getInstance()->stID);
        }
    }

    return $stackArray;
{% endhighlight %}


Stack blocks carry an extra attribute called `stID` which is the Stack Identifier. Passing this into the static `Stack::getByID` method gives us an instance of the stack. We are keeping the blocks separate above in a multidimensional array using the stack name, which is returned `snake_case` as a key for each.

Now we have that it's a matter of looping through them as you would blocks...

{% highlight php %}
    $stackBlocks = array();
    foreach ($stackArray as $key => $stack) {
            $blocks = $stack->getBlocks(STACKS_AREA_NAME);
            foreach ($blocks as $b) {
                if ($b->btHandle == "block_type_handle") {
                    $block = $b->getInstance();
                    $stackBlocks[$key] = $block;
                }
            }
    }
    return $stackBlocks;
{% endhighlight %}

In the code snippets above we are checking for a specific type of block, but this could be removed if you were interested in all blocks. This can also be useful in things like Page List's as you are looping through the pages, traversing out and grabbing the first content block on a page and extracting the first paragraph from it.

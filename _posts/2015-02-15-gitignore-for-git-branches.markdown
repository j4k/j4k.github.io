---
layout: post
title:  "Multiple .gitignore for git branches"
date:   2015-02-15 00:47:41
categories: git
---

There comes a time when you might have a Repository that has multiple git remotes, say, one on GitHub for open source sharing, and another for hosting like Heroku, or AWS, and have been faced with the fact that the traditional .gitignore in the root directory lacks configurability and acts like an all or nothing across all of your branches, and tiptoe-ing around adding individual files to the correct branches is a recipe for a headache, and maybe even a security risk. Like everything we do, you are left thinking "there has to be a better way of dealing with this".

Well, there is! What I'm outlining here is the ability to have multiple .gitignores for multiple branches. The use cases are real, such as a node application that you want to build and ship with it's dependencies on deployment, but keep vendor specific stuff off of `master` or `develop` branch, so that it's still easy to track changes without vendor specific stuff gunking up the git history.

#### Why?

Recently when building a small deployment mechanism for a node (express and angular) application, I was faced with either triggering the build process on the server, or, building off server, and deploying the built application with its dependencies as a whole. The second was more feasible, it meant that there would have to be no downtime, and no server resource was spent building the application, purely on serving the application. The downside of this strategy is that building and updating any vendor library meant that my git history was incomprehensible, which made things like `git bisect` really difficult, because it wasn't clear from the history what was being changed when. To remedy this I created a "deploy" branch with it's own .gitignore, that differed to "develop" branch .gitignore.

So let's go!

First, navigate to the project root of the repo that you want to set up multiple .gitignores for. For this, let's assume we have a database config file that is application specific that should not be public.

{% highlight bash %}
$ touch .gitignore
$ git add .gitignore
$ echo "path/to/secret/config.ini" > .gitignore # echo the path of the secret file to gitignore
$ git commit -m "master branch gitignore"
$ git branch public
$ git checkout public
$ touch path/to/secret/config.ini
$ git rm path/to/secret/config.ini
$ echo "path/to/secret/file" > .gitignore
$ git add .gitignore
$ git commit -m 'ignoring and removing important info from public'
{% endhighlight %}

So now we are up to the point where we've done some development, and want to push the results up to our public. We need to checkout public, and then most importantly, we need to merge from master onto our public branch.

Usually this would just result in the private stuff being dumped on the repo, but our strategy here will to be merge master with the "ours" strategy. This is important because it allows us to ignore anything that conflicts with the public repo.

{% highlight bash %}
$ git checkout public
$ git merge -s ours master
{% endhighlight %}

Now that that's done, we have to add remotes for both branches.

{% highlight bash %}
$ git remote add origin
https://github.com/user/public.git
$ git remote add private
https://url_to_private_repo/private.git

$ git push origin -u public
$ git push private -u master
{% endhighlight %}

And voila! Now, when you do your commit and push to the remote public repo nothing private should be seen, but when you deploy with git all those needed passwords are there.

Hope you find this useful!



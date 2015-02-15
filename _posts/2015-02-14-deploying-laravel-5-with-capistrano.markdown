---
layout: post
title:  "Deploying Laravel 5 with Capistrano"
date:   2015-02-14 00:47:41
categories: deployment
---

When deploying applications with Laravel, it can often be beneficial to automate as much as possible. Capistrano is a server automation and deployment tool. It can be used for many things, like deploying an application like we will discuss here across multiple servers, or clearing application cache on just one of our many application servers.

 Why is this beneficial? Can't we just deploy manually by SSH'ing into a server and pulling the repo down onto that server? Sure, we could, but this quickly comes tedious with anything more than a few servers, and isn't a scalable thing. This approach is also prone to errors that can be costly.

### Prerequisites

This article assumes a few things:

- That you have an laravel 5 based application you want to deploy already set up.
- The second thing is that you already have a familiarity with git, github, bitbucket, or gitlab (whichever you are using). It's also worth mentioning that you have an SSH key pair set up for key based authentication with one of these services.
- Lastly, that we are using a Linux based system.

So with all these things set up, let's get started.

### Deployment paradigms

Before talking about any specific methodology, let's brush over common deployment strategies.

#### Push Deployment

With a push model, the local development machine(s) are bundling the application to it's final state, and then pushing that complete package to the specified servers.

#### Pull Deployment

Pull deployment is different in that the local development machines don't move any files. Instead, our local machine tells each server to deploy. In turn, each server then reaches out to a git server, grabs the latest commit from a specified branch, builds/compiles, and makes the new version live.

Here servers are responsible for their own builds, and for grabbing the latest code. We will use a model based on this, but to prevent any downtime and having to trigger a rebuilding of a backend application and any front end build steps on each server, it might be beneficial to consider what we can do here to make things better.

### An aside on environment

Laravel 5 vastly simplifies environment detection. In 4, you could have multiple environment files based on environment name. I never used the environment aspect in 4 much, it seemed a bit convoluted and difficult to manage within a deployment system. I imagine you could have theoretically committed all of your environment files to your repo, but even this seemed a messy approach.

#### Introducing PHP dotenv

This is not the case in Laravel 5, which now ships with the awesome [vlucas/phpdotenv][phpdotenv], a proven library that loads from a single environment file. Laravel 5 now ships with a default .env.example

[phpdotenv]:     https://github.com/vlucas/phpdotenv

{% highlight ini %}
APP_ENV=local
APP_KEY=SomeRandomString
DB_USERNAME=homestead
DB_PASSWORD=homestead
{% endhighlight %}

In order to use this file, just copy it and name the copy .env. Why aren't we renaming the original? You'll see in a second.

Now, you can edit your APP_ENV -- which as you can tell from the default is the primary way for us to set the application environment name. Environment detection in `bootstrap/environment.php` detected an APP_ENV and falls back to production mode if APP_ENV is undefined, allowing us to deploy without these environment configuration files to production.

{% highlight php %}
<?php
$env = $app->detectEnvironment(function()
{
    return getenv('APP_ENV') ?: 'production';
});
{% endhighlight %}

This is absolutely awesome. The reason we've copied the .env.default and not just renamed it, is because this could be used as a boilerplate template with placeholder default values, that could be edited with the correct info on a post-update hook in git or capistrano, if it was required. This can be transported with the application.

### Onto deployment

For the sake of this article, let's assume that we need to set up a staging deployment mechanism, that sends our application to our staging servers.

#### Install Capistrano

Capistrano is ruby based, so we can install it with gem:

{% highlight bash %}
$ gem install capistrano
{% endhighlight %}

#### Set up SSH keys

The idea here is that the same SSH key will get us from our local machine to the staging machine, and then the staging machine will pull from the git server with the same key. This is known as SSH agent forwarding. This way, our private key lives on our machine, but our public key sits on the git server and the staging server(s).

In order to be able to do this we need two things:

1. An account on the target machine
2. Our public key added to `~/.ssh/authorized_keys` file.

So, SSH into the staging machine and add your key to `username`'s ~/.ssh/authorized_keys. You should now be able to pull from git on the server without it asking you for identification.

#### Server prep

We'll be creating a deploy group on the staging machine, adding it to any users who will deploy, and the users that any processes execute as that need to access our web application (ex: Apache/Nginx). The group of our application folder will be assigned to the deploy group.

First add a group named deploy to the server:

{% highlight bash %}
$ sudo groupadd deploy
{% endhighlight %}

We now need to add ourselves and any other users that need access (other developers) to the deploy group, and the user that executes our web server. Opening the `/etc/group` file, and look for the line that beings with `deploy:`. We append users, comma delimited (the integer value will differ for you depending on the machine). The machine I'm using has an nginx server on, so if you have apache, swap out nginx for apache.


{% highlight bash %}
deploy:x:75:myusername,friendusername,apache
{% endhighlight %}

Now we have a deploy group, whose members include us, any other developers who will be deploying and the web server user.

#### Passwordless sudo

It's worth noting that our user account may need the "no password sudo" permission on the server. This is so that Capistrano can execute commands that require root, such as restarting or reloading the web server. This is done in `/etc/sudoers` file with the below line:

{% highlight bash %}
%deploy       ALL = (ALL) NOPASSWD: ALL
{% endhighlight %}

#### Deployment structure and permissions

Let's say we're going to deploy our application to our staging servers in the directory `/var/www/application`.

We will need to create a directory structure as follows

application
-- components
-- deploy

n summary, each folder will have 775 permissions, belong to the deploy group, and have the g+s bit set.

Let's talk about that last item. So our goal is to have Capistrano be able to deploy our application from any user, without running into permission issues.

We could have everyone use a single deploy user, yes. But then when someone leaves the team, we need to change the key-pair in all locations. So instead, let's have everyone use their own key-pair and their own account to deploy. Then when you need to remove someone, you simply remove them from the deploy group and delete their account on the server.

In order to accomplish this, we need the directories and files that are created inside of the application folder to belong to the deploy group. Normally in Linux, when a file or folder is created, its group is assigned to the primary group of the user that created it--which won't be deploy.

When the "set GID bit" -- which stands for set group id -- is set on a folder, it tells the machine this: For any files or folders created under this folder, make them inherit the group that this folder has. Therefore by setting the g+s flag on the application folder, and setting its group to deploy, all files and folders created under it will also inherit the deploy group and the set GID flag. Now, so long as we are a member of the deploy group, along with our web server processes that need to access our application, we won't run into permission issues.

I'm showing each command here separately for explicit concreteness, but if you know the appropriate shortcuts to set multiple attributes at once, please feel free. Note that you may need sudo access to run these commands:

{% highlight bash %}
$ mkdir application
$ chmod 775 application
$ chmod g+s application
$ chown deploy application
$ chgrp deploy application
$ cd application
$ mkdir components deploy
{% endhighlight %}

That's all that really needs to be done here. Let's get ready to deploy.

### Back to our local machine

Inside our laravel application, we need to run cap install inside the project. Once this command is run, our project is "Capified". This places a Capfile, and a config directory into the root of our project.

We'll be working with `staging.rb` and `deploy.rb`

In addition to the files created here, we need to add two files. In addition to the files created here, we will need to add a couple more files:

1. config/myconfig.rb
2. config/myconfig.rb.example

{% highlight ruby %}
set :ssh_options, { user: 'your_username' }
set :tmp_dir, '/home/your_username/tmp'
{% endhighlight %}

myconfig.rb will contain our specific username for SSHing to the server. But since this file has personal information, it should *not* go into Version Control, and the template distribution should be used instead (like laravel env files above). Be sure to add myconfig.rb to .gitignore so the file is ignored. The example file can go in version control so that anyone new joining the project can copy and modify in a similiar fashion to laravel config.

Laravel, and most composer based distributions, come with the vendor folder .gitignore. The reason for this is because this folder is managed by composer. We could make capistrano do this every time we deploy, but this is quite a labour intensive task, and once most of your development is done these packages do not change all that much. There are a few approaches to tackle this:

1. Have multiple .gitignores set up on git, with a deploy branch that doesn't ignore changes to vendor. This way the application has a clean history on master, and the deploy branch is the complete application. This is a strategy I have used for deploying node applications with it's dependencies intact.
2. Set up capistrano to move these files once, and if we need to update it in the future, have a command specifically designed to do this.

Seeing as this is a tutorial about Capistrano and Laravel, I will show you the latter, however I do have a post about setting up a git repo with multiple .gitignore files, which is pretty handy for some use cases.

#### Adding tasks

Running `$ cap -T` from project root will show you the available commands you have. Let's add some now.

We want two commands, one to deploy the application, the other to deploy environment specific (and non git) files.

Let's look at `deploy.rb`

{% highlight ruby %}
set :application, 'application'
set :repo_url, 'git@path_to_git_repo.git'

set :deploy_to, '/var/www/application/deploy'

components_dir = '/var/www/application/components'
set :components_dir, components_dir

# Devops command
namespace :ops do

    desc 'Copy non git files to server'
    task :put_components do
        on roles(:app), in: :sequence, wait: 1 do
          system("tar -zcf ./build/vendor.tar.gz ./vendor ")
          upload! './build/vendor/tar/gz', "#{components_dir}", :recursive => true
          execute "cd #{components_dir}
          tar -zxf /var/www/application/components/vendor.tar.gz
        end
    end

end
{% endhighlight %}

Replace the parts in this file with the names of your application, and if you aren't using this directory structure on your web server, tailor accordingly.

The `:put_components` task within the namespace ops is responsible for zipping our vendor file, uploading it to our staging servers, and expanding it as a folder underneath *application/components* folder that we created.

Now let's look at our staging configuration at *config/deploy/staging.rb*

{% highlight ruby %}
role :app, #w{staging_hostname}

# require custom config
require './config/myconfig.rb'

namespace :deploy do

    desc 'Get stuff ready prior to symlinking'
    task :compile_assets do
      on roles(:app), in: :sequence, wait: 1 do
         execute "cp #{deploy_to}/../components/.env.staging.php #{release_path}"
         execute "cp -r #{deploy_to}/../components/vendor #{release_path}"
      end
    end

    after :updated, :compile_assets

end

namespace :ops do

   desc 'Copy non-git ENV specific files to servers.'
   task :put_env_components do
     on roles(:app), in: :sequence, wait: 1 do
        upload! './.env.staging.php', '#{deploy_to}/../components/.env.staging.php'
     end
   end

end
{% endhighlight %}

Notice how we are including myconfig.rb here. At the top of the file add your staging server hostnames.

`:compile_assets` is the task that moves our vendor and config files from the /components folder into the correct place of the Laravel installation.

Notice the line, after :updated, :compile_assets. This tells Capistrano to call our custom compile_assets task after the internal updated task has finished. This ensures that our files are in place before Capistrano flips the switch by changing the symlink from our old prior build, to this new build. Full details on Capistrano's flow can be found here.

Lastly, in the code sample above, we see we've added the :put_env_components task, which is the final piece of our puzzle. While the :compile_assets task moves our files into place from within the server, the :put_env_components and :put_components tasks get them onto the servers in the first place.

Why is:put_components in config/deploy.rb but :put_env_components is in config/deploy/staging.rb?

Hopefully you can see that the name of the config file for the staging environment is specific to the environment itself, therefore this custom bit has to live in the {environment}.rb file.

### Deploy!

So now from the root of our project we can run `cap -T` and we'll see our two new tasks added. Now we need to move our "components" (the vendor folder, in this case) and our config file onto the staging server. Then, we deploy.

Before deploying, make sure you have your vendor and config files into place with :put_components and :put_env_components. And finally,
let's deploy:

{% highlight bash %}

$ cap staging deploy

{% endhighlight %}

### Next steps

So hopefully you've seen how to create a customisable, configurable asset deployment pipeline in Capistrano, geared towards Laravel 5.

Next steps here would be to create your own custom commands, such as triggering server reboots, memcache server refreshes, dumping composer autoloads. A step beyond this would be to bring puppet into the mix to automate the server control entirely, and use this in conjunction with Capistrano to automate the creation of more application servers.



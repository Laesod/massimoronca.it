---
layout: post
title: A modern workflow for Wordpress using Docker and Dokku
excerpt: Deploying Wordpress is cumbersome and based on old standards. With Docker and Dokku you can improve your workflow and deploy your apps like a boss.
source-name: MIKAMAYHEM
source-url: http://dev.mikamai.com/post/85531658709/a-modern-workflow-for-wordpress-using-docker-and-dokku
---

Every developer, sooner or later, had to deal with [WordPress](http://wordpress.org/), given it is one of the most popular Blog/CMS platform, if not **the** most popular.  
According to Wikipedia, roughly 22% of the web sites run on it, (it means one web site in five) it is widely know by users, [it has a large community](https://wordpress.org/plugins/) (over 30 thousand contributed plugins) and it is easily supported by designers.  

Unfortunately WP was targeted at non-developer people, it had a great success as hosted platform, but working with it from the developer perspective, especially if we look at the workflow, looks clunky and outdated.  

Usually it involves:

- downloading a tar ball of the latest WordPress stable version
- rename `wp-config-sample.php` to `wp-config.php`
- if you're using `git` (and you should!), add the `wp-config.php` to `.gitignore`
- open a connection (possibly not FTP, but probably it will be FTP) and (slowly) upload everything on the dest server
- create a remote`wp-config.php` with the production configuration
- launch the installer
- hope nobody will overwrite `wp-config.php` with the local copy

On the other hand, modern workflows are built around some kind of version control system (usually `git`) where the deploy is managed just by pushing a branch on the public server.  
This is called push-to-deploy and is the one used by [Heroku](http://heroku.com).  
Fortunately, some smart guys created [Docker](http://www.docker.io) and [Dokku](https://github.com/progrium/dokku), two projects that make possible to build you own personal-heroku-like [PaaS](http://en.wikipedia.org/wiki/Platform_as_a_service) in a matter of minutes (If you want to try it, [Digital Ocean](https://www.digitalocean.com/) offers cloud servers with Dokku preinstalled at a starting price of 5$/month).  
Let's see how it applies to WordPress.  

> In the rest of the article I'm going to use a few placeholders
>
> - `app` is the application name
> - `dokku` is the address (or hostanme if configured) of the destination server running Docker & Dokku
> - `dokku-user` is the user running Dokku on the remote machine

First of all we're going to clone WP from [github](http://www.github.com)

```bash
git clone git@github.com:WordPress/WordPress.git app
```

Then we create `wp-config.php`, replace the configuration parameters with environment variables, and commit it.  
This way we won't have to hardcode them.

> same technique can be used to configure the [security keys](http://codex.wordpress.org/Editing_wp-config.php#Security_Keys), I didn't, to keep things short.

```php
define('DB_NAME', getenv('WP_DB_NAME'));

/** MySQL database username */
define('DB_USER', getenv('WP_DB_USER'));

/** MySQL database password */
define('DB_PASSWORD', getenv('WP_DB_PASS'));

/** MySQL hostname */
define('DB_HOST', getenv('WP_DB_HOST'));

```

```bash
git add wp-config.php
git commit -m 'added WordPress configuration'
```

If you use [Apache](http://apache.org/), you can set the values with [`SetEnv`](http://httpd.apache.org/docs/2.2/mod/mod_env.html), if you're running [Nginx](http://nginx.org/) and [phpf-pm](http://php-fpm.org/), you can use the [`ENV` section](http://www.php.net/manual/it/install.fpm.configuration.php#example-73) of your application pool.  
You don't need any of that when deploying through Dokku.  


For our first deploy, we need to add a new `remote` pointing at the Dokku  server

```bash
git remote add dokku dokku-user@dokku:app
```

and push the code

```bash
git push dokku master
```

You will see something like this

```bash
Counting objects: 163187, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (33726/33726), done.
Writing objects: 100% (163187/163187), 84.87 MiB | 4.95 MiB/s, done.
Total 163187 (delta 128758), reused 163156 (delta 128730)
-----> Building app ...
       PHP (classic) app detected
-----> Bundling NGINX 1.4.3
-----> Bundling PHP 5.5.5
-----> Bundling extensions
       phpredis
       mongo
-----> Setting up default configuration
-----> Vendoring binaries into slug
-----> Discovering process types
       Default process types for PHP (classic) -> web
-----> Releasing app ...
-----> Deploying app ...
-----> Cleaning up ...
=====> Application deployed:
       http://app_url

To dokku@dokku:app
 * [new branch]      master -> master
```  

As you can see, everything is already bundled with the PHP buildpack.  
Dokku has detected the PHP app and instructed Docker to create an isolated container that could run the application.  

You can now open up a browser and point to app_url (it can have two formats: http://ip_adress:port or http://app.defaultdomain. Either way, it should launch your app).  

Our wp-config is empty right now, the server will reply with  

![WP Error](http://i.imgur.com/JzhJclD.png)

That's good news, it means it is actually responding to our request.  

 To finish our setup we need a couple more things:

 - create a database
 - configure the app environment with the credentials  

To create a db in our app container, we'll use the [MariaDB plugin for Dokku](https://github.com/Kloadut/dokku-md-plugin).  
There's also a [MySQL plugin](https://github.com/hughfletcher/dokku-mysql-plugin), but it has some annoying bug and since MySQL and MariaDB
are virtually identical, we'll stick with the last one.  
Installing a plugin for Dokku is as easy as running

```bash
cd /var/lib/dokku/plugins
git clone https://github.com/Kloadut/dokku-md-plugin mariadb
dokku plugins-install
```

Some of them don't require the final `plugins-install` step, but it won't hurt if you run it anyway.  

> Tip: you can run dokku commands on your local machine and execute them on the remote one with:
> `ssh dokku-host dokku-command` (i.e. `ssh dokku help`)


We can now create the database

 ```bash

 ssh dokku mariadb:create app

 -----> Creating /home/dokku/app/ENV
-----> Setting config vars and restarting app
DATABASE_URL: mysql2://root:VQpzDZRrEUAkUuAI@172.17.42.1:49170/db
-----> Releasing app ...
-----> Release complete!
-----> Deploying app ...
-----> Deploy complete!

-----> app linked to mariadb/app database

-----> MariaDB container created: mariadb/app

       Host: 172.17.42.1
       Port: 49170
       User: 'root'
       Password: 'VQpzDZRrEUAkUuAI'
       Database: 'db'

 ```

 and set set the Environment variables

 ```bash
 # the format is dokku config:set app key=value key=value
 # I splitted up the command on different lines for clarity
 ssh dokku config:set app WP_DB_HOST='172.17.42.1:49170'
 ssh dokku config:set app WP_DB_NAME='db'
 ssh dokku config:set app WP_DB_USER='root'
 ssh dokku config:set app WP_DB_PASS='VQpzDZRrEUAkUuAI'
 ```

If everything went right, you should now see the standard WordPress install.  
Choose a title, create an admin user and you're ready to go.  
You can work on your local copy, add plugins, work on your theme, and when you're happy with it, you push all the changes and the app is automatically deployed and configured.  

We just scratched the surface of what is possible with Docker & Dokku.  
In a next article we'll see how to create a Dokku plugin to automate the entire process.  

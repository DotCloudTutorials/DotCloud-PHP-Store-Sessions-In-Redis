# How to Store PHP Sessions in Redis

This tutorial will show you how to deploy an app on DotCloud that stores
session information in Redis. This is very important if you’ve scaled
your PHP app and traffic is being load-balanced across multiple nodes.
Without proper session handling, the user will have a different session
on each node.

## A Simple App With Basic Session Handling

Let’s start by deploying a basic app which has multiple nodes and keeps
track of the user’s session. You’ll need to create a new directory, then
inside that empty directory create two files, “index.php” and
“dotcloud.yml”. The “index.php” file should contain:

~~~~ {.sourceCode .php}
<?php
    session_start();
    if (empty($_SESSION['count'])) {
        $_SESSION['count'] = 1;
    } else {
        $_SESSION['count']++;
    }
    echo $_SESSION['count'];
?>
~~~~

and the “dotcloud.yml” file should contain:

~~~~ {.sourceCode .yaml}
www:
    type: php
    instances: 3
~~~~

Now you can deploy these two files to DotCloud by running \`dotcloud
push sessions\` (“sessions” can be replaced by any name you like) from
inside the directory which contains these two files.

When the push is finished, the DotCloud CLI will print the URL for your
new app. Try visiting this URL. If everything worked correctly, the page
will simply show a “1”. If you reload the page, you’ll notice that your
first three page requests return “1”, the next three return “2”, the
next three return “3”, etc. This is because your requests are being
load-balanced across the three nodes of your app, and each node is
keeping track of a separate session because they don’t have access to a
shared data store. Let’s fix this problem!

## Add Redis to Your DotCloud Build File

To add a redis service to your app, simply list it in your dotcloud.yml
file. The file should now look like this:

~~~~ {.sourceCode .yaml}
www:
    type: php
    instances: 3
redis:
    type: redis
~~~~

Next time you push, DotCloud will automatically start this new service
for you.

## Add phpredis to Your App

Next we’ll add phpredis to the project, which is a helpful PHP library
that includes built-in functionality for Redis session handling. Run
these commands from the root directory of your app to add phpredis:

~~~~ {.sourceCode .bash}
mkdir vendor
cd vendor
git clone https://github.com/nicolasff/phpredis.git
~~~~

## Create php.ini

Soon we’ll add a postinstall script that installs phpredis, but first
let’s create a php.ini file in the root directory of your app. Inside
this file, we’ll tell PHP where to find redis.so (the compiled phpredis
library file), and then we’ll set Redis as our session handler.

~~~~ {.sourceCode .ini}
extension=/home/dotcloud/libs/redis.so
session.save_handler = redis
session.save_path = "tcp://@@HOST:@@PORT?auth=@@PASSWORD"
~~~~

The DotCloud posinstall script, which we’re about to add, will replace
the @@HOST, @@PORT, and @@PASSWORD template values for us by reading
them from the environment.json file of your app.

## Add a postinstall script

The last thing we need to do before we push is add a postinstall script.
Each time you push, DotCloud looks for a file called “postinstall” in
your approot and runs that file if it’s present. There are two things we
need to do each time we push: 1) Look for the file “redis.so”. If it’s
not where we expect it to be, we need to compile phpredis and copy
“redis.so” to the right place. 2) Replace @@HOST, @@PORT, and @@PASSWORD
in php.ini with the correct values (which can be found in
/home/dotcloud/environment.json).

~~~~ {.sourceCode .bash}
#!/bin/sh

if [ ! -e /home/dotcloud/libs/redis.so ]
then
    cd /home/dotcloud/current/vendor/phpredis
    phpize
    ./configure
    make
    mkdir /home/dotcloud/libs
    cp modules/redis.so /home/dotcloud/libs/
fi

cd /home/dotcloud
redis_host=$(grep -o -P '(?<=REDIS_HOST": ").+(?=")' environment.json)
redis_port=$(grep -o -P '(?<=REDIS_PORT": ").+(?=")' environment.json)
redis_password=$(grep -o -P '(?<=REDIS_PASSWORD": ").+(?=")' environment.json)
cd /home/dotcloud/current
sed -i s/@@HOST/$redis_host/ php.ini
sed -i s/@@PORT/$redis_port/ php.ini
sed -i s/@@PASSWORD/$redis_password/ php.ini
~~~~

## Push to DotCloud

Now you can push your app to DotCloud! Simply run \`dotcloud push
sessions\`. Now, when you visit the app in your browser, each page
request will return an incremented count. Because your session is now
stored in Redis, all three nodes have access to the same session.


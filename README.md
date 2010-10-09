# Nginx configuration for running Drupal

## Introduction

   This is an example configuration from running Drupal using
   [nginx](http://nginx.org). Which is a high-performance non-blocking
   HTTP server.

   Nginx doesn't use a module like Apache does for PHP support. The
   Apache module approach simplifies a lot of things because what you
   have in reality is nothing less than a PHP engine running on top of
   the HTTP server. 

   Instead nginx uses [FastCGI](http://en.wikipedia.org/wiki/FastCGI)
   proxy all requests for PHP processing to a php fastcgi daemon that
   is waiting for incoming requests and then handles the php file
   being requested.

   Although the fcgi approach is more cumbersome to set up it provides
   a greater degree of control over which actions are permitted, hence
   greater security.

   This configuration uses a lot of stuff stolen from both
   [yhager](github.com/yhager/nginx_drupal),
   [omega8cc](http://github.com/omega8cc/nginx-for-drupal) and
   [Brian Mercer](http://test.brianmercer.com/content/nginx-configuration-drupal)
   configurations. I've incorporated some tidbits of advice I've
   gotten from both the nginx mailing list and the
   [nginx Wiki](http://wiki.nginx.org).

## Layout
   
   The configuration has **two** possible choices.

   1. A **non drush aware** version that uses `wget/curl` to run cron
      and updating the site using `update.php`, i.e., via a web
      interface.

   2. A **drush aware version** that runs cron and updates the
      site using [drush](http://drupal.org/project/drush).

      To get drush to run cron jobs the easiest way is to define your
      own [site aliases](http://drupal.org/node/670460). See the
      example aliases file `example.aliases.drushrc.php` that comes
      under the `examples` directory in the drush distribution.

      Example: You create the aliases for example.com and example.org,
      with aliases `@excom` and `@exnet` respectively.

      Your crontab should contain something like:

          COLUMNS 80
          */50 * * * * /path/to/drush @excom cron > /dev/null
          1 2 * * * /path/to/drush @exnet cron > /dev/null

      This means that the cron job for example.com will be run every
      50 minutes and the cron job for example.net will be run every
      day at 02:01 hours. Check the section 7 of the Drupal
      `INSTALL.txt` for further details about running cron.

      Note that the `/path/to/drush` is the path to the **shell script
      wrapper** that comes with drush not to to the `drush.php`
      script. If using `drush.php` then add `php` in front of the
      `/path/to/drush.php`.
    
    
## Drupal 7
    
   The example configuration can be used in a **drupal 7** or **drupal
   6** site. In drupal 7 there are plenty of new great things. Not only is
   [image handling](http://drupal.org/node/371374) in core. But also
   there's no need for a regex with capturing for appending the query
   string. Therefore the rewrite rule for the `@drupal` location is
   much simpler.
  
   For using the drupal 7 configuration, uncomment out the:

       include sites-available/drupal7.conf;

   line. And comment out:

       include sites-available/drupal.conf;

   Note that you can use the drupal 6 config with drupal 7. But the
   drupal 7 config is **faster** since there's no regex involved in
   the rewrite and also there's no need for the drupal 6 imagecache
   handling rule.
    
## General Features

   1. The use of two `server` directives to do the domain name
   rewriting, usually redirecting `www.example.com` to `example.com`
   or vice-versa. As recommended in
   [nginx Wiki Pitfalls](http://wiki.nginx.org/Pitfalls#Server_Name)
   page.

   2. **Clean URL** support.

   3. Access control for `cron.php`. It can only be requested from a
   set of IPs addresses you specify. This is for the **non drush
   aware** version.

   4. Support for the [Boost](http://drupal.org/project/boost) module.

   5. Support for virtual hosts. The `example.com` file.

   6. Support for [Sitemaps](http://drupal.org/project/site_map) RSS feeds.

   7. Support for the
      [Filefield Nginx Progress](http://drupal.org/project/filefield_nginx_progress)
      module for the upload progress bar.

   8. Use of **non-capturing** regex for all directives that are not
      rewrites that need to use URI components.

   9. IPv6 and IPv4 support.

   10. Use of UNIX sockets in `/tmp/` subdirectory with permissions
       **700**, i.e., accessible only to the user running the process.
   
   You may consider the
   [init script](github.com/perusio/php-fastcgi-debian-script) that I
   make available here on github that launches the PHP FastCGI daemon
   and spawns new instances as required.

## Security Features

   1. The use of a `default` configuration file to block all illegal
      `Host` HTTP header requests.

   2. Access control using
      [HTTP Basic Auth](http://wiki.nginx.org/NginxHttpAuthBasicModule)
      for `install.php` and other Drupal sensitive files. The
      configuration expects a password file named `.htpasswd-users` in
      the top nginx configuration directory, usually `/etc/nginx`. I
      provide an empty file. This is also for the **non drush aware**
      version.

      If you're on Debian or any of its derivatives like Ubuntu you
      need the
      [apache2-utils](http://packages.debian.org/search?suite%3Dall&section%3Dall&arch%3Dany&searchon%3Dnames&keywords%3Dapache2-utils)
      package installed. Then create your password file by issuing:

          htpasswd -d -b -c .htpasswd-users <user> <password>

      You should delete this command from your shell history
      afterwards with `history -d <command number>` or alternatively
      omit the `-b` switch, then you'll be prompted for the password.

      This creates the file (there's a `-c` switch). For adding
      additional users omit the `-c`.

      Of course you can rename the password file to whatever you want,
      then accordingly change its name in drupal_boost.conf.

   3. Support for
      [X-Frame-Options](https://developer.mozilla.org/en/The_X-FRAME-OPTIONS_response_header)
      HTTP header to avoid Clickjacking attacks.

   4. Protection of the upload directory. You can try to bypass the
      UNIX `file` utility or the PHP `Fileinfo` extension and upload a
      fake jpeg:
   
          echo -e "\xff\xd8\xff\xe0\n<?php echo 'hello'; ?>" > test.jpg
      
      If you run `php test.jpg`  you get 'hello'. The fact is that **all
      files** with php extension are either matched by a particular
      location, as is the case for `index.php`, `xmlrpc.php`,
      `update.php` and `install.php` or match the last directive of
      the configuration:

          location ~* ^.+\.php$ {
            return 404; 
          }

      Returning a 404 (Not Found) for every PHP file not matched by
      all the previous locations.

## Enabling and Disabling Virtual Hosts

   I've created a shell script
   [nginx_ensite](http://github.com/perusio/nginx_ensite) that lives
   here on github for quick enabling and disabling of virtual hosts.

## On groups.drupal.org

   There's a [nginx](http://groups.drupal.org/nginx)
   groups.drupal.org group for sharing and learning more about using
   nginx with Drupal.

## Monitoring nginx

   I use [Monit](http://mmonit.com) for supervising the nginx
   daemon. Here's my
   [configuration](http://github.com/perusio/monit-miscellaneous) for
   nginx.

## Caveat emptor

   You should **always** test the configuration with `nginx -t` to see
   if everything is correct. Only after a successful should you reload
   nginx. On Debian and any of its derivatives you can also test the
   configuration by invoking the init script as: `/etc/init.d/nginx
   testconfig`.

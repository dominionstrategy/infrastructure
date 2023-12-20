# Upgrade to Ubuntu 22.04

This document details the process that was followed in order to
upgrade the DominionStrategy server from Ubuntu 10.04 to Ubuntu 22.04,
incorporating the following software upgrades:

| Software  | Old version | New version | Old release date | New release date |
|-----------|-------------|-------------|------------------|------------------|
| Apache    | 2.2.14      | 2.4.52      | 2009-10-08       | 2021-12-21       |
| Caddy     | 2.6.4       | 2.7.5       | 2023-02-14       | 2023-10-11       |
| MediaWiki | 1.19.2      | 1.35.6      | 2012-08-30       | 2022-03-31       |
| MySQL     | 5.1.63      | 8.0.35      | 2012-05-07       | 2023-10-25       |
| PHP       | 5.3.2       | 8.1.2       | 2010-03-04       | 2022-01-20       |
| SMF       | 2.0.13      | 2.1.4       | 2017-01-04       | 2023-06-10       |

## Initial setup

### Server creation

First, I provisioned a new Linode server using the following settings
to match the existing server:

* Image: Ubuntu 22.04 LTS
* Region: Amsterdam, NL (nl-ams)
* Plan: Dedicated 4 GB ($36/mo, 4GB RAM, 2 CPUs, 80GB storage, 4TB
  transfer, 40Gbps/4Gbps bandwidth in/out)

### Account setup

Connected via SSH and created an admin user to use instead of root:

```
% useradd -m -G sudo -s /usr/bin/bash admin
% passwd admin
```

Logged in as admin user and did all following work from there. Copied
my ssh public key to `~/.ssh/authorized_keys` for subsequent access.
System upgrade:

```
% sudo apt update
% sudo apt dist-upgrade
```

### Apache setup

Disable the default Apache site:

```
% sudo rm /etc/apache2/sites-enabled/000-default.conf
```

Create a site configuration for the forum at
`/etc/apache2/sites-enabled/forum.conf`:

```
<VirtualHost *>
    DocumentRoot /var/local/smf
    ServerName forum.dominion.intuitiveexplanations.com
</VirtualHost>
```

And one for the wiki at `/etc/apache2/sites-enabled/wiki.conf`, the
directory selector is to disable RCE via the uploads folder, this
snippet is recommended by MediaWiki:

```
<VirtualHost *>
    DocumentRoot /var/lib/mediawiki
    ServerName wiki.dominion.intuitiveexplanations.com

    <Directory /var/lib/mediawiki/images>
        AllowOverride None
        AddType text/plain .html .htm .shtml .phtml
        php_admin_flag engine off
        Header set X-Content-Type-Options nosniff
    </Directory>
</VirtualHost>
```

Put Apache on port 8080 because we will proxy through Caddy, edit
`/etc/apache2/ports.conf` to:

```
Listen 127.0.0.1:8080
```

We also need to enable the headers module for the `Header` directive
in the wiki site config to work:

```
% sudo ln -s ../mods-available/headers.load /etc/apache2/mods-enabled/
```

Enable the sites and restart Apache:

```
% sudo ln -s ../sites-available/forum.conf ../sites-available/wiki.conf /etc/apache2/sites-enabled/
% sudo systemctl restart apache2
```

We should be able to get a 403 error at least on
`http://localhost:8080` now.

### Caddy setup

Install Caddy:

```
% sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
% curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
% curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
% sudo apt update
% sudo apt install caddy
```

Setup the configuration file at `/etc/caddy/Caddyfile`, I tried using
the `php_fastcgi` directive but couldn't get it to work, it would
always return a 502 error:

```
https://forum.dominion.intuitiveexplanations.com:443 {
    reverse_proxy 127.0.0.1:8080
}

https://wiki.dominion.intuitiveexplanations.com:443 {
    reverse_proxy 127.0.0.1:8080
}
```

Make sure the DNS records are setup for these domains to resolve
correctly, there need to be unproxied A records for the Linode public
IP. Otherwise cert issuance will fail and you might hit Let's Encrypt
rate limits. Then restart Caddy:

```
% sudo systemctl restart caddy
```

### Database setup

The database is running very close to maximum capacity on the original
machine and it seems when copying over a mysqldump to the new machine
the disk fills up entirely. This appears to be due to the binlog. That
shouldn't happen due to the default setting of `max_binlog_size =
100M` but perhaps the binlog isn't flushed within a single operation.
Either way, disable the binlog at least temporarily (see below).

Also, when upgrading MediaWiki from the old version using the upgrade
script, one gets `Error 1206: The total number of locks exceeds the
lock table size` with the default settings, so change the buffer pool
size to larger than the default.

Create `/etc/mysql/mysql.conf.d/zz-site.cnf` with contents:

```
[mysqld]
skip-log-bin
innodb_buffer_pool_size=1GB
```

And then `sudo systemctl restart mysql`. Can verify as follows:

```
% sudo mysql
mysql> SELECT @@global.log_bin;
+------------------+
| @@global.log_bin |
+------------------+
|                0 |
+------------------+
1 row in set (0.00 sec)
```

Update the root mysql user so it is possible to login with password,
we will need this for MediaWiki:

```
% sudo mysql
mysql> SELECT User, Host, plugin FROM mysql.user;
+------------------+-----------+-----------------------+
| User             | Host      | plugin                |
+------------------+-----------+-----------------------+
| debian-sys-maint | localhost | caching_sha2_password |
| mysql.infoschema | localhost | caching_sha2_password |
| mysql.session    | localhost | caching_sha2_password |
| mysql.sys        | localhost | caching_sha2_password |
| root             | localhost | auth_socket           |
+------------------+-----------+-----------------------+
5 rows in set (0.00 sec)

mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'somerandomstring';
Query OK, 0 rows affected (0.00 sec)
```

Now it should be possible to login with `mysql -u root -p` and the
provided password. Create a database for MediaWiki:

```
mysql> CREATE DATABASE dswiki;
Query OK, 1 row affected (0.01 sec)
```

Also generate a new password for the MediaWiki user (best to rotate it
after more than a decade), and create the user, since users aren't
copied over by mysqldump:

```
mysql> CREATE USER 'dswikiuser'@'localhost' IDENTIFIED BY 'somerandomstring';
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT ALL PRIVILEGES ON dswiki.* TO 'dswikiuser'@'localhost' WITH GRANT OPTION;
Query OK, 0 rows affected (0.00 sec)
```

<https://www.mediawiki.org/wiki/Manual:Installing_MediaWiki#MariaDB/MySQL>
documents the required permissioning setup for MediaWiki.

We follow the same steps to get a forum database and application user.
The old server had the server using root, so we fix that now:

```
mysql> CREATE DATABASE smf;
Query OK, 1 row affected (0.00 sec)

mysql> CREATE USER 'smfuser'@'localhost' IDENTIFIED BY 'somerandomstring';
Query OK, 0 rows affected (0.08 sec)

mysql> GRANT ALL PRIVILEGES ON smf.* TO 'smfuser'@'localhost' WITH GRANT OPTION;
Query OK, 0 rows affected (0.00 sec)
```

Documentation on the required permissions for the SMF user:
<https://wiki.simplemachines.org/smf/Installing>

### MediaWiki setup

Install MediaWiki, the welcome page should then be accessible on port
\443. This might not be ideal from a security perspective so you could
also stop Caddy and just proxy port 8080 over SSH instead. But this is
probably good enough:

```
% sudo apt install mediawiki
```

Copy over the MediaWiki site settings from the old server:

```
% sudo scp -oHostKeyAlgorithms=+ssh-dss youraccount@oldmachine-ip:/var/www/mediawiki/LocalSettings.php /etc/mediawiki/LocalSettings.php
```

Update the following things in the local settings file after copying:

* `$wgDBpassword` - update to the password previously generated for
  `dswikiuser`.
* `$wgSecretKey` - rotate just for best practices. Try `head -c32
  /dev/urandom | xxd -p | tr -d '\n'` to get a good value.
* `$wgUpgradeKey` - generate a new random value, and set.
* `$wgServer` - if testing on a different hostname, then replace
  `http://wiki.dominionstrategy.com` and
  `https://wiki.dominionstrategy.com` with the appropriate values.

To migrate to the new extension registration system, replace these
lines:

```
require_once( "$IP/extensions/ParserFunctions/ParserFunctions.php" );
require_once("$IP/extensions/Cite/Cite.php");
require_once("$IP/extensions/ParserFunctions/ParserFunctions.php");
require_once("$IP/extensions/ConfirmEdit/ConfirmEdit.php");
require_once("$IP/extensions/ConfirmEdit/QuestyCaptcha.php");
```

As such:

```
wfLoadExtension('Cite');
wfLoadExtension('ConfirmEdit');
wfLoadExtension('ConfirmEdit/QuestyCaptcha');
wfLoadExtension('ParserFunctions');
```

Copy over the custom wiki skin (no need for anything fancier than
`scp` as this is just a few dozen files):

```
% sudo ln -s /etc/mediawiki/skins/common /etc/mediawiki/skins/dsskin /etc/mediawiki/skins/Dsskin.php /usr/share/mediawiki/skins/
% sudo mkdir /etc/mediawiki/skins
% sudo scp -r -oHostKeyAlgorithms=+ssh-dss youraccount@oldmachine-ip:/var/www/mediawiki/skins/dsskin /etc/mediawiki/skins/
% sudo scp -r -oHostKeyAlgorithms=+ssh-dss youraccount@oldmachine-ip:/var/www/mediawiki/skins/common /etc/mediawiki/skins/
% sudo scp -oHostKeyAlgorithms=+ssh-dss youraccount@oldmachine-ip:/var/www/mediawiki/skins/Dsskin.php /etc/mediawiki/skins/
```

We'll need to migrate this skin to the new loading system that was
introduced in MediaWiki 1.25 and later:
<https://www.mediawiki.org/wiki/Manual:Skin_autodiscovery#Migration_guide>
<https://www.mediawiki.org/wiki/ResourceLoader/Developing_with_ResourceLoader>

```
% sudo mv /etc/mediawiki/skins/Dsskin.php /etc/mediawiki/skins/dsskin/dsskin.php
```

Also add this at the top of the file, just after the `MEDIAWIKI`
define check:

```
$wgValidSkinNames['dsskin'] = 'Dsskin';
```

And replace this:

```
                $out->addModuleScripts( 'skins.dsskin' );
```

With this:

```
                $out->addModules( 'skins.dsskin' );
```

And this:

```
                                wfRunHooks( $hook, array( &$this, true ) );
```

With this:

```
                                Hooks::run( $hook, array( &$this, true ) );
```

Now add to `/etc/mediawiki/LocalSettings.php` before the setting of
`$wgDefaultSkin`:

```
require_once "$IP/skins/dsskin/dsskin.php";
```

<https://www.mediawiki.org/wiki/Manual:Upgrading> documents the
MediaWiki upgrade process. We'll copy some more files later, during
the failover window.

Upgrade docs say to check your extensions, but no work is required in
our case. The following extensions were installed on the old server:

```
Cite
ConfirmAccount
ConfirmEdit
Gadgets
Nuke
ParserFunctions
Renameuser
Vector
WikiEditor
```

These mostly come by default in MediaWiki 1.35.6 (we are upgrading
from 1.19.2), the missing ones being `ConfirmAccount` and `Vector`:

* <https://www.mediawiki.org/wiki/Extension:ConfirmAccount>
* <https://www.mediawiki.org/wiki/Extension:Vector>

However, the code in `LocalSettings.php` that loads `ConfirmAccount`
is commented on the old server, so there is no need to install the
extension on the new server. And the wiki page for `Vector` says you
no longer need it because its functionality is provided by default in
MediaWiki 1.22 and later.

There was a custom `.htaccess` file for the wiki which blocked a
specific /24 CIDR range, last edited in 2017. I decided it was
probably obsolete and did not copy it over.

### SMF setup

Install SimpleMachineForum (SMF):

```
% sudo mkdir /var/local/smf
% cd /var/local/smf
% sudo wget 'https://download.simplemachines.org/index.php?thanks;filename=smf_2-0-13_install.tar.gz' https://download.simplemachines.org/index.php/smf_2-0-13_install.tar.gz
% sudo rm index*
% sudo mv smf* smf-old.tar.gz
% sudo wget https://download.simplemachines.org/index.php/smf_2-1-4_upgrade.tar.gz -O smf-new.tar.gz
% sudo tar -xf smf-old.tar.gz
% sudo tar -xf smf-new.tar.gz
% sudo rm smf*.tar.gz
% sudo chown -R --from=500:500 www-data:www-data .
% sudo chown -R --from=1001:1003 www-data:www-data .
```

Note that the way SMF upgrades work, is they expect you to upgrade in
place, by extracting the "upgrade" package on top of the existing
installation. To achieve a blue-green cutover, we instead create a
fresh installation of the old version, copy over specific desired user
data, and upgrade that.

Also note that the reason for the bizarre first wget command is that
if you try to download the tar-ball directly, then the genius
sysadmins in charge of this website will serve you a 403 that says:

```
Sorry but you can not directly download an archived file without first going through the Simple Machines website.%
```

You have to visit the "thanks" page to get a `PHPSESSID` cookie set,
which enables the download. Real great file server you are running
there, folks. Super convenient. I'm sure this has not broken anybody's
packaging scripts or deployment pipelines.

Luckily wget has a built-in functionality of retaining session cookies
when processing multiple URLs in the same command line. Unluckily
there is no way to use `-O` with multiple files (why...?), see
<https://superuser.com/a/336672>, so we have to do the rename dance.

## Critical window: migrate wiki

This section covers restoring the db over to the new server. We have
to put the old wiki into read-only mode, copy it over, and update the
DNS records.

First we edit `/var/www/mediawiki/LocalSettings.php` on the old server
and add to the bottom:

```
$wgReadOnly = "We are upgrading MediaWiki, please be patient. This wiki will be back in a few hours. You can follow along on Discord in #wiki-general if you want.";
```

We also have to actually put the db into read-only mode because as
MediaWiki documentation notes, `$wgReadOnly` does not stop automated
processes from making changes.

```
% mysql -u root -p -r <<< "SELECT TABLE_NAME FROM information_schema.tables WHERE TABLE_SCHEMA = 'dswiki'" | sed 's/.*/LOCK TABLE & READ;/' > lock_tables.sql
% mysql -u root -p
mysql> SOURCE lock_tables.sql;
... perform migration while keeping session open ...
mysql> UNLOCK TABLES;
```

Next step is to restore the db. We have to do this as an online
migration because the db is too large a percentage of the total
available disk size (on both old and new machines) to hold more than
one copy of it at a time. Luckily what we can do is `mysqldump` it
from the old machine and pipe directly into `mysql` on the new
machine. Setting up the direct pipe is a little tricky because we have
to have a real tty on the old machine in order to enter auth
credentials, which means we have to use a reverse tunnel. Run these
two commands in parallel from the new machine:

```
% ssh -R 127.0.0.1:2222:127.0.0.1:22 -oHostKeyAlgorithms=+ssh-dss youraccount@oldmachine-ip
% nc -l localhost 8888 -q0 | gunzip | mysql -u root -p dswiki
```

Then in the SSH session on the old machine:

```
% mysqldump -u root -p dswiki --single-transaction --quick --lock-tables=false --disable-keys --extended-insert --no-autocommit | sed 's/tinytext/varchar(255)/' | gzip | nc localhost 8888
```

The wiki database at time of dumping was around 50GB and took about 30
minutes to fully dump and restore when using the options above. It
might be possible to make things go faster with some tuning, but this
should be acceptable I think.

We also want to copy over the image files, this is documented as a
separate step in the MediaWiki upgrade guide. The image copying can be
done in parallel with the database restore. Just `scp` isn't
sufficient because there are too many small files and `scp` is not
well optimized for that. Using `tar` speeds it up considerably.

```
% sudo -u www-data -g www-data bash -c "ssh -oHostKeyAlgorithms=+ssh-dss -oUserKnownHostsFile=/dev/null youraccount@oldmachine-ip tar -C /var/www/mediawiki/images -czf - . | tar -C /var/lib/mediawiki/images --skip-old-files -xzf -"
```

With this command, copying 46,500 images comprising 3.6GB took 2m30s.

Once the database and other files are copied over we can set up
MediaWiki and it should be able to detect the existing database and
upgrade it. Go to <https://wiki.dominion.intuitiveexplanations.com/>
and there should be a prompt to set up the wiki interactively through
the web interface. Again it might be smart to do this with a port
forward without exposing Apache outside localhost, but it's probably
not a huge deal.

The environmental checks should all be passing based on the
configuration we did earlier, so double check those if any are
reported failing.

Now once we get to the database connection page we actually don't want
to do this through the web interface. Instead we want to follow
<https://www.mediawiki.org/wiki/Manual:Upgrading#Run_the_update_script>
to upgrade at the command line, because running 10 years of migrations
on a 50GB database takes a bit of time and we don't want our browser
session to timeout (it is configured in `php.ini` to do so after 24
minutes according to what I read).

Go to `/var/lib/mediawiki/maintenance` and run (no superuser needed):

```
php update.php
```

The update takes around 5-10 minutes in my testing.

## Critical window: migrate forum

This section covers restoring the db over to the new server. We have
to put the old forum into read-only mode, copy it over, and update the
DNS records.

FIXME: put into read-only mode

The forum database is small enough that we can hold the entire dump on
disk at once, so we don't need to do complex streaming shenanigans
like with the wiki.

Connect to the old server:

```
% ssh -oHostKeyAlgorithms=+ssh-dss youraccount@oldmachine-ip
```

Create a mysqldump, this took about 45 seconds in my testing:

```
% mysqldump -u root -p smf --single-transaction --quick --lock-tables=false --disable-keys --extended-insert --no-autocommit > /tmp/forum.sql
```

Copy it over from the new machine, this took about a minute to
transfer 1.7GB in my testing:

```
% scp -oHostKeyAlgorithms=+ssh-dss youraccount@oldmachine-ip:/tmp/forum.sql /tmp/forum.sql
```

Then on the old machine, delete it to avoid wasting space:

```
% rm /tmp/forum.sql
```

Now restore the dump into the database that you already created, this
took a little over 5 minutes when I tried:

```
% mysql -u root -p smf < /tmp/forum.sql
% rm /tmp/forum.sql
```

We also have to copy over settings and user files from the old server.
We'll copy everything to start with (less than 1GB, takes under a
minute to copy in my testing), and then selectively extract. From the
old server:

```
% mkdir /tmp/forum
% ssh -oHostKeyAlgorithms=+ssh-dss youraccount@oldmachine-ip tar -C /var/www/dominionstrategy.com/forum -czf - . | tar -C /tmp/forum -xzf -
```

And, copy over the requisite files:

```
% sudo cp /tmp/forum/Settings.php /var/local/smf/
```

FIXME more...

Now we can upgrade the database. From `/var/local/smf`:

```
php upgrade.php
```

## Troubleshooting info

Try this in `LocalSettings.php`:

```
// at beginning (after php block)
error_reporting( -1 );
ini_set( 'display_errors', 1 );

// at end
$wgShowExceptionDetails = true;
```

Without that, you just get an empty / truncated response in the
browser when there is an error, and the logs go nowhere because
apparently PHP doesn't have a real concept of server side error
logging because of course not.

## FIXMEs

* Forum todo...
* Forum needs to be put into read-only mode (or a banner added at
  least)
* Need to figure out TLS cutover
* DNS settings

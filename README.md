Install Mastodon on CentOS7 cPanel server.

Preface: 
 To get Mastodon working on a cPanel install we will work with its Easy Apache web service and use the cPanel Ruby which will be upgraded in place after install to the version that works with current version and LTS version of NodeJS packages and Apache modules, the standard FOSS packages that we will install are Postgresql 11 as the default postgres for cPanel is the outdataded 9.2, this means you have to make sure you are not using the cPanel version of Postgres, as it would make it very hard to install. We will also install Nodejs package manger called Yarn from the source package and the EPEL repository REDIS object cache package installed.  


Step 1:

Install some of the required packages and repositories. 
```
yum -y install epel-release

yum update

yum -y install ImageMagick git libxml2-devel libxslt-devel gcc bzip2 openssl-devel zlib-devel gdbm-devel ncurses-devel autoconf automake bison gcc-c++ libffi-devel libtool patch readline-devel sqlite-devel glibc-headers glibc-devel libyaml-devel libicu-devel libidn-devel protobuf protobuf-compiler protobuf-devel

yum install curl git gpg gcc git-core zlib zlib-devel gcc-c++ patch readline readline-devel libyaml-devel libffi-devel openssl-devel make autoconf automake libtool bison curl sqlite-devel ImageMagick libxml2-devel libxslt-devel gdbm-devel ncurses-devel glibc-headers glibc-devel libicu-devel libidn-devel 
```

Step 2:

Install a few more dependencies which are required to build the Ruby installation and other dependencies. You may have to remove(*yum remove duplicate_package_name*)duplicates from conflicts with other package to install below.
```
yum  groupinstall 'Development Tools'
```

Step 3:

Install cPanel Nodejs package and create symlinks to binaries. 
```
yum install ea-nodejs16

ln -s /opt/cpanel/ea-nodejs16/lib/node_modules/npm/bin/npm-cli.js /bin/ea-npm16
ln -s /bin/ea-npm16 /bin/npm

ln -s /opt/cpanel/ea-nodejs16/bin/node /bin/ea-node16
ln -s /bin/ea-node16 /bin/node

```

Step 4:

Install REDIS object cache and enable it for reboot.
```
yum -y install redis

systemctl start redis
systemctl enable redis
```

Step 5:

Add the Software Collections SIG repository and install Postgresql 11 database. Although newer versions are in repo, version 11 has the development files needed to compile the Ruby postgres Gems against. 
```
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
yum install -y yum-utils centos-release-scl-rh
yum-config-manager --disable centos-sclo-rh
yum --enablerepo=centos-sclo-rh install llvm-toolset-7-clang
yum install postgresql11-server postgresql11-devel libpqxx-devel libpqxx  ea-openssl-devel
```

Step 6:

Change user to postgres to setup database and then exit back to root to start and enable the database to start at boot.
```
su - postgres
/usr/pgsql-11/bin/initdb
exit
systemctl enable postgresql-11
systemctl start postgresql-11
```

Step 7 :
Login to the PostgreSQL shell:live
Create a new user for the Mastodon instance that is able to create DB tables and alter to add a password:

```
sudo -u postgres psql
postgres=# CREATE USER mastodon CREATEDB;
 CREATE ROLE
postgres=# ALTER USER mastodon WITH PASSWORD 'th1s1sagr3atpa$$w0rd';
 ALTER ROLE
\q
```

Step 8:

Install cPanel Ruby package. The final version will be 3.0.4, but this base install is needed to be able to upgrade the language.  
```
yum install ea-ruby27-ruby
rvm install "ruby-3.0.4"
rvm use 3.0.4 --default
```

Step 9:

Dwnload and install NodeJS package dependency installer Yarn
```
cd ~
curl -o- -L https://yarnpkg.com/install.sh | bash
yarn install --pure-lockfile
```

Step 10:

Inside WHM/cPanel you can create a separate cPanel account with different user or in this example as a sub-directory of the primary account. 
Follow the below breadcrumb path in WHM/cPanel to create it.  
```
WHM -> List Accounts -> User cPanel -> Domains -> Create A New Domain. 
Create a New Domain
mastodon.spotcheckit.org
Then click "Submit"

Should be able to check the new url at https://mastodon.spotcheckit.org
```

Step 11:

Now switch to the home directory of the Mastodon user and clone the application repository files from Github.
```
cd /home/dyount/mastodon.spotcheckit.org
git clone https://github.com/tootsuite/mastodon.git live
cd ./live
git checkout $(git tag -l | grep -v 'rc[0-9]*$' | sort -V | tail -n 1)
gem install bundler
```

Step 12:

Change the cPanel account document root from the cPanel -> Domain  -> domain.tld -> click "Manage" button -> New Document Root enter "mastodon.spotcheckit.org/live/public" Click "Update". Now when you try the url https://mastodon.spotcheckit.org it will show the raw Mastodon files instead of the subdirectory root. 

Step 13:

Make modification to Gem files for current postgresql gem that will work with Postgresaql version 11 and install needed support libaries. 
```
gem install pg -- --with-pg-config=/usr/pgsql-11/bin/pg_config
```
Edit Gemfile and Gemfile.lock and change all refrences to pg 1.4 or pg 1.4.3 to 1.4.5
```
bundle config unset deployment
bundle install
```

Step 14:

Copy example enviroment file and before editing this file, generate four different secrets by running the following command three times. You will need to set these secrets in the configuration file variables RAILS_ENV,VAPID_PRIVATE_KEY,VAPID_PUBLIC_KEY We will foll out some other parts , but the install in the next step will ask as well, just make sure to get the keys installed.  
```
cp .env.production.sample .env.production

RAILS_ENV=production bundle exec rake secret
RAILS_ENV=production bundle exec rake secret
RAILS_ENV=production bundle exec rake secret
RAILS_ENV=production bundle exec rake secret
```
Sections needing to be changed in .env.production file. 
Use the database credtials we used in Step 7. We wont 
be using Elasticsearch or S3 in this install
```
LOCAL_DOMAIN=mastodon.spotcheckit.org

DB_USER=mastodon
DB_NAME=mastodon_production
DB_PASS=th1s1sagr3atpa$$w0rd

ES_ENABLED=false

S3_ENABLED=false

SECRET_KEY_BASE=<SECRET GENERATED> 
OTP_SECRET=<SECRET GENERATED>
VAPID_PRIVATE_KEY=<SECRET GENERATED>
VAPID_PUBLIC_KEY=<SECRET GENERATED>

SMTP_SERVER=spotcheckit.org
SMTP_PORT=587
SMTP_LOGIN=notifications
SMTP_PASSWORD=AReallyGo0Dpas$word
SMTP_FROM_ADDRESS=notifications@mastodon.spotcheckit.org
```

Step 15:

Now that we will be using the email from the cPanel Exim service. We will create the email for mastodon in cPanel by following the breadcrumbs and make sure we use the same credtials as in the env.production file from previous step. 

WHM -> cPanel -> Email -> Email Accounts -> Click "Create" -> Select subdomain "mastodon.spotcheckit.org" -> Username "notifications" -> Password Click the Set Password Now radio button "AReallyGo0Dpas$word" -> Create

Step 16:

Now we will run the Mastodon install from inside the live directory and fill out the install options. 

```
[root@host live]# RAILS_ENV=production bundle exec rake mastodon:setup

Your instance is identified by its domain name. Changing it afterward will break things.
Domain name: mastodon.spotcheckit.org

Single user mode disables registrations and redirects the landing page to your public profile.
Do you want to enable single user mode? No

Are you using Docker to run Mastodon? no

PostgreSQL host: /var/run/postgresql
PostgreSQL port: 5432
Name of PostgreSQL database: mastodon_production
Name of PostgreSQL user: mastodon
Password of PostgreSQL user: 
Database configuration works! 🎆

Redis host: localhost
Redis port: 6379
Redis password: 
Redis configuration works! 🎆

Do you want to store uploaded files on the cloud? No

Do you want to send e-mails from localhost? yes
E-mail address to send e-mails "from": Mastodon <notifications@mastodon.spotcheckit.org>
Send a test e-mail with this configuration right now? Yes
Send test e-mail to: factorf2@yahoo.com

This configuration will be written to .env.production
Save configuration? Yes

Now that configuration is saved, the database schema must be loaded.
If the database already exists, this will erase its contents.
Prepare the database now? Yes
Running `RAILS_ENV=production rails db:setup` ...


Created database 'mastodon_production'
Done!

The final step is compiling CSS/JS assets.
This may take a while and consume a lot of RAM.
Compile the assets now? Yes
Running `RAILS_ENV=production rails assets:precompile` ...


yarn install v1.22.19
[1/6] Validating package.json...
[2/6] Resolving packages...
[3/6] Fetching packages...
[4/6] Linking dependencies...
warning Workspaces can only be enabled in private projects.
[5/6] Building fresh packages...
[6/6] Cleaning modules...
Done in 18.71s.
...
...
Compiling...
Compiled all packs in /home/dyount/mastodon.spotcheckit.org/live/public/packs
Done!

All done! You can now power on the Mastodon server 🐘

Do you want to create an admin user straight away? Yes
Username: admin
E-mail: factorf2@yahoo.com
You can login with the password: Ag00d paSSw0rd
You can change your password once you login.
```

Step 17:

Create SystemD files to handle stop, start and restart of the Mastodon service. 
```
vim /etc/systemd/system/mastodon-web.service
[Unit]
Description=Mastodon Web Service
After=network.target

[Service]
Type=simple
User=root
Group=root
WorkingDirectory=/home/dyount/mastodon.spotcheckit.org/live/
Environment="RAILS_ENV=production"
Environment="PORT=3000"
ExecStart=/bin/bash -lc 'bundle exec puma -C config/puma.rb'
ExecReload=/bin/kill -SIGUSR1 $MAINPID
TimeoutSec=15
Restart=always

[Install]
WantedBy=multi-user.target

vim /etc/systemd/system/mastodon-sidekiq.service
[Unit]
Description=mastodon-sidekiq
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/home/dyount/mastodon.spotcheckit.org/live
Environment="RAILS_ENV=production"
Environment="DB_POOL=5"
ExecStart=/bin/bash -lc 'bundle exec sidekiq -c 5 -q default -q push -q mailers -q pull'
TimeoutSec=15
Restart=always

[Install]
WantedBy=multi-user.target

vim /etc/systemd/system/mastodon-streaming.service
[Unit]
Description=mastodon-streaming
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/home/dyount/mastodon.spotcheckit.org/live
Environment="NODE_ENV=production"
Environment="PORT=4000"
ExecStart=/bin/ea-npm run start
TimeoutSec=15
Restart=always

[Install]
WantedBy=multi-user.target
```

Step 18:

Create and setup cPanel's Easy Apache reverse proxy files for web sockets. 
```
mkdir /usr/local/apache/conf/userdata/ssl/2_4/dyount/mastodon.spotcheckit.org/ 
cd /usr/local/apache/conf/userdata/ssl/2_4/dyount/mastodon.spotcheckit.org/
vim proxy_pass.conf

ProxyPreserveHost On
RequestHeader set X-Forwarded-Proto "https"
ProxyAddHeaders On

# these files / pathes don't get proxied and are retrieved from DocumentRoot
ProxyPass /500.html !
ProxyPass /sw.js !
ProxyPass /robots.txt !
ProxyPass /manifest.json !
ProxyPass /browserconfig.xml !
ProxyPass /mask-icon.svg !
ProxyPassMatch ^(/.*\.(png|ico)$) !
ProxyPassMatch ^/(assets|avatars|emoji|headers|packs|sounds|system) !
# everything else is either going to the streaming API or the web workers
ProxyPass /api/v1/streaming ws://localhost:4000
ProxyPassReverse /api/v1/streaming ws://localhost:4000
ProxyPass / http://localhost:3000/
ProxyPassReverse / http://localhost:3000/


mkdir /usr/local/apache/conf/userdata/std/2_4/dyount/mastodon.spotcheckit.org/
cd /usr/local/apache/conf/userdata/std/2_4/dyount/mastodon.spotcheckit.org/
vim proxy_pass.conf

ProxyPreserveHost On
RequestHeader set X-Forwarded-Proto "https"
ProxyAddHeaders On

# these files / pathes don't get proxied and are retrieved from DocumentRoot
ProxyPass /500.html !
ProxyPass /sw.js !
ProxyPass /robots.txt !
ProxyPass /manifest.json !
ProxyPass /browserconfig.xml !
ProxyPass /mask-icon.svg !
ProxyPassMatch ^(/.*\.(png|ico)$) !
ProxyPassMatch ^/(assets|avatars|emoji|headers|packs|sounds|system) !
# everything else is either going to the streaming API or the web workers
ProxyPass /api/v1/streaming ws://localhost:4000
ProxyPassReverse /api/v1/streaming ws://localhost:4000
ProxyPass / http://localhost:3000/
ProxyPassReverse / http://localhost:3000/
```

Step 19:

Add reverse proxy to apaches configuration scan,then rebuild and redstart Easy apache web service.  
```
/scripts/ensure_vhost_includes --user=dyount
/usr/local/cpanel/scripts/rebuildhttpdconf
/usr/local/cpanel/scripts/restartsrv_httpd
```

Step 20:

Now we will change the permissions of the Mastodon install to the cPanel account owner. 
```
cd /home/dyount/mastodon.spotcheckit.org/
chown -R dyount. live
```

Start the SystemD and enable the three services. 
```
systemctl enable mastodon-web.service
systemctl start mastodon-web.service
systemctl status mastodon-web.service

systemctl enable mastodon-sidekiq.service
systemctl start mastodon-sidekiq.service
systemctl status mastodon-sidekiq.service

systemctl enable mastodon-streaming.service
systemctl start mastodon-streaming.service
systemctl status mastodon-streaming.service

```

At this point you should be able to go to the URL in your browser setup for Mastodon and it will be ready to login and setup.
I will add some scripts to make it easier to add additonal instances of Mastodon so other subdomains or accounts can have their own install. 

Below are the B steps to not have you repeat the above steps and do a totally reinstall once you have a base to clone from on the server and this can be repeated over and over for new domains or cPanel accounts by changing the names and increasing the port numbers by one. This would be a primary reason someone would want to install Mastodon on a cPanel server and take advantage of multiple user setups with Apache. While each user can now have their own instance , the start and stop of the service will still need to be done by root user. May find a way to have each user have this capability in future edits. 

Step 7B:
```
I changed the user from mastodon to mastodon2
```

Step 10B:
```
I changed the domain name from mastodon.spotcheckit.org to mastodon2.spotcheckit.org
```

Step 11B:
```
Copied the live directory to the new domain. 
cd /home/dyount/mastodon.spotcheckit.org
cp -a ./live /home/dyount/mastodon2.spotcheckit.org/
```

Step 12B:
```
Change the document root to the mastodon public files. 
New Document Root enter "mastodon2.spotcheckit.org/live/public" Click "Update".
```

Step 14B:
```
Create four new secret keys. 
RAILS_ENV=production bundle exec rake secret
RAILS_ENV=production bundle exec rake secret
RAILS_ENV=production bundle exec rake secret
RAILS_ENV=production bundle exec rake secret

Make sure you change the database name and user to the new one the rest we will reenter on install.
vim .env.production 

DB_USER=mastodon2
DB_NAME=mastodon2_production
```

Step 16B:
```
Now we will run the second instance of the Mastodon install from inside the new live directory.
[root@host live]# RAILS_ENV=production bundle exec rake mastodon:setup
```

Step 17B:
```
Create SystemD files to handle stop, start and restart of the second Mastodon service.
We can copy the systemD files to the new instance. 

cp /etc/systemd/system/mastodon-web.service /etc/systemd/system/mastodon2-web.service
Change the domain name from mastodon.spotcheckit.org to mastodon2.spotcheckit.org
Change PORT=3000 to PORT=3001

cp /etc/systemd/system/mastodon-sidekiq.service /etc/systemd/system/mastodon2-sidekiq.service
Change the domain name from mastodon.spotcheckit.org to mastodon2.spotcheckit.org

cp /etc/systemd/system/mastodon-streaming.service /etc/systemd/system/mastodon2-streaming.service
Change the domain name from mastodon.spotcheckit.org to mastodon2.spotcheckit.org
Change Environment="PORT=4000" to Environment="PORT=4001"
```

Step 18B:
```
mkdir /usr/local/apache/conf/userdata/ssl/2_4/dyount/mastodon2.spotcheckit.org/ 
cd /usr/local/apache/conf/userdata/ssl/2_4/dyount/mastodon2.spotcheckit.org/
vim proxy_pass.conf
Change ports 4000 to 4001
Change ports 3000 to 3001

mkdir /usr/local/apache/conf/userdata/std/2_4/dyount/mastodon2.spotcheckit.org/
cd /usr/local/apache/conf/userdata/std/2_4/dyount/mastodon2.spotcheckit.org/
vim proxy_pass.conf
Change ports 4000 to 4001
Change ports 3000 to 3001
```

Step 19B:
```
Redo this step as is. 
```

Step 20B:
```
Make sure the ownership is correct to the account and start the services with new name.

systemctl enable mastodon2-web.service
systemctl start mastodon2-web.service
systemctl status mastodon2-web.service

systemctl enable mastodon2-sidekiq.service
systemctl start mastodon2-sidekiq.service
systemctl status mastodon2-sidekiq.service

systemctl enable mastodon2-streaming.service
systemctl start mastodon2-streaming.service
systemctl status mastodon2-streaming.service
```




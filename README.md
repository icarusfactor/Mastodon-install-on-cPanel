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
masotodon.spotcheckit.org
Then click "Submit"
```

Step 10:

Now switch to the home directory of the Mastodon user and clone the application repository files from Github.
```
cd /home/dyount/mastodon.spotcheckit.org
git clone https://github.com/tootsuite/mastodon.git live
cd ~/live
git checkout $(git tag -l | grep -v 'rc[0-9]*$' | sort -V | tail -n 1)
gem install bundler
```

Step 11:

Change the cPanel account document root from the cPanel -> Domain  -> domain.tld -> click "Manage" button -> New Document Root enter "mastodon.spotcheckit.org/live/public" Click "Update".

Step 12:

Make modification to Gem files for current postgresql gem that will work with Postgresaql version 11 and install needed support libaries. 
```
gem install pg -- --with-pg-config=/usr/pgsql-11/bin/pg_config
```
Edit Gemfile and Gemfile.lock and change all refrences to pg 1.4 or pg 1.4.3 to 1.4.5
```
bundle config unset deployment
bundle install
```

Step 13:

Copy example enviroment file and before editing this file, generate three different secrets by running the following command three times. You will need to set these secrets in the configuration file variables RAILS_ENV,VAPID_PRIVATE_KEY,VAPID_PUBLIC_KEY
```
cp .env.production.sample .env.production

RAILS_ENV=production bundle exec rake secret
RAILS_ENV=production bundle exec rake secret
RAILS_ENV=production bundle exec rake secret
```

Step 14:

Example of Email section of Mastodon environment. 
```
# Sending mail
# ------------
SMTP_SERVER=spotcheckit.org
SMTP_PORT=587
SMTP_LOGIN=notifications
SMTP_PASSWORD=AReallyGo0Dpas$word
SMTP_FROM_ADDRESS=notifications@mastodon.spotcheckit.org
```

Step 15:

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

Step 16:

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

Step 17:

Add reverse proxy to apaches configuration scan,then rebuild and redstart Easy apache web service.  
```
/scripts/ensure_vhost_includes --user=dyount
/usr/local/cpanel/scripts/rebuildhttpdconf
/usr/local/cpanel/scripts/restartsrv_httpd
```


Step 18:

Now we will change the permissions of the Mastodon install to the cPanel account owner. 
```
cd /home/dyount/mastodon.spotcheckit.org/
chown -R dyount. live
```

At this point you should be able to go to the URL in your browser setup for Mastodon and it will be ready to login and setup.

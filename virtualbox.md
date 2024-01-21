## Glossary
**Host** -- your workstation, hardware computer you are physically working

**Guest** -- a virtual machine running in VirtualBox on your host computer

This ‘how to’ assumes Ubuntu host system. 

## Set up your host machine

### Get & install VirtualBox and Vagrant:
[https://www.virtualbox.org/wiki/Linux_Downloads](https://www.virtualbox.org/wiki/Linux_Downloads)

[https://www.vagrantup.com/downloads.html](https://www.vagrantup.com/downloads.html)

Install both:

`sudo dpkg -i VirtualBox-yourdistronumber.deb`

`sudo dpkg -i vagrant-yourdistronumber.deb`

Download image:

`vagrant box add ubuntu/xenial64`

`vagrant box list`

Next:

`mkdir -p ~/project/project_name`

`cd ~/project/project_name`

`git clone https://github.com/project_name/repo_name.git`

Appears ~/project/project_name/repo_name with site code

`mkdir -p ~/project/project_name/virtualbox`

`mkdir -p ~/project/project_name/aux`

`cd ~/project/project_name/virtualbox`

`vagrant init ubuntu/xenial64`

Open file Vagrantfile and edit it (uncomment or add lines):

Set IP-address for guest machine. Set IP as you wish in 192.168.0.1/32 network. Remember it.

`config.vm.network "private_network", ip: "192.168.66.99"`

Map two host folders to guest machine. First contains our site, second services as auxiliary point for file swapping between both machines.

`config.vm.synced_folder "~/project/project_name/repo_name", "/home/vagrant/project"`

`config.vm.synced_folder "~/project/project_name/aux", "/home/vagrant/aux"`

`config.vm.synced_folder "~/project/project_name/repo_name/data", "/home/vagrant/project/data", :mount_options => ["dmode=777","fmode=777"]`

Save Vagrantfile.

Set up DNS:

`sudo vim /etc/hosts`

Scroll to end and add string:

`192.168.66.99 your_super_project.dev www.your_super_project.dev`

Save /etc/hosts

`cd ~/project/project_name/virtualbox`

`vagrant up`

`vagrant ssh`

You have logged on guest machine as user vagrant who can sudo without password.

## Set up guest machine

`sudo su`

`apt update && apt upgrade`

`apt install vim mc htop apache2`

`sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'`

`wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -`

`add-apt-repository ppa:ondrej/php`

`apt update && apt upgrade`

Now we can install any of php5.6 - 7.1 and postgresql 9.x .
For our deal we choose php5.6 and postgresql 9.4 .

`apt install postgresql-9.4 php5.6 php5.6-cli php-imagick php5.6-pgsql php5.6-xml php5.6-curl php5.6-xdebug`

### Set up Apache
Apache setting is quite straightforward:

/etc/apache2/conf-available/project_name.conf
```
<VirtualHost *:80>
    DocumentRoot "/home/vagrant/project"
    ServerName your_super_project.dev
    ServerAlias www.your_super_project.dev
    <Directory /home/vagrant/project>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

`sudo a2enconf project_name.conf`

`sudo a2enmod rewrite`

`sudo mkdir -p /var/www/uploads && chmod 777 /var/www/uploads`

`sudo service apache2 restart`

### Set up PostgreSQL

* Log on dev server and get dump (it may take a while and couple of gigabytes):

`pg_dump -Ual_web_user -hlocalhost -dyour_super_project -W | gzip > your_super_project.sql.gz`

* Download dump on host machine via scp:

`scp -i your_rsa_key your_login@dev.project_name.com:/path/to/file/on/server ~/project/project_name/aux`

* Set up guest machine:

`sudo -u postgres psql`

`create user "al_web_user" with password 'your_super_password';`

`create database your_super_project;`

`grant all privileges on database your_super_project to al_web_user;`

`grant all privileges ON all tables in schema public TO al_web_user;`

`grant all privileges on all sequences in schema public to al_web_user;`

`grant all privileges on all functions in schema public to al_web_user;`

`grant create on schema public TO al_web_user;`

`\q`

* Fill database with dump:

`gunzip -c /home/vagrant/aux/your_super_project.sql.gz | psql -Ual_web_user -hlocalhost -dyour_super_project -W`

* Open database for external connect (i.e. pgadmin3)

In /etc/postgresql/9.4/main/pg_hba.conf

`host all all 192.168.66.1/24 md5`

In /etc/postgresql/9.4/main/postgresql.conf

`listen_addresses = '*'`

`sudo service postgresql restart`


### Set up Xdebug
#### Configure xdebug.ini on guest
(/etc/php/php5.6/mods-available/xdebug.ini)
Add:

```
xdebug.remote_enable=on
xdebug.remote_log=/var/log/xdebug.log
xdebug.remote_handler=dbgp
xdebug.remote_port=9123
xdebug.collect_params=1
xdebug.remote_connect_back=Off
xdebug.remote_host=192.168.66.1
xdebug.default_enable=1
xdebug.profiler_enable_trigger=1
xdebug.profiler_output_dir=/var/www
xdebug.remote_autostart=1
xdebug.idekey=PHPSTORM
xdebug.show_local_vars=1
```

enable mod (`ln -s` from mods-available to apache2, cli or fpm), restart php and apache/nginx

#### Configure PHPStorm
**Settings > Languages & Framework > PHP > Debug :**

Ignore external connections through unregistered server configurations:	False

Detect path mappings from deployment configurations:	True

Max. simultaneous connections:	3

Xdebug port:	9123

Can accept external connections:	True

**Settings > Languages & Framework > PHP > Servers**

Host :  192.168.66.99 or your_super_project.dev

Mappings:

File/Directory: ~/project/project_name/repo_name	

Absolute path on the server: /home/vagrant/project

**Run > Edit Configurations... > New**

Set Servers and Ide key

# Installing third party components:

Install soap support
`apt-get install php5.6-soap'
or
`yum install php55-php-soap`


Install PDF command line tools:

`yum install poppler-utils`

Install GD-library support for PHP:

`yum install php55-gd`
or
`apt-get install php5.6-gd`

Dependencies for /usr/bin/sendEmail script:

`apt-get install libnet-ssleay-perl libio-socket-ssl-perl`

Install IMAGEMAGIK-library support for PHP and command line tools:

`apt-get install imagemagick php5-imagick`

Comment
`<policy domain="coder" rights ="none" pattern="LABEL">`
in /etc/ImageMagick/policy.xml 

Install the following to be able to download photos  in one ZIP archive

`sudo apt-get install php5.6-zip`

## Install ImageMagick PHP Plugin

For PHP5.5 commands are the following
* yum install gcc
* yum install php55-devel.x86_64
* yum install php55-php-pear.noarch
* yum install ImageMagick ImageMagick-devel

If "pecl" was installed via php55-php-pear.noarch package try running

`yum install --enablerepo remi php-pear php-devel`

* pecl install imagick
* echo "extension=imagick.so" > /etc/php.d/imagick.ini #Make sure this is correct plugins folder
* service httpd restart

## Install Wkhtmltopdf

```
cd /tmp/
wget https://downloads.wkhtmltopdf.org/0.12/0.12.4/wkhtmltox-0.12.4_linux-generic-amd64.tar.xz
sudo tar -xf wkhtmltox-0.12.4_linux-generic-amd64.tar.xz -C /usr --strip-components=1
rm wkhtmltox-0.12.4_linux-generic-amd64.tar.xz`
```

Install xvfb-run

```
sudo apt install xvfb
```

Some wkhtmltopdf features require additional libraries. All of them can be installed with following command

```
sudo apt-get install zlib1g-dev libfontconfig1-dev fontconfig libfreetype6-dev libx11-dev libxext-dev libxrender-dev xpdf
```

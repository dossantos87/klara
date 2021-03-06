# Installing Klara

## Requirements for running Klara:

- GNU/Linux (we recommend Ubuntu 16.04 or latest LTS)
- MySQL / MariaDB DB
- Python 2.7
- Python virtualenv package
- Yara (installed on workers)

Installing Klara consists of 4 parts:

- Database installation
- Worker installation
- Dispatcher installation
- Web interface installation

Components are connected between themselves as follows:

```
                              +----------+          +----------------+
                              |          |          |                |
                  +---------->+ Database +<--+      |     nginx      |
                  |           |          |   |      |   (optional)   |
                  |           +----------+   |      |                |   
           +------+------+                   |      +-------+--------+   
           |             |                   |              |             
    +----->|  Dispatcher | <---+             |              |            
    |      |             |     |             |              |            
    |      +------+------+     |             |              v            
    |             |            |             |      +-------+--------+
    |             |            |             |      |                |
    |             |            |             |      |                |
+---+----+   +----+---+   +----+---+         ^------+   Web server   |
|        |   |        |   |        |                |                |
| Worker |   | Worker |   | Worker |                |                |
|        |   |        |   |        |                +----------------+
+--------+   +--------+   +--------+


```
Workers connect to Dispatcher using a simple HTTP REST API. Dispatcher and the Web server 
connect the MySQL / MariaDB Database using TCP connections. Because of this, components can be installed on 
separated machines / VMs. The only requirements is that TCP connections are allowed between them.

# Installing on Windows

Since entire project is written in Python, Dispatcher and Workers can be set up to run in an Windows environment. Unfortunately, as we only support Ubuntu, instructions will be provided for this platforms, but other GNU/Linux flavours should be able to easily install Klara as well.

## Database installation

Please refer to these [instructions](database.md)

## Dispatcher installation

Install the packages needed to run Dispatcher:
```
sudo apt -y install python-virtualenv libmysqlclient-dev python-dev git
```

We recommend running dispatcher using a non-privileged user. Create an user which will be responsible to run the worker and the master:

```
sudo groupadd -g 500 projects
sudo useradd -m -u 500 -g projects projects
```

Create a folder needed to store all the Klara project files:

```
sudo mkdir /var/projects/
sudo chown projects:projects /var/projects/
```

Substitute to projects user and create the virtual env + folders needed to run the Dispatcher:

```
su projects
mkdir /var/projects/klara/ -p
# Create the virtual-env
virtualenv ~/.virtualenvs/klara
```
From now one, all commands will be run under `projects` user

Clone the repository:

```
git clone git@github.com:kasperskylab/klara.git ~/klara-github-repo
```

Copy Dispatcher's files to the newly created folder and install python dependencies:
```
cp -R ~/klara-github-repo/dispatcher /var/projects/klara/dispatcher/
cd /var/projects/klara/dispatcher/
cp config-sample.py config.py
pip install -r ~/klara-github-repo/install/requirements.txt
```

Now fill in the settings in config.py:

```
# Setup the loglevel
logging_level  = "debug"

# What port should Dispatchers's REST API should listen to
listen_port = 8888

# Main settings for the master
# Set debug lvl
logging_level  = "debug"

# What port should the server listen to
listen_port = 8888

# Notification settings
# Do we want to sent out notification e-mails or not?
notification_email_enabled  = True
# FROM SMTP header settings
notification_email_from     = "klara-notify@example.com"
# SMTP server address
notification_email_smtp_srv = "127.0.0.1"

# MySQL / MariaDB settings for the Dispatcher to connect to the DB
mysql_host      = "127.0.0.1"
mysql_database  = "kl-klara"
mysql_user      = "root"
mysql_password  = ""
```
Once the settings are set, you can check Dispatcher is working by running these commands
```
sudo su projects
# We want to enable the virtualenv
source  ~/klara-github-repo/install/activate.sh
cd /var/projects/klara/dispatcher/
./klara-dispatcher
```
If everything went well, you should see:
```
[01/01/1977 13:37:00 AM][INFO]  ###### Starting KLara Job Dispatcher ######
```

In order to start Dispatcher automatically at boot, please check [Supervisor installation](supervisor.md)

Next step would be starting Dispatcher from supervisor:
```
sudo supervisorctl start klara_dispatcher start
```


# Worker installation


Install the packages needed to run Worker:
```
sudo apt -y install python-virtualenv python-dev git
```

We recommend running Worker using a non-privileged user. Create an user which will be responsible to run the worker and the master:

```
sudo groupadd -g 500 projects
sudo useradd -m -u 500 -g projects projects
```

Create a folder needed to store all the Klara project files:

```
sudo mkdir /var/projects/
sudo chown projects:projects /var/projects/
```

Substitute to projects user and create the virtual env + folders needed to run the Worker:

```
su projects
mkdir /var/projects/klara/ -p
# Create the virtual-env
virtualenv ~/.virtualenvs/klara
```
From now one, all commands will be run under `projects` user

Clone the repository:

```
git clone git@github.com:kasperskylab/klara.git ~/klara-github-repo
```

Copy Worker's files to the newly created folder and install python dependencies:
```
cp -R ~/klara-github-repo/worker /var/projects/klara/worker/
cd /var/projects/klara/worker/
cp config-sample.py config.py
pip install -r ~/klara-github-repo/install/requirements.txt
```

Now fill in the settings in config.py:

```
# Setup the loglevel
logging_level  = "debug"

# Api location for Dispatcher. No trailing slash !!
# Dispatcher is exposing the API at "/api/" location
api_location = "http://127.0.0.1:8888/api"
# The API key set up in the `agents` SQL table
api_key      = "test"

# Specify worker refresh time in seconds
refresh_new_jobs    = 60

# Yara settings
# Set-up path for Yara binary
yara_path           = "/opt/yara-latest/bin/yara"
# Use 8 threads to scan and scan dirs recursively
yara_extra_args     = "-p 8 -r"
# Where to store Yara temp results file
yara_temp_dir       = "/tmp/"

# md5sum settings
# binary location
md5sum_path         = "/usr/bin/md5sum"

# tail settings
# We only want the first 1k results
head_path_and_args  = ["/usr/bin/head", "-1000"]

# Virus collection should NOT have a trailing slash !!
virus_collection                = "/var/projects/klara/repository"
virus_collection_control_file   = "repository_control.txt"
```
Once the settings are set, you can check Worker is working by running these commands
```
sudo su projects
# We want to enable the virtualenv
source  ~/klara-github-repo/install/activate.sh
cd /var/projects/klara/worker/
./klara-worker
```
If everything went well, you should see:
```
[01/01/1977 13:37:00 AM][INFO]  ###### Starting KLara Worker ######
```

In order to start Worker automatically at boot, please check [Supervisor installation](supervisor.md)

Next step would be starting Worker from supervisor:
```
sudo supervisorctl start klara_worker start
```


## Installing Yara on worker machines

Install the required rependencies
```
sudo apt -y install libtool automake libjansson-dev libmagic-dev libssl-dev build-essential

# Get the latest stable version of yara from https://github.com/virustotal/yara/releases
# Usually it's good practice to check the hash of the archive you download, but here we can't, since it's from GH
# 
wget https://github.com/VirusTotal/yara/archive/vx.x.x.tar.gz
cd yara-3.x.0
./bootstrap.sh

./configure --prefix=/opt/yara-x.x.x --enable-cuckoo --enable-magic --enable-address-sanitizer --enable-dotnet
make -j4
sudo make install
```

Now you should have the chosen yara version installed on `/opt/yara-x.x.x/`

Create a symlink to the latest version, so when we update Yara, workers don't have to be reconfigured:
```
# Make a symlink to the actual folder
cd /opt/
ln -s yara-3.x.x/ yara-latest
```

## Setting up worker's scan repositories (virus collections)

Each time workers contact the Dispatcher in order to check if there are new jobs, they will check if they can execute these jobs. 
Basically, if a new job to scan `/mach-o_collection` can be picked up by a free worker, with the following `config.py` settings:

```
virus_collection                = "/var/projects/klara/repository"
virus_collection_control_file   = "repository_control.txt"
``` 
it will check if it has the following file/folders:
```
/var/projects/klara/repository/mach-o_collection/repository_control.txt
```

If this file and the corresponding folders exist, then the Worker will accept the job and start the yara scan with the specified job rules searching files in parent folder
`/var/projects/klara/repository/mach-o_collection/`

# Web interface installation

Requirements for installing the web interface are:

- web server running at least PHP 5.6
- the following php5 extensions should be installed:

```
apt install php7.0-fpm php7.0 php7.0-mysqli php7.0-curl php7.0-gd php7.0-intl php-pear php-imagick php7.0-imap php7.0-mcrypt php-memcache  php7.0-pspell php7.0-recode php7.0-sqlite3 php7.0-tidy php7.0-xmlrpc php7.0-xsl php7.0-mbstring php-gettext php-apcu
```

Once you have this installed, copy the `/web/` folder to the HTTP server document root. Update and rename the following sample files: 

- `application/config/config.sample.php` -> `application/config/config.php`
- `application/config/project_settings.sample.php` -> `application/config/project_settings.php`

You must configure the `base_url`, `encryption_key` from `config.php` as well as other settings in `database.php`.
More info about this here:

- https://www.codeigniter.com/user_guide/installation/upgrade_303.html
- https://codeigniter.com/user_guide/libraries/encryption.html
- https://www.codeigniter.com/user_guide/database/configuration.html


======

That's it! If you have any issues with installing this software, please submit a bug report. Happy hunting!



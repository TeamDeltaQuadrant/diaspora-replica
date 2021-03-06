# diaspora*-replica

## Table of contents

1. [Overview - What is diaspora*-replica?](#overview)
2. [Deploy diaspora* on local environment](#deploy-diaspora-on-local-environment)
3. [Deploy diaspora* on production environment](#deploy-diaspora-on-production-environment)
4. [Using PostgreSQL Database](#using-postgresql-database)
5. [How to set up a Development Environment](#how-to-set-up-a-development-environment)
6. [How to upgrade diaspora*-replica](#how-to-upgrade-diaspora-replica)
7. [Which Operating Systems are supported?](#which-operating-systems-are-supported)
8. [How to contribute this project](#how-to-contribute-this-project)

## Overview

The aim of this project is to provide some tools that can help you to deploy [diaspora*] POD through the automation of two tasks:

* The deploy and configuration of the machine with [Vagrant 2] (1.6.x) and [Puppet]
* The deploy of Diaspora* itself with [Capistrano 3] (3.1)

With these two tasks you can automatically set up different environments, from development to production installation.

### Configure FQDNs in your system

Vagrantfile, Puppet and Capistrano are already configured to handle three kind of environment: ``development``, ``staging`` and ``production``. If you want to try them you have to update ``/etc/hosts`` file, adding to it the three FQDN for the local diaspora* installation.

Put these entries in your ``/etc/hosts``
```
192.168.11.2    development.diaspora.local
192.168.11.3    staging.diaspora.local
192.168.11.4    production.diaspora.local
```

### Initialize project

```
git clone https://github.com/joebew42/diaspora-replica.git
cd diaspora-replica
git submodule update --init
```

## Deploy diaspora* on local environment

If you want to try diaspora* without messing up your computer by installing and configuring extra packages, you can set up a virtual machine that is executed by Vagrant and then automatically configured by Puppet.
Now that you have a fully configured virtual machine ready to host a diaspora application, will be very easy to deploy it with Capistrano.

### Set up the virtual machine with Vagrant/Puppet

```
vagrant up production
```
Wait until the virtual machine is automatically setted up with puppet and is up and running.

### Install Capistrano with bundle (if you haven't)

If you have not installed Capistrano on your computer, you can easily run bundle to install it.

```
cd capistrano/ && bundle
```

### Deploy diaspora* with Capistrano

When the virtual machine is up and running, then you can deploy diaspora* on it using Capistrano

```
cd capistrano
cap production deploy
```

When executed the first time, this step can take several minutes (about 20, based on your internet connection), because the diaspora git repository must be cloned and the bundler will install ruby gems.
Once capistrano completed the deploy task, you can start diaspora through ``foreman``

```
cap production foreman:start
```

Wait until unicorn workers are ready (about 30/40 seconds) and then your diaspora* installation will be up and running at ``http://production.diaspora.local``

### Start, stop and restart

You can use Capistrano tasks to start, stop or restart diaspora*

```
cap production foreman:start
cap production foreman:stop
cap production foreman:restart
```

### Deploy other branches

If you want to deploy a different branch of diaspora* (ex. ``develop`` instead of ``master``) you have to update `puppet/manifest/site.pp` by specifying correct `rvm` and `ruby` version:

```puppet
node 'production.diaspora.local' {
  class { 'diaspora':
    hostname            => $fqdn,
    environment         => 'production',
    rvm_version         => '1.26.3',
    ruby_version        => '2.1.5',
    app_directory       => '/home/diaspora',
    user                => 'diaspora',
    group               => 'diaspora',
    db_provider         => 'mysql',
    db_host             => 'localhost',
    db_port             => '3306',
    db_name             => 'diaspora_production',
    db_username         => 'diaspora',
    db_password         => 'diaspora',
    db_root_password    => 'diaspora_root',
    unicorn_worker      => 4,
    sidekiq_concurrency => 5,
    sidekiq_retry       => 10,
    sidekiq_namespace   => 'diaspora',
    foreman_web         => 1,
    foreman_sidekiq     => 1
  }
}
```

Set up the the `repo_url` and/or the `branch` that we want to deploy by editing `capistrano/config/deploy/production.rb`

```ruby
...
set :repo_url, 'https://github.com/diaspora/diaspora.git'
set :branch, 'develop'
...
```

Execute the provision of the machine and the deploy of diaspora*

```
vagrant up production
cd capistrano/
cap production deploy
cap production foreman:start
```

Check out your diaspora* installation at `http://production.diaspora.local`

## Deploy diaspora* on production environment

If you want to use these tools to deploy a production installation (e.g. staging or production), you have to configure some properties inside ``Vagrantfile``, ``puppet/manifests/site.pp``, ``capistrano/config/deploy/production.rb`` and of course, SSL certs and private/public keys for the server.

### Vagrantfile

You have to configure your ``Vagrantfile`` based on the virtual machine provider you are going to use (e.g. Amazon AWS, DigitalOcean, and other). Please see the [Vagrant Provider Documentation] for detailed instructions. If you are not going to use vagrant you can skip this section and apply puppet manually, or configure a puppet master/agent environment. See the Puppet documentation for more informations.

### puppets/manifests/site.pp

```puppet
node 'myproduction.domain.com' {
  class { 'diaspora':
    hostname            => $fqdn,
    environment         => 'production',
    rvm_version         => '1.26.3',
    ruby_version        => '2.1.5',
    app_directory       => '/home/diaspora',
    user                => 'diaspora',
    group               => 'diaspora',
    db_provider         => 'mysql',
    db_host             => 'localhost',
    db_port             => '3306',
    db_name             => 'diaspora_production',
    db_username         => 'diaspora',
    db_password         => 'diaspora',
    db_root_password    => 'diaspora_root',
    unicorn_worker      => 4,
    sidekiq_concurrency => 5,
    sidekiq_retry       => 10,
    sidekiq_namespace   => 'diaspora',
    foreman_web         => 1,
    foreman_sidekiq     => 1
  }
}
```
Of course, you have to change *myproduction.domain.com* with your real Fully Qualified Domain Name, and adjust settings, if needed.

### Setup the SSL certificate for your server

You have to put the SSL key and certificate in ``puppet/modules/diaspora/files/certs/``. The file names must contain the FQDN followed by .crt and .key. See the examples that already exists.

### Setup the public key of the user

Put in ``puppet/modules/diaspora/files/diaspora.pub`` the public key of the user that will be granted to execute commands from Capistrano.

### Apply Puppet configuration

Now that your Puppet configuration is complete, you have to execute it to your production server. If you use vagrant configured with one of the supported providers it can be done automatically. If you are not able to configure vagrant, you can apply puppet in other ways. But this topic will be not covered here. See the Puppet documentation for this.

### capistrano/config/deploy/production.rb

Here you have to configure the FQDN, the git repository URL, the name of the branch and the user of the remote server.

### Capistrano public key

In order to allow Capistrano to execute commands on the remote server you need to put in ``capistrano/ssh_keys`` the private and the public keys of the user. The public key should be the same of ``puppet/modules/diaspora/files/diaspora.pub``.

### Deploy diaspora* with Capistrano

Once you have successfully configured the server, you can deploy and start diaspora*

```
cd capistrano
cap production deploy foreman:start
```

## Using PostgreSQL Database

If you want to use PostgreSQL [1] instead of the default MySQL, you can configure it through ``puppet/manifests/site.pp``:

```puppet
node 'development.diaspora.local' {
  class { 'diaspora':
    hostname         => $fqdn,
    environment      => 'development',
    rvm_version      => '1.26.3',
    ruby_version     => '2.1.5',
    app_directory    => '/home/diaspora',
    user             => 'diaspora',
    group            => 'diaspora',
    db_provider      => 'postgres',
    db_host          => 'localhost',
    db_port          => '5432',
    db_name          => 'diaspora_development',
    db_username      => 'diaspora',
    db_password      => 'diaspora',
    db_root_password => 'diaspora_root'
  }
}
```
note the `db_provider` and `db_port` parameters.

And you have to uncomment the line:

```ruby
# set :default_env, { DB: 'postgres' }
```
That is present in your ``capistrano/config/deploy/development.rb``

[1] Puppet will install PostgreSQL 9.1

### PostgreSQL for Staging/Production environments

Because of "--deployment" flag that is set up by default in capistrano bundler, it is necessary to fork diaspora* in a personal git repository and bundle it with PostgreSQL support:

```bash
$ DB=postgres bundle
```

and then add the generated Gemfile.lock under version control. Once you have done that, to enable PostgreSQL you have to uncomment this line,:

```ruby
# set :default_env, { DB: 'postgres' }
```

in ``capistrano/config/deploy/production.rb`` (or ``capistrano/config/deploy/staging.rb``, depends on which stage you are going to deploy.) Of course, you have to specify your git repository, too.

## How to set up a Development Environment

You can use these tools to easly set up a fully development environment for diaspora*. The ``development`` machine is configured within ``Vagrantfile`` with enough RAM (2GB) to run all tests.
In this way you can write code using your preferred IDE or editor (``vim``, ``emacs``, ``eclipse`` and so on) directly from your local environment (the host machine), by executing tests within the ``development`` virtual machine.

### Cloning your git repo to you host

``Vagrantfile`` is configured to sync an host directory (``src``) with a guest directory (``diaspora_src``), for better I/O performance read the [Vagrant Synced Folder Documentation]. The first step is to clone your own diaspora* git repository into the local directory ``src``.

```
cd diaspora_replica
git clone your_own_diapora_git_repo src
```

### Run development virtual machine

```
vagrant up development
```

### Enabling vagrant user to use rvm (first time set up)

```
vagrant ssh development
vagrant@development:~$ sudo usermod -aG rvm vagrant && newgrp rvm
vagrant@development:~$ /bin/bash --login
```

### Prepare the Rails application

```
vagrant@development:~$ cd diaspora_src
```

Prepare your configuration files ``diaspora.yml`` and ``database.yml``, put it into ``config/`` directory.

### Configure Rubies and Gemsets

```
vagrant@development:~$ rvm use 2.1
vagrant@development:~$ rvm gemset create diaspora_dev
vagrant@development:~$ rvm gemset use diaspora_dev
```

### Install gems and create databases

```
vagrant@development:~$ bundle
vagrant@development:~$ rake generate:secret_token
vagrant@development:~$ rake db:create
vagrant@development:~$ rake db:migrate
```

### Run all tests

```
vagrant@development:~$ rspec
```

## How to upgrade diaspora*-replica

Follow these steps in order to upgrade an existing installation of diaspora*. In this example we'll consider the ``production`` environment (you can apply the same instructions for ``staging``, ``development`` or user defined environment).

```
cd diaspora-replica/
git pull --rebase origin master
git submodule update

vagrant up production
vagrant provision production --provision-with puppet

cd capistrano/
cap production deploy
cap production foreman:restart
```

## Which Operating Systems are supported?

* Ubuntu 14.04LTS Server
* Ubuntu 12.04LTS Server
* CentOS 6.4

### Choosing the OS from Vagrantfile

By default the ``Vagrantfile`` is configured to run an Ubuntu 14.04LTS Server box. If you want to switch to Ubuntu 12.04LTS Server or CentOS 6.4, you have to edit the file.

## How to contribute this project

This project is under development. At the moment the Puppet provides support and, has been tested on Ubuntu 14.04LTS Server, Ubuntu 12.04LTS Server and CentOS 6.4. It could be useful if someone can test it over other version of Ubuntu or CentOS, or provides support for other GNU/Linux distributions.
The Database section of the Puppet does not consider parameters like hostname and port at the moment. Furthermore there a lot of variables of diaspora.yml that are not covered (e.g. mail server configuration, unicorn workers, and more).

  [diaspora*]: https://github.com/diaspora/diaspora
  [Vagrant 2]: http://www.vagrantup.com/
  [Vagrant Provider Documentation]: http://docs.vagrantup.com/v2/providers/index.html
  [Vagrant Synced Folder Documentation]: http://docs.vagrantup.com/v2/synced-folders/nfs.html
  [Puppet]: http://puppetlabs.com/
  [Capistrano 3]: http://www.capistranorb.com/

---
layout: post
title: "Dockerize existing ruby on rails application"
date: 2015-04-20 11:33
comments: true
external-url:
categories:
---

> This article will focus on dockerizing existing rails app because it's common and challenging.
> But as long as a new app will finally become old, you may find some tips here still be helpful.

Say we have an existing ruby on rails project and we decide to somehow run it inside a docker container so we can get those great benefits of Docker ecosystem.

Ready? Let's start.

First we can divide the task into several phases:

- Setup a base image with ruby environment
- Install gems with bundler
- Compile static assets
- Provision app before start
- Start web server

## Setup a base image with ruby environment

Both [Ruby](https://registry.hub.docker.com/_/ruby/) and [Rails](https://registry.hub.docker.com/_/rails/) have offical images available.
But it's also not too hard to build it from scratch.

### Using Rails official images

We don't recommend Rails official images for production use because

- only rails 4+ has offical images
- we can't control ruby version, which brings trouble in case we want to upgrade ruby for vulnerability issues
- rails could simply be installed by ```bundle install```

### Using Ruby offcial images

Ruby official images are turned to be helpful, they offer options from down to ruby 1.9.3 and up to latest stable version.
If you're lucky, your target ruby is one of the official versions, it could be the best choice to start from a official ruby image.

### Build base Ruby image from scratch

But if you're not so lucky, like our case, we have legacy rails apps running on ree-1.8.7, so there's no way except building a base ruby image by ourselves.

We've tried both [RVM](http://rvm.io/) and [ruby-build](https://github.com/sstephenson/ruby-build), and we finally chose ruby-build for its simplicity.

> Note ree-1.8.7 need some patches to make it work correctly, create a directory called patch, and put [ree-1.8.7-2011.12](https://gist.githubusercontent.com/bencao/32bf9c69c606e2b58666/raw/ab232ae9992cc026e393b9c8825c6c390dc29e92/ree-1.8.7-2011.12) inside of it.

Here's the Dockerfile demonstrating how we build it:

```shell
FROM centos:7

# install those basic tools we will use in debugging
RUN yum install -y \
  git \
  tar vim \
  gcc gcc-c++ make patch \
  hostname nmap-ncat readline-devel; \
  yum -y clean all

# install ruby build
RUN git clone https://github.com/sstephenson/ruby-build.git /root/ruby-build && /root/ruby-build/install.sh

# install ree, gcc44 is a must-have
RUN yum install -y compat-gcc-44; yum -y clean all
ADD ./patch/* /usr/local/share/ruby-build/
RUN CC=gcc44 ruby-build ree-1.8.7-2011.12 /ruby
RUN echo "export PATH=$PATH:/ruby/bin" > /etc/profile.d/ruby.sh
ENV PATH $PATH:/ruby/bin

# use taobao gem source, as in China rubygems.org has been blocked
RUN gem sources --remove https://rubygems.org/ && gem sources --add https://ruby.taobao.org/

# install bundler
RUN gem install -N bundler
```

Then build image with

```shell
docker build -t my-company/ruby-base .
```

## Install gems with bundler

### Install System Libraries

Before installing gems, we should first install those system libraries that we need for some gems to compile to native extensions.
For example, we need to install ```mysql-devel``` package manually in Centos before we install ```mysql2``` gem.

### Bundle Install

Simply run a bundle install in Dockerfile is a solution. But it has a drawback. See this typical example:

```shell
FROM my-company/ruby-base:latest

ADD . /my-app
WORKDIR /my-app

RUN bundle install --jobs 3 --retry 3
RUN bundle clean --force
```

Because we add project files first, the ADD command invalidate the Docker cache, then bundle install can be a expensive command, in our case without any optimization this take more than 10 minutes.

How could we make it faster?

We could also separate the cost into 2 parts:

- Time takes for downloading Gems from remote gem servers
- Time takes for compiling gems with native extensions

Let's deal with them one by one.

#### Reduce downloading time

We've tried many means and finally we find this solution being helpful and more importantly being elegant.

```shell
# this is a command line context, it could be a Jenkins Job or our local terminal
# in the same directory we placed the Dockerfile above
bundle package --all
docker build -t my-company/incredible-app
```

After ```bundle package --all```, gem sources will be cached in ```vendor/cache``` directory, and during docker build we will spend no time downloading it from remote server.
For the first time, ```bundle package --all``` could be slow, but from second time it will be very fast.
Which means we have the same solution in local and in CI environment, and there's no any invasion into Dockerfile.
Beatuiful, just like the progressive enhancement idea that was once famous in frontend world.

#### Reduce compiling time

It's easy to see which extension takes the most time. Considering Gems should be quite stable, we could introduce a new image layer which install some gems in advance. Here's our example:

```shell
FROM my-company/ruby-base:latest

# only purpose for this image is to speed up normal build

# save 3 min
RUN gem install -N nokogiri -v 1.6.2.1

# save 1min10s
RUN gem install -N curb -v 0.8.5

# save around 30s
RUN gem install -N poltergeist -v 1.5.1

# save more than 30s
RUN gem install -N unicorn -v 4.8.3

# save 16s
RUN gem install -N therubyracer -v 0.12.1

# save 10s
RUN gem install -N oj -v 2.12.0

# save 10s
RUN gem install -N mysql2 -v 0.3.18

# save 10s
RUN gem install -N ruby-prof -v 0.15.2
```

And some slightly modification is needed for app's Dockerfile

```shell
# update FROM section to point to the image created above
FROM my-company/ruby-incredible-app-base:latest
```

## Compile static assets

This step is quite simple, just add a ```RUN rake assets:precompile``` and it will work.

## Provision the app before start

### Generate Config Files

Config files are prerequisite for rails application.
Pass them in via volume is an option, but if you have ever agreed [Twelve Factor](http://12factor.net/) you may prefer the environment variables option.
We want to discuss how to generate config files given injected ENV variables.

Let's think about it deeper.

1. The generation will happen in container runtime, which limit the possible dealing points in ```CMD``` and ```ENTRYPOINT``` instruction.
2. The main difference between ```CMD``` and ```ENTRYPOINT``` is ```CMD``` is very likely to be overriden in ```docker run``` command.
3. So it depends on whether we want the config files to be generated if command is overridden?

For our case, we want to generate config files even if other commands, such as ```rspec spec``` is given.
So we chose ```ENTRYPOINT``` as a point to generate config files.

Here's our sample docker-entrypoint.sh

```shell
#!/bin/bash

# stop execution if any commands fail
set -e

# generate database.yml
source /my-app/docker-initializers/generate_database_yml.sh /my-app/config/database.yml

exec "$@"

```

docker-initializers/generate_database_yml.sh

```shell
#!/bin/bash

cat << EOF > $1
defaults: &defaults
  adapter: mysql2
  reconnect: true
  encoding: utf8
  host: $MYSQL_MY_APP_HOST
  port: $MYSQL_MY_APP_PORT
  username: $MYSQL_MY_APP_USERNAME
  password: $MYSQL_MY_APP_PASSWORD

test:
  <<: *defaults

development:
  <<: *defaults

production:
  <<: *defaults

EOF

```

So from the above samples you could find that we will be dependent on these MYSQL_* parameters.

### Link Support

It's possible users will link the app container with mysql or some other service containers.
In order to support them, we could adapt link ENV variables to previous MYSQL_* parameters.

docker-initializers/link_support.sh

```shell
#!/bin/bash

# --link mysql support
if [ -n "$MYSQL_PORT_3306_TCP_ADDR" ]; then
  export MYSQL_MY_APP_HOST=$MYSQL_PORT_3306_TCP_ADDR
  export MYSQL_MY_APP_PORT=$MYSQL_PORT_3306_TCP_PORT
  export MYSQL_MY_APP_USERNAME=default_user
  export MYSQL_MY_APP_PASSWORD=default_password
fi

```

Adding link support to docker-entrypoint.sh, it now becames:

```shell
#!/bin/bash

# stop execution if any commands fail
set -e

# handling case such as docker run --link mysql-1:mysql
source /my-app/docker-initializers/link_support.sh

# generate database.yml
source /my-app/docker-initializers/generate_database_yml.sh /my-app/config/database.yml

exec "$@"
```

### Wait Support

Now our database.yml will be automatically generated upon start. But there's another problem, if mysql is still initializing while rails tries to connect to it, rails will throw an error about database connection.
The reason is rails need to get the metadata of DB tables to allow ActiveRecord work correctly.
How to deal with this problem?
We could ensure this from outside, but add some protection inside is still harmless. Here's how we do it:

> We've employed ```nmap-ncat``` package in ruby-base image, now it's time for it to shine.

docker-initializers/wait_support.sh

```shell
#!/bin/bash

function wait_for() {
  service=$1
  host=$2
  port=$3

  echo "waiting for $service to be up on $host:$port..."

  if [ -n "$host" -a -n "$port" ]; then
    while ! nc -w 1 -c echo $host $port
    do
      echo -n .
      sleep 1
    done

    echo 'ok'
  else
    echo "[ERROR] invalid host=$host or port=$port for $service"
    exit 1
  fi
}

wait_for "database connection - $MYSQL_MY_APP_DBNAME" $MYSQL_MY_APP_HOST $MYSQL_MY_APP_PORT

```

Adding wait support, docker-entrypoint.sh now becomes:

```shell
#!/bin/bash

# stop execution if any commands fail
set -e

# handling case such as docker run --link mysql-1:mysql
source /my-app/docker-initializers/link_support.sh

# wait for other service ports to be ready, this can be enabled by a environment variable
if [ "$WAIT_FOR_DEPENDED_SERVICES" = "true" ]; then
  source /my-app/docker-initializers/wait_support.sh
fi

# generate database.yml
source /my-app/docker-initializers/generate_database_yml.sh /my-app/config/database.yml

exec "$@"

```


### Default Params Support

In the above section we add a switch for wait support. Only if user set WAIT_FOR_DEPENDED_SERVICES to be true we will enjoy the benefit of wait support.
It's natural to consider adding a default value for this variable. Also for mysql db name we can utilize this design, as following:

docker-initializers/default_env_params.sh

```shell
#!/bin/bash

if [ -z "$MYSQL_MY_APP_DBNAME" ]; then
  export MYSQL_MY_APP_DBNAME=my_app_database
fi

if [ -z "$WAIT_FOR_DEPENDED_SERVICES" ]; then
  export WAIT_FOR_DEPENDED_SERVICES=true
fi

```

Including some simple cleanups, now docker-entrypoint.sh reach its final state:

```shell
#!/bin/bash

# stop execution if any commands fail
set -e

# handling case such as docker run --link mysql-1:mysql
source /my-app/docker-initializers/link_support.sh

# set ENV params if they're not set by users
source /my-app/docker-initializers/default_env_params.sh

# wait for other service ports to be ready
if [ "$WAIT_FOR_DEPENDED_SERVICES" = "true" ]; then
  source /my-app/docker-initializers/wait_support.sh
fi

# generate database.yml
source /my-app/docker-initializers/generate_database_yml.sh /my-app/config/database.yml

# prepare log and tmp directories
mkdir -p /my-app/log
mkdir -p /my-app/tmp
rm -rf /my-app/tmp/*

exec "$@"

```

## Start web server

### Choose a proper web server

Depends on each team's situation, maybe you have some experts about puma, maybe you prefer event machine based implementations like thin.
It's a total freedom to choose whatever we're comfortable to power your rails app.

From our case, we were using unicorn as our web server so we kept using it in dockerized app.
It worth mentioning that unicorn has a fantastic feature that is it can dynamically adjust its worker count in runtime, using process signal like TTIN and TTOU.

Say we have unicorn.conf.rb in our project root, then we can start server in default ```CMD`` command:

```shell
FROM my-company/ruby-incredible-app-base:latest

ADD . /my-app
WORKDIR /my-app

RUN bundle install --jobs 3 --retry 3
RUN bundle clean --force

EXPOSE 3000

ENTRYPOINT ["/my-app/docker_entrypoint.sh"]
CMD ["unicorn_rails", "-l", "3000", "-c", "/my-app/unicorn.conf.rb"]

```

## Conclusion

Some of you may figure out there's still a problem, now it's quite complex for us to start the application.
We need to either offering some MYSQL_* ENV variables or linking our app container to a mysql container.
This is a great question, and it opens the door for container orchestration, which is another fascinating area.

Maybe in the future I will post a blog post about it.
For now, just some quick advices. If you want to simplify local development or small amount deployment, you may be interested in [docker-compose](https://docs.docker.com/compose/).

It's a long run, sincerely thank you for reading till here!

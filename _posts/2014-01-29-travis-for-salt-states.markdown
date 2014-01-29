---
layout: post
title:  "Travis CI for Salt States: Part I"
date:   2014-01-29
image: http://i.imgur.com/FW9Tjgb.png
description: Get Travis CI to check the validity of your Salt States
categories: [automation, travis, travis ci, configuration management, continuous integration, salt, chef, saltstack, Salt Stack]
---

# Verifying salt states

Verifying salt states works simply if you're not using any amount of templating (jinja, etc.), grains, or pillar data. Just run the states through a YAML validation tool. For everything else (read as everything), you need to mirror your production environment via vagrant, Jenkins, or some other CI tool.

What if we could do some light validation (or more) on travis CI?

# Validating Salt States on Travis CI

To get setup on Travis, we simply need to install salt, set up some light configuration, and use `salt-call --local` to run salt modules.

This configuration requires two files:

* .travis/minion
* .travis.yml

Let's take these apart piece by piece. You can also go straight to the travis.yml.

## .travis/minion

```
file_client: local
file_roots:
  base:
    - /srv/salt/states
```

## .travis.yml piece by piece

### Declare the language
In this example, we'll use python for our travis box(semi-arbitrarily chosen because [nbviewer](http://nbviewer.ipython.org) and salt run on python). Later, I'd like to see if adding salt as a base image on travis could be [possible](https://github.com/travis-ci/travis-ci/issues/1549) (or feasible). Hit me up if you'd like to see this and use this yourself.

```
language: python
python:
- '2.7'
```

## Perform updates and install salt master

```
before_install:
  - sudo apt-get update
  - curl -L http://bootstrap.saltstack.org | sudo sh -s -- git develop
```

# "Install" the states

We'll copy `.travis/minion` over to `/etc/salt/minion`, copy the states over, restart, and spit out some debugging information.

```
install:
  # Copy these states
  - sudo mkdir -p /srv/salt/states
  - sudo cp -r . /srv/salt/states
  - sudo cp .travis/minion /etc/salt/minion
  - sudo service salt-minion restart

  # Additional debug help
  - sudo cat /var/log/salt/*

  # See what kind of travis box you're on
  - sudo salt-call grains.items --local
```

# High state

```
script:
  - sudo salt-call state.show_highstate --local
  - sudo salt-call state.show_lowstate --local
  - sudo salt-call state.highstate --local
```


# Sample runs


At least for the salt states that [the IPython Notebook Viewer](http://nbviewer.ipython.org) is using, it makes sense for us to be able to test the validity continuously, as 

1. Anyone can submit a pull request
2. These states are pulled via Salt's gitfs backend
3. They're used in production (along with some hidden magic)


```
# Since salt/salt states aren't a language proper, this is a hackaround for now
# I could probably add this to travis later.
# https://github.com/travis-ci/travis-ci/issues/1549
language: python
python:
- '2.7'

before_install:
  - sudo apt-get update
  # Install salt master and salt minion on the same box
  - curl -L http://bootstrap.saltstack.org | sudo sh -s -- git develop

install:
  # Copy these states
  - sudo mkdir -p /srv/salt/states
  - sudo cp -r . /srv/salt/states
  - sudo cp .travis/minion /etc/salt/minion
  - sudo service salt-minion restart
  # Additional debug help
  - sudo cat /var/log/salt/*
  # See what's available on travis
  - sudo salt-call grains.items --local

#script: sudo salt '*' state.highstate
script:
  - sudo salt-call state.show_highstate --local
  - sudo salt-call state.show_lowstate --local
  #- sudo salt-call state.highstate --local
  #- wget 127.0.0.1:8888
```

# The Future

It would be really nice if these can be r


Since most salt states use a templating system (jinja by default), the yaml can't be verified directly
Verifying

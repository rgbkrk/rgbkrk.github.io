---
layout: post
title:  "Travis CI for Salt States: Part I"
date:   2014-01-29
image: http://i.imgur.com/FW9Tjgb.png
description: Get Travis CI to check the validity of your Salt States
categories: [automation, travis, travis ci, configuration management, continuous integration, salt, chef, saltstack, Salt Stack]
---

# Verifying salt states

At least for the salt states that [the IPython Notebook Viewer](http://nbviewer.ipython.org) is using, it makes sense for us to be able to test the validity continuously, as 

1. Anyone can submit a pull request
2. These states are pulled via Salt's gitfs backend
3. They're used in production (along with some other states + pillar data)

Verifying salt states works simply if you're not using any amount of templating (jinja, etc.), grains, or pillar data. Just run the states through a YAML validation tool. For everything else (read as: everything), you need to mirror your production environment via vagrant, Jenkins, or some other CI tool.

What if we could do some light validation (or more) on travis CI?

# Validating Salt States on Travis CI

To get setup on Travis, we simply need to install salt, set up some light configuration, and use `salt-call --local` to run salt modules.

This configuration requires two files:

* .travis/minion
* .travis.yml

Let's take these apart piece by piece. You can also go [straight to the travis.yml]().

## `.travis/minion`

```
file_client: local
file_roots:
  base:
    - /srv/salt/states
```

## `.travis.yml` piece by piece

### Declare the language
In this example, we'll use python for our travis box(semi-arbitrarily chosen because [nbviewer](http://nbviewer.ipython.org) and salt run on python). Later, I'd like to see if adding salt as a base image on travis could be [possible](https://github.com/travis-ci/travis-ci/issues/1549) (or feasible). Hit me up if you'd like to see this and use this yourself.

```
language: python
python:
- '2.7'
```

### Perform updates and install salt master

```
before_install:
  - sudo apt-get update
  - curl -L http://bootstrap.saltstack.org | sudo sh -s -- git develop
```

### "Install" the states

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

### High state

As a first pass, we can run `state.show_highstate`. Since this seems to always return 0, we'll need to make sure this returned appropriately. For now you'll want to check the output. When I figure out an ideal way to verify this, I'll update this post.

```
script:
  - sudo salt-call state.show_highstate --local
```

## `.travis.yml` in whole

```
language: python
python:
- '2.7'

before_install:
  - sudo apt-get update
  - curl -L http://bootstrap.saltstack.org | sudo sh -s -- git develop

install:
  # Copy these states
  - sudo mkdir -p /srv/salt/states
  - sudo cp -r . /srv/salt/states
  - sudo cp .travis/minion /etc/salt/minion
  - sudo service salt-minion restart

  # Additional debug help
  - sudo cat /var/log/salt/*

  # See what kind of travis box you're on
  # to help with making your states compatible with travis
  - sudo salt-call grains.items --local

script:
  - sudo salt-call state.show_highstate --local
```

# Where's the pillar data?

For this "deployment", we're cheating a bit in that there are default values for the pillar data. For something more reasonably complex, you'll want to create some pillar data to use, possibly [encrypting it for travis](http://docs.travis-ci.com/user/encryption-keys/).

# The Future

It would be really nice to be able to:

* Run multiple minions on the same box as needed
* Use a pre-fab travis box, which has a precursor:
* Create a travis box for salt, which requires:
 * Adding salt to [travis-build](https://github.com/travis-ci/travis-build)
 * Adding a cookbook for salt to [travis-cookbooks](https://github.com/travis-ci/travis-cookbooks)
 * Adding [documentation on how to use it](https://github.com/travis-ci/travis-ci.github.com/tree/master/docs/user/languages)
* Create a tool to generate the config you need, automatically (or add it on to the travis rubygem)


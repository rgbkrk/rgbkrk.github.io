---
layout: post
title:  "Travis CI for Salt States: Part I"
date:   2014-01-29
description: Get Travis CI to check the validity of your Salt States
categories: [automation, travis, travis ci, configuration management, continuous integration, salt, chef, saltstack, Salt Stack]
---


[The IPython Notebook Viewer](http://nbviewer.ipython.org) now uses salt for its infrastructure management, keeping the base portion of its deployment as [open source salt states](http://github.com/ipython/salt-states-nbviewer). Being able to test and verify these states with open source tools would be excellent as:

1. Anyone can submit a pull request
2. The states are pulled via Salt's gitfs backend
3. They're used in production (along with some other states + pillar data)

# Verifying salt states

Verifying salt states works simply if you're not using any amount of templating (jinja, etc.), grains, or pillar data. Just run the states through a YAML validation tool. For everything else (read as: everything), you need to mirror your production environment via vagrant, Jenkins, or some other CI tool.

What if we could do some light validation (or more) on travis CI?

# Validating Salt States on Travis CI

<img style="display:block; margin-left: auto; margin-right: auto;" src="http://i.imgur.com/FW9Tjgb.png" alt="Travis, the Mustachioed Builder" />

To test your states on Travis, we simply need to install salt, set up some light configuration, and use `salt-call --local --retcode-passthrough` to run salt modules.

This configuration requires two files:

* `.travis/minion` - Simple `/etc/salt/minion` file
* `.travis.yml` - Basic setup for a simple states layout

Of course, you'll also need to [enable the repository on travis](http://docs.travis-ci.com/user/getting-started/).

Let's take these apart piece by piece. You can also go [straight to the travis.yml](#_in_whole).

## .travis/minion

This is pulled straight from the [standalone minion tutorial](http://docs.saltstack.com/topics/tutorials/standalone_minion.html).

```yaml
file_client: local
file_roots:
  base:
    - /srv/salt/states
```

## .travis.yml piece by piece

### Declare the language

In this example, we'll use python for our travis box(semi-arbitrarily chosen because [nbviewer](http://nbviewer.ipython.org) and salt run on python). Later, I'd like to see if adding salt as a base image on travis could be [possible](https://github.com/travis-ci/travis-ci/issues/1549) (or feasible). Hit me up if you'd like to see this and use this yourself.

```yaml
language: python
python:
- '2.7'
```

### Perform updates and install salt master

```yaml
before_install:
  - sudo apt-get update
  - curl -L http://bootstrap.saltstack.org | sudo sh -s -- git develop
```

### "Install" the states

We'll copy `.travis/minion` over to `/etc/salt/minion`, copy the states over, restart, and spit out some debugging information.

```yaml
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

Show the highstate, making sure to pass the retcode through. It might be wise to show the lowstate instead of or in addition to, since it shows the final output after the data has been "compiled".

```yaml
script:
  - sudo salt-call state.show_highstate --local --retcode-passthrough
```

## .travis.yml in whole

```yaml
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
  - sudo salt-call state.show_highstate --local --retcode-passthrough
```

# See a build

After your build runs, you can drill down into each line that was run from the `.travis.yml` file:

<img style="display:block; margin-left: auto; margin-right: auto;" src="http://i.imgur.com/kd3zalP.png" alt="nbviewer's salt states on travis" />

To see this in action, check out [Build #1](https://travis-ci.org/ipython/salt-states-nbviewer/builds/17864495) for [nbviewer's salt states](http://github.com/ipython/salt-states-nbviewer).

# Where's the pillar data?

For this "deployment", we're cheating a bit in that there are default values for the pillar data. For something more reasonably complex, you'll want to create some pillar data to use, possibly [encrypting it for travis](http://docs.travis-ci.com/user/encryption-keys/).

# The Future

It would be really nice to be able to:

* Run multiple minions on the same box as needed
* Create a tool to generate the travis config you need, automatically (or add it on to the travis rubygem)
* Create a travis box for salt

Creating a travis box for salt [likely requires](https://github.com/travis-ci/travis-ci/issues/1549):
* Adding salt to [travis-build](https://github.com/travis-ci/travis-build)
* Adding a cookbook for salt to [travis-cookbooks](https://github.com/travis-ci/travis-cookbooks)
* Adding [documentation on how to use it](https://github.com/travis-ci/travis-ci.github.com/tree/master/docs/user/languages)


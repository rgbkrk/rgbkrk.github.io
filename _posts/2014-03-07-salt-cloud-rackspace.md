---
layout: post
title:  "Using Salt Cloud with Rackspace"
date:   2014-03-07
image: http://i.imgur.com/0gclhEZ.jpg
description: Use the OpenStack provider with salt-cloud to build boxes on Rackspace.
categories: [automation, cloud, salt-cloud, configuration management, continuous integration, salt, saltstack, Salt Stack]
---

<img style="display:block; margin-left: auto; margin-right: auto;" src="http://i.imgur.com/0gclhEZ.jpg" alt="salt cloud shaker" />

Now that `salt-cloud` is bundled with salt itself (and has been for 3+ months), it's even easier to use salt cloud to provision minions easily.

As a Rackspace user, make sure to use the OpenStack provider. The current Rackspace driver actually tries to connect to the old cloud instances (pre-2010). Newer accounts won't even have access to those, but that's a discussion for a separate thread.

Let's build stuff!

This article assumes you have a working salt master setup. The primary pieces you need are `/etc/salt/cloud.providers` and `/etc/salt/cloud.profiles`, both of which go on the salt master.

## Example `/etc/salt/cloud.providers`

This setup needs to be replicated near exactly, replacing all the `{keyword}` with your own.

```yaml
rackspace-cloud:
  # Setup the minion configuration, including the salt-master IP
  # This can be the service net if the master is in the same
  # Data Center.
  minion:
    master: {master_ip}

  # Configure Rackspace using the OpenStack plugin
  provider: openstack
  identity_url: 'https://identity.api.rackspacecloud.com/v2.0/tokens'
  compute_name: cloudServersOpenStack
  protocol: ipv4

  # IAD 
  compute_region: IAD

  # Rackspace authentication credentials
  user: {rackspace_user}
  tenant: {rackspace_tenant_id} # Account number
  apikey: {rackspace_api_key}
```


## Example `/etc/salt/cloud.profiles`

To use salt-cloud, you need to set up profiles for the type of box you want to use, pairing size, image and provider.

```yaml
ubuntu_13.10_pvhvm_perf1.8:
    provider: rackspace-cloud
    size: performance1-8
    image: Ubuntu 13.10 (Saucy Salamander) (PVHVM)
```

After getting all these set up, make sure to restart your salt master.

# Launching servers via salt-cloud

Launching a single server is pretty easy. Just specify the profile from above and a name.

```console
# salt-cloud -p ubuntu_13.10_pvhvm_perf1.8 bessie
[INFO    ] salt-cloud starting
[INFO    ] Creating Cloud VM bessie
[INFO    ] Rendering deploy script: ...
```

To launch a configured group of servers, create a map which can be located anywhere. For the following, I store the cloud map at `/etc/salt/cloud.map`:

```yaml
ubuntu_13.10_pvhvm_perf1.8:

  - qa01.nbviewer.ipython.org:
        minion:
          log_level: debug
  - qa02.nbviewer.ipython.org:
        minion:
          log_level: debug

  - prod01.nbviewer.ipython.org:
        minion:
          log_level: info
  - prod02.nbviewer.ipython.org:
        minion:
          log_level: info
```

Then to launch the configuration, simply use `salt-cloud -m mapfile -P`:

```console
# salt-cloud -m /tmp/cloud.map -P
[INFO    ] salt-cloud starting
[INFO    ] Applying map from '/tmp/cloud.map'.
...
```

If you have any questions, feel free to reach out to me. I'm more than happy to help! <!-- Ha ha not anymore I haven't touched in salt in what feels like forever. -->


---
layout: post
title:  "Get Salted"
date:   2013-09-24
description: After starting at Rackspace I picked up Chef to work with the automated deployment folks. Chef is Ruby at its core, which makes it very expressive and easy to work with. As a Pythonista though...
categories: [automation, command and control, infrastructure management, configuration management,salt,chef,saltstack, Salt Stack]
---

After starting at [Rackspace](http://www.rackspace.com) I picked up [Chef](http://www.opscode.com/chef/) to work with the automated deployment folks. Chef is Ruby at its core, which makes it very expressive and easy to work with. As a Pythonista though, [something](http://saltstack.com/) kept pulling at me.

<img style="display:block; margin-left: auto; margin-right: auto;" src="http://i.imgur.com/0NUdprd.png" alt="Salt that Python" />

[Salt Stack](http://saltstack.com/) is a collection of free and open source infrastructure management software, designed to make deploying services a breeze. It's also a distributed remote execution system so you can execute arbitrary commands across all your nodes (or an arbitrary selection of them).

[Tom Hatch](https://twitter.com/thatch45), the original author of Salt, came and gave us a talk a couple months ago but I didn't have a chance to try it until this weekend. So far, I'm impressed.

## Getting Started

The [walkthrough for Salt Stack](http://docs.saltstack.com/topics/tutorials/walkthrough.html) has you run through a bit of manual setup for installation. I imagine later I'll be able to [create infrastructure on demand](https://salt-cloud.readthedocs.org/en/latest/), provision it with the right resources and get services running all in one command, regardless of the provider. For now, I had to bootstrap the servers.

First I set up the master, which more or less is the commander across all your servers. The servers, in Salt Stack parlance, are called minions. With full on parallel [remote execution](http://docs.saltstack.com/topics/tutorials/modules.html) and terminology like minions and targets, it sounds like you're building **Enterprise Botnet 3.0**.

Create at least two boxes (virtualbox, vagrant, Rackspace, AWS, physical boxes, whatever). Create more and you can broadcast your commands across minions. In fact, don't even bother with the tutorial if you're not going to launch more than one server. You won't grasp the raw power of salt without at least two minions.

### Bootstrap the Master

The quickest way to get the master started is to use the Salt Stack [bootstrap](http://docs.saltstack.com/topics/tutorials/modules.html) [script](http://bootstrap.saltstack.org). SSH in and run this one liner:

```console
curl -L http://bootstrap.saltstack.org | sudo sh -s -- -M -N git develop
```

  `-M` installs salt-master

  `-N` requests not to install salt-minion

### Bootstrap the Minions

The minions use the same script, but without the options above

```console
curl -L http://bootstrap.saltstack.org | sudo sh -s -- git develop
```

For each of the minions, edit `/etc/salt/minion`.

Turn this

```yaml
##### Primary configuration settings #####
##########################################

# Per default the minion will automatically include all config files
# from minion.d/*.conf (minion.d is a directory in the same directory
# as the main minion config file).
#default_include: minion.d/*.conf

# Set the location of the salt master server, if the master server cannot be
# resolved, then the minion will fail to start.
#master: salt
```

Into this

```yaml
##### Primary configuration settings #####
##########################################

# Per default the minion will automatically include all config files
# from minion.d/*.conf (minion.d is a directory in the same directory
# as the main minion config file).
#default_include: minion.d/*.conf

# Set the location of the salt master server, if the master server cannot be
# resolved, then the minion will fail to start.
master: 67.207.156.173 # Put the IP for your salt master here
```

Clearly you really only need one line (`master: 67.207.156.173`). The text above is what gets installed by default using the bootstrap script, you can clean it up like any other Linux config file.

### Start the Minions!

On each of the minions, run `salt-minion` as a daemon.

```console
salt-minion -d
```

They should each make a callback to the master. Log back on to master and run `salt-key -L`. Now you'll see some unaccepted keys.

<pre class="highlight">
root@master:~# salt-key -L
<span style="color:green;font-weight:bold;">Accepted Keys:</span>
<span style="color:red;font-weight:bold;">Unaccepted Keys:</span>
<span style="color:red;">test-minion1</span>
<span style="color:red;">test-minion2</span>
<span style="color:blue;font-weight:bold;">Rejected Keys:</span>
</pre>

If you're paranoid (as you probably should be), you can print out the public key for each of the minions that are waiting to do your bidding.

<pre class="highlight">
root@master:~# salt-key --print-all
<span style="color:red;font-weight:bold;">Unaccepted Keys:</span>
<span style="color:red;">
test-minion1:  -----BEGIN PUBLIC KEY-----
MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEA2at6hnlYqWNSXszqYojo
h5hw6rdnxkLGi0qeEI+dPE0A1UidvNcEu1i1tJHTGhkMqXTExkA765/ZLfgmLzUn
ekyytuhTjbZbfuiZinQLkZAkeaY5gqcArYpIPNAKSihRaxsKJYZSxtyZnAoEa3Xq
sycxLu95UoOO7D/JKEbHv2OsEwi21sFtGhem0irPIZRc1xCYdgVt7eKE90gyTNYk
m66zohDSn54GVeEoqgQt4GyQ+0Qg5GKQ6W3NhmDB96TFwGXlU72RWSQY+i0A1F4/
wQGjZacXh24YOJre54mC2JnQ2ZX4I8oGBIPBgUz18cabeYX1YOz+0pVnI9JwujVe
+XGMohvaapVMJIietho1nP0QwzNx8vdJJcn56COFQUyzbCImxSj+7g0hToqGWB4k
izCkxD96zH0tU4tRXDtxxOL2o0JLkmA+yLt8cjUiXvmXGAZ64flIorD1B6jFKSTQ
vC9wqlhtfG2/DP0GXlvrPTV2utx6+ecfeh42d4UnOHnRBrHoKJMlEmiski+EInhX
wRRgUx8LdWJEjIgi45eBjpXyyvjiYhu2UbQOMYh6OlDaQ+2ztJfxljIl23VoBt0W
/3gw5QhVc6W3hUWUuNYLyood8122dASymkQ44NJF9AwKY3Dnp7GwereoxHkZfY0X
RYbuBlMgEeXP5Z0h/uivHPkCAwEAAQ==
-----END PUBLIC KEY-----

test-minion2:  -----BEGIN PUBLIC KEY-----
MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEA3xQP2soMLUSlstLTB45H
spVlBlWvW8bgOuQk6EwB9wRr7/kJCxyXP1OZ8ddszd0FBjL2iX2+7xBVzhvnWk4q
ptskbeh8k9RqpRQ4HFX6lsYf3SSuCl+FrfXYikF4N2X8gTAvntWOQT6vtVgTdaoT
QfWdL5t5NcEnH2cxxqpR/f3+JMnDDFYar/TNcGVkIar6uEiPlOOMz82APYLVqOzj
Wo0fiiJpuBYvcJ9+hrp1q/DtWRXvsHlWPfZszIJ3NhTHdybrlqIRsJ5Lieot/7Pk
QxT6Hky0CNsDWVh5DcUt9f+kcFyhCxReHOSWsb4LjvVurBDlYk77GmVfc5e6KBCw
vr82YJqsSJBnOzGvcrrpu6VAEwnpk+YwD5AvhXp6mXrjc0ULAzbRz6xGlD2coYTu
9dXiBQRzqFfIUOWF/0EK9Aj8fqPM5rxxTTzA1Ck8ecANkfXSOtIIMpfNb1MOZTgS
9tquzaIYSFqQ1SOcZTw1La2zyOR0YwwGDW33fUAt/wE3z+WHKpTIwFa4gCx4vpzX
LCEAcK8o6w9ORz9Bf/XsJ+zqLWqc5GqOfd0lcpEc50jCQTW1Hrk5BvmOiO26XUFt
vRmmW63LFUBWC8ObxNgW3OHMwgbHgEbQrkK8aAqFZkI/PnqygyyOwC42Kt6+QH76
oOKfWkwPE7VVPXkjQfnsVfECAwEAAQ==
-----END PUBLIC KEY-----
</span>
</pre>

You can verify that these match the public keys stored at `/etc/salt/pki/minion/minion.pub` on each of the minions. Then you can simply run `salt-key -A` to accept all the keys (with confirmation). It worries me a little that I can't check the public key while going through the confirmation.

## Alright Minions, Let's Play

<img style="display:block; margin-left: auto; margin-right: auto;" src="http://i.imgur.com/LtfoAaK.png" alt="ABCD RPC" />

Now that we've got everything configured, let's do some remote commands on our minions.

If you want to perform traceroutes across lots of servers/minions no matter their location, it's easy:

```console
root@test-master:/srv/salt# salt '*' pkg.install traceroute
root@test-master:/srv/salt# salt '*' network.traceroute www.github.com
```

Salt will even hand you back JSON if you prefer:

```console
root@test-master:/srv/salt# salt '*' network.traceroute www.github.com --out=json
```

How about installing pip on all the nodes:

<pre class="highlight">
root@master:~# salt '*' pkg.install python-pip
<span style="color:teal;">test-minion1</span>:
    <span style="color:teal;">----------</span>
    <span style="color:teal;">python-pip</span>:
        <span style="color:teal;">----------</span>
        <span style="color:teal;">new</span>:
            <span style="color:green;">1.0-1build1</span>
        <span style="color:teal;">old</span>:
            <span style="color:green;"></span>
    <span style="color:teal;">python-setuptools</span>:
        <span style="color:teal;">----------</span>
        <span style="color:teal;">new</span>:
            <span style="color:green;">0.6.24-1ubuntu1</span>
        <span style="color:teal;">old</span>:
            <span style="color:green;"></span>
<span style="color:teal;">test-minion2</span>:
    <span style="color:teal;">----------</span>
    <span style="color:teal;">python-pip</span>:
        <span style="color:teal;">----------</span>
        <span style="color:teal;">new</span>:
            <span style="color:green;">1.0-1build1</span>
        <span style="color:teal;">old</span>:
            <span style="color:green;"></span>
    <span style="color:teal;">python-setuptools</span>:
        <span style="color:teal;">----------</span>
        <span style="color:teal;">new</span>:
            <span style="color:green;">0.6.24-1ubuntu1</span>
        <span style="color:teal;">old</span>:
            <span style="color:green;"></span>
</pre>

Well that was easy. It even reported back all the dependencies it needed.

Ok, let's install a package using pip (yes, we should be using a virtualenv):

<pre class="highlight">
root@master:~# salt '*' pip.install ipython[notebook]
<span style="color:teal;">test-minion1</span>:
    <span style="color:teal;">----------</span>
    <span style="color:teal;">pid</span>:
        <span style="color:olive;font-weight:bold;">14678</span>
    <span style="color:teal;">retcode</span>:
        <span style="color:olive;font-weight:bold;">0</span>
    <span style="color:teal;">stderr</span>:
        <span style="color:green;"></span>
    <span style="color:teal;">stdout</span>:
        <span style="color:green;">Downloading/unpacking ipython[notebook]</span>
        <span style="color:green;">  Running setup.py egg_info for package ipython</span>
        <span style="color:green;">    </span>
        <span style="color:green;">Installing collected packages: ipython</span>
        <span style="color:green;">  Running setup.py install for ipython</span>
        <span style="color:green;">    </span>
        <span style="color:green;">    Installing ipcontroller script to /usr/local/bin</span>
        <span style="color:green;">    Installing iptest script to /usr/local/bin</span>
        <span style="color:green;">    Installing ipcluster script to /usr/local/bin</span>
        <span style="color:green;">    Installing ipython script to /usr/local/bin</span>
        <span style="color:green;">    Installing pycolor script to /usr/local/bin</span>
        <span style="color:green;">    Installing iplogger script to /usr/local/bin</span>
        <span style="color:green;">    Installing irunner script to /usr/local/bin</span>
        <span style="color:green;">    Installing ipengine script to /usr/local/bin</span>
        <span style="color:green;">Successfully installed ipython</span>
        <span style="color:green;">Cleaning up...</span>
<span style="color:teal;">test-minion2</span>:
    <span style="color:teal;">----------</span>
    <span style="color:teal;">pid</span>:
        <span style="color:olive;font-weight:bold;">15025</span>
    <span style="color:teal;">retcode</span>:
        <span style="color:olive;font-weight:bold;">0</span>
    <span style="color:teal;">stderr</span>:
        <span style="color:green;"></span>
    <span style="color:teal;">stdout</span>:
        <span style="color:green;">Downloading/unpacking ipython[notebook]</span>
        <span style="color:green;">  Running setup.py egg_info for package ipython</span>
        <span style="color:green;">    </span>
        <span style="color:green;">Installing collected packages: ipython</span>
        <span style="color:green;">  Running setup.py install for ipython</span>
        <span style="color:green;">    </span>
        <span style="color:green;">    Installing ipcontroller script to /usr/local/bin</span>
        <span style="color:green;">    Installing iptest script to /usr/local/bin</span>
        <span style="color:green;">    Installing ipcluster script to /usr/local/bin</span>
        <span style="color:green;">    Installing ipython script to /usr/local/bin</span>
        <span style="color:green;">    Installing pycolor script to /usr/local/bin</span>
        <span style="color:green;">    Installing iplogger script to /usr/local/bin</span>
        <span style="color:green;">    Installing irunner script to /usr/local/bin</span>
        <span style="color:green;">    Installing ipengine script to /usr/local/bin</span>
        <span style="color:green;">Successfully installed ipython</span>
        <span style="color:green;">Cleaning up...</span>
</pre>

Sweet. Of course, no one wants to configure these systems and deploy code by hand. The next stop is salt states. By creating some simple yaml files (that use Jinja templates), we can configure quite a bit about a machine.

The walkthrough has you install nginx but has you do something a bit strange in that it makes you state that the nginx package is installed to run the service. It's stated twice so it seems redundant. Here's `/srv/salt/nginx/init.sls`:

```yaml
nginx:
  pkg:
    - installed
  service:
    - running
    - require:
      - pkg: nginx # I feel like I am repeating myself
```

## Looking Forward

The features I'm most interested in are automated deployments with git. In particular, I want to be able to deploy based off of git hooks to production, staging, and development for the [IPython Notebook Viewer](http://nbviewer.ipython.org).

It's simple enough to clone the repo across specific minions

```console
salt 'nbviewer-dev' git.clone /var/www/nbviewer git://github.com/ipython/nbviewer
```

It's also easy to install specific commits. I'm really looking forward to grokking the [gitfs backend](https://salt.readthedocs.org/en/latest/topics/tutorials/gitfs.html) next, but I'm not sure what the right approach is.


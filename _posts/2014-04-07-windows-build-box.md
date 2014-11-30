---
layout: post
title:  "Bootstrapping a Rackspace Windows Box"
date:   2014-03-07
#image: http://i.imgur.com/6hZdRzP.png?1
description: Create a build box that can be managed externally from a Linux box
categories: [automation, cloud, devops]
---

<img style="display:block; margin-left: auto; margin-right: auto;" src="http://i.imgur.com/6hZdRzP.png?1" alt="Surprised that I try anything on Windows Servers." />

There are several projects I want to help out with Windows build automation (at least until Travis CI supports Windows builds) on cloud servers.

Since I'm not familiar *at all* with managing a Windows Server, let alone Windows security, there are several things I'd like to do:

1. <s>Stop using Windows and go back to Linux</s>
2. Launch a fresh server only when we need it
3. Cordon off the server to a private network

Since **1** isn't really an option, we'll just have to lock things down to our best ability and figure out how to manage a Windows box remotely without having to resort to using Remote Desktop. It turns out the de facto way to do this is to use <a href="http://msdn.microsoft.com/en-us/library/aa384426(v=vs.85).aspx">WinRM (Windows Remote Management)</a>.

The Windows Rackspace images (at the time of this writing) don't have the firewall for WinRM open, so you'll need to do that yourself. For even further sanity, we can put this Windows box on a private network.

Our steps:

1. Create a private network
2. Build a "publicly" accessible Linux box
3. Write a bootstrap script that sets up the Windows firewall
4. Boot a Windows box on the private network with the bootstrap script

# Open some Windows

First, let's build a network to be shared between the controlling Linux machine and our Windows box(es).

```console
$ nova network-create "BuildNet" "192.168.4.0/24"
+----------+--------------------------------------+
| Property | Value                                |
+----------+--------------------------------------+
| cidr     | 192.168.4.0/24                       |
| id       | e22b534d-4835-43dd-9896-f5b3f6123569 |
| label    | BuildNet                             |
+----------+--------------------------------------+
```

Then we'll build a Linux box that uses the above network and an SSH key.

```console
$ nova boot controller.dfw \
    --image 6110edfe-8589-4bb1-aa27-385f12242627 \
    --flavor performance2-15 \
    --nic net-id=e22b534d-4835-43dd-9896-f5b3f6123569 \
    --key-name rgbkrk --poll
```

Miss writing batch files for Windows? Well now is your chance to relive it.
Next up is writing the bootstrap script for the Windows box. This starts up the
Windows time service, syncs it, and enables winrm (over http...). This was
blatantly copied and modified from the [Step-by-step Walkthrough to Using Chef to Bootstrap Windows Nodes on the Rackspace Cloud](http://developer.rackspace.com/blog/step-by-step-walkthrough-to-using-chef-to-bootstrap-windows-nodes-on-the-rackspace-cloud.html).

```winbatch
REM Make sure to change 192.168.4.1 to the IP you will be connecting from
REM Comments will need to be removed to render this on the box (size requirement)

REM Start the Windows time service and sync it with time.dfw1.rackspace.com
net start w32time
w32tm /config /manualpeerlist:"time.dfw2.rackspace.com time.dfw1.rackspace.com" /syncfromflags:manual /reliable:yes /update
w32tm /resync

REM Allow connections from 192.168.4.1 on the winrm port
netsh advfirewall firewall set rule group="remote administration" new enable=yes & netsh advfirewall firewall add rule name="WinRM Port" dir=in action=allow protocol=TCP remoteip=192.168.4.1 localport=5985

REM Add controller to the hosts file (set controller to your hostname)
echo 192.168.4.1 controller.dfw >> C:\Windows\system32\drivers\etc\hosts
```

Save that to `open_hatch.cmd`, locally.

Now we can boot the Windows box, using the private network (as well as disabling both service-net and the public internet) and uploading the bootstrap script.

```console
$ nova boot
    --image 240b8fb2-da7a-482a-9429-891b374cb57c \
    --flavor performance2-15 \
    --nic net-id=e22b534d-4835-43dd-9896-f5b3f6123569 \
    --file "C:\\cloud-automation\\bootstrap.cmd=open_hatch.cmd" \
    --no-service-net --no-public \
    --poll windex
```

In order to get your script to run, it has to be placed at `C:\cloud-automation\bootstrap.cmd`. If you want to use a PowerShell script you're not completely out of luck, as you can upload more than one file (more `--file` arguments, up to 5). Then just execute the PowerShell script from the batch file.

To test this out, try out [pywinrm](https://github.com/diyan/pywinrm) and cloudbase's neat little [`wsmancmd.py`](https://github.com/cloudbase/winrm-scripts/blob/master/wsmancmd.py) script:

```console
$ python wsmancmd.py
    -U http://192.168.4.2:5985/wsman \
    -u Administrator \
    -p <plain_password> \
    'powershell -Command Get-NetIPConfiguration'
```

You can also install [winexe](http://sourceforge.net/projects/winexe/) if you feel like shaving that yak.

# What's next?

Now you can use the [Linux box as a bridge](http://blog.fict.io/automation/cloud/devops/2014/03/07/windows-build-box/), to run commands and use tools in automation. From here, setting up the Windows box as a salt minion isn't that bad either. Maybe I'll figure out a great way to roll some of this into salt-cloud, if just as an extension.

If I had more time for this, it would be really great to get [WinRM HTTPS certificate authentication](http://www.cloudbase.it/windows-without-passwords-in-openstack/) working out of the box as well, possibly by transmutating your SSH keys from nova.

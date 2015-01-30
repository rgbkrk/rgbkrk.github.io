---
layout: post
title:  "Instant Temporary IPython Notebooks"
date:   2014-11-02 20:01
categories: IPython, Jupyter
permalink: "/ipythonjupyter-tmpnb-debuts/"
---

It's about time I write about [tmpnb](https://github.com/jupyter/tmpnb), a project the Jupyter/IPython developers have been working on. [Brian Granger](https://twitter.com/ellisonbg), one of the project leads, has been asking me over the past few months about enabling some more ambitious use cases of our Docker containers within [IPython](https://registry.hub.docker.com/repos/ipython/) and [Jupyter](https://registry.hub.docker.com/repos/jupyter/):

* Provisioned notebooks for usability studies
* Instant notebooks for demos
* JupyterHub for our team

[Helen Shen](https://twitter.com/HelenShenWrites), science writer, and [Richard Van Noorden](https://twitter.com/Richvn), [Nature](http://www.nature.com/) reporter, have been working on [an article on the IPython Notebook](http://www.nature.com/news/interactive-notebooks-sharing-the-code-1.16261). They asked us if we could create a way for readers to try the notebook. After reflecting on the current JupyterHub code, I realized that with our Docker images and our [configurable http proxy](https://github.com/jupyter/configurable-http-proxy) (node-http-proxy with an admin API), we could easily setup paths and route users. Like any good (deranged) programmer, I first wrote this in bash:

```bash
#!/usr/bin/env bash
myrand=`head -c 30 /dev/urandom | xxd -p`
cont=$( docker run -it -d -P -e RAND_BASE=$myrand jupyter/demo )
port=$( docker port $cont 8888 | cut -d":" -f2 )
 
# proxy on         *:8000
# admin on 127.0.0.1:8001
curl -H "Authorization: token $CONFIGPROXY_AUTH_TOKEN" \
     -XPOST -d '{"target":"http://localhost:'$port'"}' \
     http://localhost:8001/api/routes/$myrand
curl -H "Authorization: token $CONFIGPROXY_AUTH_TOKEN" \
     http://localhost:8001/api/routes | python -m json.tool

# Spit out the URL for the new notebook server
echo `curl -s icanhazip.com`:8000/$myrand/
```

Yay! As soon as I had this working it was time to start on a Tornado app to do the same dynamically. I immediately told [Min RK](https://twitter.com/minrk) about it and he seemed to like it.

*"Nifty! Temporary notebook servers. That could be a thing."*

![](https://speakerd.s3.amazonaws.com/presentations/f5dca2a02e41013233bb1ac45923d988/slide_10.jpg?1412783050)

<h1 style="text-align: center;">↓</h1>

![](https://speakerd.s3.amazonaws.com/presentations/f5dca2a02e41013233bb1ac45923d988/slide_11.jpg?1412783050)

Roughly speaking, the tornado application does the above and just a bit more. When the user lands at a hosted tmpnb server:

* A Docker container gets started (the [jupyter/demo image](https://registry.hub.docker.com/u/jupyter/demo/) by default)
* A random path is created for them, e.g. `/user-asdf/`
* The proxy is told to route `/user-asdf/` to the local Docker container running on some port
* The user is redirected to their fresh server.

*Whoa whoa whoa, are you saying this will run any Docker container with IPython notebook inside?*

Yes, with some caveats. tmpnb needs to be able to tell the underlying containers what path they're being served from so that instead of trying to load `/static/notebook.js`, it loads `/user-A/static/notebook.js`

We're working to make this compatible with any IPython notebook Docker container while allowing some configurability. [Ash Wilson](https://twitter.com/smashwilson) wants to go further, making this work with [any Docker container](https://github.com/jupyter/tmpnb/pull/69#discussion_r19055529). It's possible so long as all paths are relative (`../static/util.js`) or adjusted for their assigned base path (`/user-a/static/util.js`).

## Tutorials! Books! Learning!

![](https://d23f6h5jpj26xu.cloudfront.net/rujc2e9z7jada_small.gif)

[Andrew Odewahn](https://github.com/odewahn), the CTO of O'Reilly Media, and Paco Nathan, mad scientist, have been [driving efforts](https://github.com/odewahn/docker-at-oreilly) to make Dockerfiles & Docker images [paired with books](https://github.com/ceteri/jem-docker). They're doing fantastic work so that you can type:

```
docker run -it -p 8888:8888 ceteri/jem
```

to pull up an IPython notebook with content for a book and play along.

Just prior to the [public demo of tmpnb at PyTexas](https://speakerdeck.com/rgbkrk/tmpnb-at-pytexas), I called up Andrew to tell him about the tmpnb efforts. He was super excited, just as I was, to get something like this working for authored content.

Andrew: *"We're going to be running a tutorial at Strata using a Docker image we've been working on, do you think we could use this for that?"*

Kyle: *"Well, there's some engineering work to be done but we can try to make it work."*

Fast forward many PRs, some last minute coordinating of a trip to Strata, and we got our prototype pushed forward for the tutorial. Meanwhile, the IPython team at large is very excited about tmpnb. Notables: Min [added the culling](https://github.com/jupyter/tmpnb/pull/10) and wrote [a trie in Javascript](https://github.com/jupyter/configurable-http-proxy/pull/8) (key-value lookups are not O(1) in JavaScript??!?!?).

Once we got to a stable state and Andrew put up the Just Enough Math server to Strata, we did some load testing which was a combination of slimerjs/phantomjs and [Min's nbbot](https://github.com/minrk/nbbot) (that he wrote just for this). It was clear we were going to need spawn pools in the long run (containers ready to be routed) and that we had some socket and file descriptor issues.

![](https://d23f6h5jpj26xu.cloudfront.net/qvrhxbeay9zxpq_small.jpg)

The morning of PyData at Strata, Brian Granger comes over to me before his and Fernando's opening talks, "Is tmpnb ready?" I can only respond "Yes? Maybe. Sure, why not." As Brian walks back up to the stage, worry comes over me as I had not applied the updates to the currently deployed system. "Really hope this works." What preceded was 30 minutes of relative bliss as 5... 10... 20... 40... 80 users logged on to tmpnb.org. Everyone was able to follow along with the notebooks themselves, clicking and opening the notebooks that Fernando was running through. We got our first tweet as this was going on

<blockquote class="twitter-tweet" lang="en"><p>Attending <a href="https://twitter.com/hashtag/PyData?src=hash">#PyData</a> at <a href="https://twitter.com/hashtag/Strataconf?src=hash">#Strataconf</a> -- already impressed at the hosted <a href="http://t.co/2zUVPbfFAj">http://t.co/2zUVPbfFAj</a> notebooks.</p>&mdash; A.J. Rader (@indyrader) <a href="https://twitter.com/indyrader/status/522373218066505728">October 15, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Woohoo! Eventually though, the dreaded hand raised:

*"Hey, tmpnb.org isn't working anymore. Can you share a link to the notebooks?"*

The proxy could no longer get file descriptors.

```
13:28:11.164 - Proxy error:  code=EMFILE, errno=EMFILE, syscall=connect
13:28:11.408 - Proxy error:  code=EMFILE, errno=EMFILE, syscall=connect
13:28:11.666 - Proxy error:  code=EMFILE, errno=EMFILE, syscall=connect
```

Ouch.

While checking on what was going on with the file descriptors, we noticed that `lsof` + Docker didn't play well since the filesystems aren't the same.

```
root@demo:~# lsof 2> /dev/null | grep python | cut -c 88- | sort | uniq -c | sort -nr | head
  25861 anon_inode
  13570 can't identify protocol
   7814 pipe
   3067 /dev/urandom
   2682 /home/jupyter/.ipython/profile_default/history.sqlite
   1833 /dev/null
   1637 /
   1587 /usr/lib/x86_64-linux-gnu/libzmq.so.3.1.0 (stat: No such file or directory)
   1587 /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.19 (path dev=8,1, inode=7783)
   1587 /usr/lib/x86_64-linux-gnu/libsqlite3.so.0.8.6 (path dev=8,1, inode=7737)
```

Even with the troubles above, Olivier Grisel wanted to run one for his tutorial too. We quickly wrapped one up based on the `jupyter/demo` image within a few minutes and off he went.

```
FROM jupyter/demo

USER root
RUN rm -rf ipython_examples *.ipynb
ADD . /home/jupyter
RUN chown jupyter:jupyter . -R
RUN pip3 install psutil

USER jupyter
```

Apparently his ran without any hitches. He is one of my favorite people in the world. Note: he did not have nginx in front - it was straight to the configurable proxy AND the proxy was running in Docker instead of supervisor.

One optimization I took out of this (not clear in the above `anon_inode` references) was to pull the static content out of the container and serve via nginx. That really helped performance in addition to saving file descriptors.

### Just Enough Math

When it came to the Just Enough Math tutorial, I was able to get static content served on the nginx box quickly by pulling the right static content out of the docker images directly using `docker cp`. With that fix just in time for the tutorial, it was another chance to see tmpnb live (albeit on the strataconf.com domain).

<blockquote class="twitter-tweet" lang="en"><p>Watching <a href="https://twitter.com/pacoid">@pacoid</a> and <a href="https://twitter.com/allenday">@allenday</a> rocking the IPython notebook and the audience using tmpnb at <a href="https://twitter.com/hashtag/Strataconf?src=hash">#Strataconf</a> <a href="http://t.co/HwSyGn3RP2">pic.twitter.com/HwSyGn3RP2</a></p>&mdash; Kyle R Kelley (@rgbkrk) <a href="https://twitter.com/rgbkrk/status/522449621378154496">October 15, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Great! It started off well and for most folks they got to just experience the content. No thought to the platform it was running on, they just got to focus on the IPython notebook and the tutorial content. Wonderful.

45 minutes in this time, with a bigger crowd (around 150) we hit our limits again. This time we ran out of actual socket descriptors for the proxy. Doing some digging live during the tutorial, we found out where the issues laid.

For each notebook of each user, 3 extra sockets were being opened - one for each of IPython's websockets for `shell`, `stdin`, and `iopub`. *That's on every notebook file.* This meant that the first notebook was "ok" but then new notebooks (Not servers, the actual files, e.g. Untitled0.ipynb, Untitled1.ipynb) would open 3 more socket descriptors. Because this was proxied from nginx -> node -> docker, you can double that number too.

With only two notebooks per user and 110 users, that's on the order of 1430 sockets. Node was running as nobody and could only open 1024 ports. Despite my (primitive) attempts to change ulimits on the fly, I couldn't get it stable right then and there.

### NYC Python Meetup

[James Powell](https://twitter.com/dontusethiscode), having seen [my talk at PyTexas on tmpnb](https://speakerdeck.com/rgbkrk/tmpnb-at-pytexas), asked if I could speak at the NYC Python meetup. Since I had the lightning talk ready and had a new system up with [Ash Wilson](https://twitter.com/smashwilson)'s spawn pool PR, I decided to give it a go.

<blockquote class="twitter-tweet" data-cards="hidden" lang="en"><p>Showed off <a href="https://twitter.com/smashwilson">@smashwilson</a>&#39;s tmpnb PR <a href="https://t.co/RWSj3Rd9ps">https://t.co/RWSj3Rd9ps</a> at <a href="https://twitter.com/nycpython">@nycpython</a> tonight. Flawless 0s nb demo! Thanks <a href="https://twitter.com/dontusethiscode">@dontusethiscode</a> and pals!</p>&mdash; Kyle R Kelley (@rgbkrk) <a href="https://twitter.com/rgbkrk/status/522965532770074624">October 17, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Needless to say, that went flawlessly (besides my own laptop display issues) and the audience was wonderful.

## Put some Teeth on that Cloud

![](https://d23f6h5jpj26xu.cloudfront.net/j98s5ozwffseya_small.png)

Currently I've been deploying this on fairly hefty machines, subdividing the machines up to give individual users (roughly) the same amount of storage they would come to expect on a VPS (virtual private server). One of Rackspace's [OnMetal](http://www.rackspace.com/cloud/servers/onmetal/) Memory boxes has half a terabyte of RAM, so I figured I could subdivide that up into 512 MB chunks. With 1000 users, that gives us roughly 12 GB for the main OS.

The price of the OnMetal Memory machines is $1600, so the per user per month price is *$1.60*. Packing bare metal with containers looks very economical to me but I'm also subdividing 24 cores across 1000 users. What does a typical cheap VPS do for slicing cores (other than telling you that you're sharing a CPU)? I'd like to explore all this quite a bit more. If I'm off, please let me know.

## Thanks Rackspace, O'Reilly, Nature, Others

The temporary notebook service has been graciously hosted by [Rackspace](https://developer.rackspace.com) on their OnMetal servers. Andrew Odewahn & O'Reilly ran the Just Enough Math server at Strata. I am forever grateful to both companies for driving forward with such new technologies and letting me explore with them.

Thank you to Richard Van Noorden, Brian Granger, and Min RK for spawning the initial impetus and ideas behind tmpnb. Oh and Docker! :D

--------------------

Comments on [Reddit via `/r/python`](http://www.reddit.com/r/Python/comments/2l3gk4/creating_an_instant_temporary_ipython_notebook/) or [on Twitter](https://twitter.com/rgbkrk/status/529005294064775168).

### Read this next

[Ops Lessons on Instant Temporary IPython Notebooks](/ops-lessons-and-instant-temporary-ipython-jupyter-notebooks/)

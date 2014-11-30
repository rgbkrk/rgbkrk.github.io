---
layout: post
title:  "The Encrypted Message Service I'm Not Building"
date:   2014-05-16 20:01
categories: Encryption, messaging
permalink: "/the-encrypted-message-service-im-not-building/"
---

Every time I need to share a credential (password, API Key, etc.) for a shared account I'm faced with a dilemma. How do I pass someone a secret, encrypted just for them? Public keys are an obvious choice. What if they're not using GPG though? What else does a typical developer have?

**SSH Keys.**

Armed with the knowledge that
* Just about everyone has SSH keys
* GitHub provides public SSH keys at https://github.com/{user}.keys
* `ssh-keygen` and `openssl` are typically available to handle crypt

It's pretty simple to do this.

```bash
# Get the user's keys, use the last key
wget https://github.com/$user.keys --quiet -qO- | tail -n 1 > $pubkey
 
# Convert to a pem file
ssh-keygen -f $pubkey -e -m PKCS8 > $pubkey.pem

# Encrypt some message
openssl rsautl -encrypt -pubin -inkey $pubkey.pem -ssl -in $infile -out $outfile
```

This relies on the fact that you can encrypt something small using a public key. Usually this "something small" is a key for a symmetric encryption algorithm (e.g. AES), which in turn is used to encrypt a much larger message body. Instead, we're encrypting our whole message (which is given some random padding by `openssl`).

There are some issues with doing this:

* It relies on a third party for key integrity, no fingerprints
* You can only encrypt up to a certain number of bytes

For a typical key size, it still gives you plenty more characters than Twitter does though.

Key size (bits) | Maximum Message Size (bytes)
--------------- | ----------------------------
768             | 85
1024            | 117
2048            | 246
4096            | 502
8192            | 1018

## Enter CryptHub ![](https://d23f6h5jpj26xu.cloudfront.net/knsubajidfhzqg_small.png)

Even with these concerns and a looming fear that there would be something else I'm missing, I wrote a gist and shared it with the world.

<blockquote class="twitter-tweet" lang="en"><p>Hack of the day: Encrypting a file using a GitHub user&#39;s public key - <a href="https://t.co/PJWYIWAfs2">https://t.co/PJWYIWAfs2</a></p>&mdash; Kyle Ray Kelley (@rgbkrk) <a href="https://twitter.com/rgbkrk/statuses/409156138634973184">December 7, 2013</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

After a [good response on Twitter and the gist](https://gist.github.com/rgbkrk/7827691#comment-965445) I realized that, well, *maybe this could be a thing*. What if we build an ephemeral message service that relies on people's public keys from GitHub? (Or BitBucket, GitLab, etc.) Anyone with a GitHub account can post a message to another GitHub user, relying on the public keys the recipient has stored on GitHub. Authentication can be handled using GitHub's OAuth.

At the time I was thinking of this, Rackspace had just come out with [CloudQueues](http://www.rackspace.com/cloud/queues/) which is built from [OpenStack Marconi](https://wiki.openstack.org/wiki/Marconi). You can think of it like [AWS's SQS](https://aws.amazon.com/sqs/) or using RabbitMQ yourself. Each of these expire messages after a certain amount of time and consume short messages (for the astute reader, yes, I'm hand waving here).

Throw the encrypted messages on a queue per user, let the recipients download them later. Messages get deleted when the time is up or the recipient trashes them. The service can't see your keys or your plaintext messages.

### Informal Design

To get setup, a user

* goes to CryptHub.io, logs in using GitHub
* downloads a simple client application
* sets up GitHub OAuth keys with the client app

#### Sending a message

```
$ hubencrypt smashwilson -m "Drink more ovaltine"
Getting the key for smashwilson. ☁
Converting public key to a PEM PKCS8 public key. √
Encrypting message. √
Message sent!
```

The message gets placed on a message queue for a recipient to download.

#### Recipient

```
$ hubdecrypt -r
Downloading messages. ☁
Message from rgbkrk:
>> Drink more ovaltine
```

# Reservations and the Future

What are the security implications? Who would use it? Will this look poorly on me if this is Fundamentally a Bad Idea™?

This doesn't even solve the problem for people not on GitHub (unless they share public keys through other channels). If they're not even a techie, this is mostly useless for them. We'd have to build a native application for Windows that can generate the keys, etc. OK, maybe that isn't so bad.

Since this was just a side project I was thinking about with [Ash Wilson](https://github.com/smashwilson) while my wife and are currently chasing a 2 year old and an infant, I left this on the backburner. In the meantime, [Jan Schaumann](https://gist.github.com/jschauma) updated [jass](https://github.com/jschauma/jass) to use keys from Github, completing the client side portion.

Then [keybase](https://keybase.io/) came along and I thought *well there's one way to do it*. Not quite the way I'd want it, but pretty close. I won't lie that it made me kick myself in the butt for not just pushing out what we had so far in open source.

That's ok, there are more problems to be solved.

Still wondering though...

**Would you use it?**

* [Discussion on Reddit](http://www.reddit.com/r/programming/comments/25r5lf/the_encrypted_message_service_im_not_building/)

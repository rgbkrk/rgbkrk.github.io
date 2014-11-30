---
layout: post
title:  "rm -rf remains"
date:   2014-06-06 20:01
categories: Linux tomfoolery
tags: featured
permalink: "/rm-rf-remains/"
---

Just for fun, I decided to launch a new Linux server and run `rm -rf /` as root to see what remains. As I found out, `rm` lives in the future with idiots like me, so you have to specify `--no-preserve-root` to kick this exercise off.

    # rm -rf --no-preserve-root /

After committing this act of tomfoolery, great utilities like

* `/bin/ls`
* `/bin/cat`
* `/bin/chmod`
* `/usr/bin/file`

will all be gone! You should still have your connection over SSH as well as your existing bash session. This means you have all the bash builtins, like `echo`. 

## Becoming Bash McGyver

```console
root@rmrf:/# ls
-bash: /bin/ls: No such file or directory
```

There is no `ls`, but `echo` and fileglobs are still around. What can we do with those?

```console
root@rmrf:/# echo *
dev proc run sys
# echo /dev/pts/*
/dev/pts/0 /dev/pts/3 /dev/pts/ptmx
```

Hey, we got to keep `/dev`, `/proc`, `/run`, and `/sys`.  Now that we have `ls`, we  might as well make it a little easier to read.

```console
root@rmrf:/# for file in /dev/pts/*; do echo $file; done
/dev/pts/0
/dev/pts/3
/dev/pts/ptmx
```

[Several](http://www.reddit.com/r/programming/comments/27j311/rm_rf_remains/ci1gc4i) [Redditors](http://www.reddit.com/r/programming/comments/27j311/rm_rf_remains/ci1gpli) pointed out that `printf` is available.

```console
root@rmrf:/# ls() { printf '%s\n' ${1:+${1%/}/}*; }
```

"*printf will [apply] the format string until it runs out of args.*" - [camh-](http://www.reddit.com/user/camh-)

Since you can define functions in bash, we can create an `ls` utility, albeit very limited.

```console
root@rmrf:/# ls() { printf '%s\n' ${1:+${1%/}/}*; }
-bash: syntax error near unexpected token `('
```

What? That should be completely valid. Is `ls` already hashed or aliased?

```console
root@rmrf:/# type ls
ls is aliased to `ls --color=auto'
```

Ah, that gets expanded to `ls--color=auto () { printf '%s\n' ${1:+${1%/}/}*; }`. Yuck. Well, we can `unalias` that.

```console
root@rmrf:/# unalias ls
root@rmrf:/# ls() { printf '%s\n' ${1:+${1%/}/}*; }
root@rmrf:/# ls
/dev
/proc
/run
/sys
root@rmrf:/# ls /dev
/dev/pts
```

And save our work.
```console
root@rmrf:/# echo 'ls() { printf '%s\n' ${1:+${1%/}/}*; }' >> utils.sh
root@rmrf:/# source utils.sh
```

How about `cat`? The `read` builtin comes in handy along with pipes and redirection, so we can piece together a rudimentary `cat`.

```console
root@rmrf:/# (while read line; do echo "$line"; done) < utils.sh
ls() { printf '%s\n' ${1:+${1%/}/}*; }
```

With these abilities and the fact that we can write arbitrary bytes with `echo`, we could rebuild and then `curl` or `wget` the binaries we want directly. My first choice, [echoed by others](http://eusebeia.dyndns.org/bashcp), would be to get [busybox](http://www.busybox.net/about.html). Busybox is the Swiss Army Knife of Embedded Linux, with builtin versions of `wget`, `dd`, `tar`, and many others. [Eusebeîa goes into great detail about how to get a fully escaped version of busybox on your system](http://eusebeia.dyndns.org/bashcp), so I won't do that here.

There is a problem though.

Even if we `echo` all the bytes we need into creating entire binaries, those files won't be executable. No way to start busybox. The easiest workaround for this is to find something which is executable and overwrite it with `echo`. We've nuked all of `/usr` and `/bin` at this point though, so that's a bit tricky.

We can use shell globs and bash logic to find files with the executable bit set, making sure to ignore directories.

```console
executable () { if [[ ( ! -d $1 ) && -x $1 ]] ; then echo "$1"; fi }
```

### Find the executables!

```console
root@rmrf:/# for file in /*; do executable $file; done
root@rmrf:/# for file in /*/*; do executable $file; done
root@rmrf:/# for file in /*/*/*; do executable $file; done
/proc/1107/exe
/proc/1136/exe
/proc/1149/exe
/proc/1179/exe
/proc/1215/exe
/proc/1217/exe
/proc/1220/exe
/proc/1221/exe
/proc/1223/exe
/proc/1248/exe
/proc/1277/exe
/proc/1468/exe
/proc/1478/exe
/proc/1625/exe
/proc/1644/exe
/proc/1/exe
/proc/374/exe
/proc/378/exe
/proc/471/exe
/proc/616/exe
/proc/657/exe
/proc/self/exe
```

Great! Hold on a minute though, those are all symbolic links to executables that no longer exist on disk. We'll just update `executable()` to ignore symbolic links.

```console
root@rmrf:/# executable () { if [[ ( ! -d $1 ) && ( ! -h $1 ) && -x $1 ]] ; then echo "$1"; fi }
root@rmrf:/# for file in /*/*/*; do executable $file; done
root@rmrf:/# for file in /*/*/*/*; do executable $file; done
root@rmrf:/# for file in /*/*/*/*/*; do executable $file; done
root@rmrf:/# for file in /*/*/*/*/*/*; do executable $file; done
```

Well now, that's bad news bears. Perhaps there is something at the kernel level we can use. After all, we can restart the box using sysrq *magic*:

```console
root@rmrf:/# echo 1 > /proc/sys/kernel/sysrq
root@rmrf:/# echo "b" > /proc/sysrq-trigger
```

~~Now we're locked out and I should do something else on a Friday. Thanks for reading! Let me know if you figure out how to get an executable bit set.~~

UPDATE: Redditor throw_away5046 posted [a full, beautiful, solution to this](http://www.reddit.com/r/linux/comments/27is0x/rm_rf_remains/ci199bk).

#### From a non-hosed, same architecture box:

```console
$ mkdir $(xxd -p -l 16 /dev/urandom)
$ cd $_
$ apt-get download busybox-static
$ dpkg -x *.deb .
$ alias encode='{ tr -d \\n | sed "s#\\(..\\)#\\\\x\\1#g"; echo; }'
$ alias upload='{ xxd -p | encode | nc -q0 -lp 5050; }'
$ upload < bin/busybox
```

#### Back on the rmrf'ed machine

```console
# cd /
# alias decode='while read -ru9 line; do printf "$line"; done'
# alias download='( exec 9<>/dev/tcp/{IP OF NON HOSED BOX}/5050; decode )'
# download > busybox
```

#### Now to create a shared object that will change permissions on busybox

```console
$ cat > setx.c <<EOF
extern int chmod(const char *pathname, unsigned int mode);

int entry(void) {

        return !! chmod("busybox", 0700);
}
char *desc[] = {0};

struct quick_hack {

        char *name; int (*fn)(void); int on;
        char **long_doc, *short_doc, *other;

} setx_struct = { "setx", entry, 1, desc, "chmod 0700 busybox", 0 };
EOF
$ gcc -Wall -Wextra -pedantic -nostdlib -Os -fpic -shared setx.c -o setx
$ upload < setx
```

#### Time to `enable` setx as a built in and get busybox executable
```console
# ( download > setx; enable -f ./setx setx; setx; )
# /busybox mkdir .bin
# /busybox  --install -s .bin
# PATH=/.bin
```

In action:

[![](https://d23f6h5jpj26xu.cloudfront.net/wwijp23ztne0tw_small.gif)](http://img.svbtle.com/wwijp23ztne0tw.gif)

---------

*For all my blog posts I've decided to hold discussion on Reddit, linking to the post. Today's post is on [/r/linux](http://www.reddit.com/r/linux/comments/27is0x/rm_rf_remains/) and on [/r/programming](http://www.reddit.com/r/programming/comments/27j311/rm_rf_remains/), but feel free to cross post it. PM me if you want me to link it here. Alternatively, you can [reach me on Twitter](https://twitter.com/rgbkrk).*

-------------

Go forth and build your <a target="_blank" href="http://www.amazon.com/s/?_encoding=UTF8&camp=1789&creative=390957&keywords=bash&linkCode=ur2&qid=1402239800&rh=n%3A283155%2Ck%3Abash&sort=relevancerank&tag=lamops-20&linkId=OXSOLI7W574KTESZ">bash knowledge</a>. Personally, I recommend <a href="http://www.amazon.com/gp/product/0596003307/ref=as_li_tl?ie=UTF8&camp=1789&creative=390957&creativeASIN=0596003307&linkCode=as2&tag=lamops-20">Unix Power Tools</a>.

<a href="http://www.amazon.com/gp/product/0596003307/ref=as_li_tl?ie=UTF8&camp=1789&creative=390957&creativeASIN=0596003307&linkCode=as2&tag=lamops-20"><img border="0" src="http://ws-na.amazon-adsystem.com/widgets/q?_encoding=UTF8&ASIN=0596003307&Format=_SL160_&ID=AsinImage&MarketPlace=US&ServiceVersion=20070822&WS=1&tag=lamops-20" style="display:block; margin-left:auto; margin-right:auto"></a>

Through college and career, flipping through this book has always showed me something new.


<img src="https://ir-na.amazon-adsystem.com/e/ir?t=lamops-20&l=as2&o=1&a=0596003307" width="1" height="1" border="0" style="border:none !important; margin:0px !important;" />
<img src="https://ir-na.amazon-adsystem.com/e/ir?t=lamops-20&l=ur2&o=1" width="1" height="1" border="0" style="border:none !important; margin:0px !important;" />


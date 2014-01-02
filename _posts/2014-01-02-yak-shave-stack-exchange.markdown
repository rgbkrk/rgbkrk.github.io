---
layout: post
title:  "Yaks=StackExchange;HTTPretty;urlparse"
date:   2014-01-02
image:  https://s3-us-west-2.amazonaws.com/s.cdpn.io/18885/httpretty-logo_1.svg 
description: I didn't want to shave a yak today, really. StackExchange and urlparse made me.
published: false
categories: [httpretty, testing, stackexchange, parse_qs]
---

I'm working on something that queries the StackExchange API for tagged questions.

The search API comes out looking like

```
/2.1/search?order=desc&sort=activity&tagged=python;ruby&site=stackoverflow
```

[Search](http://api.stackexchange.com/docs/search) requires tags to be separated by semi-colons. That's not a problem. Everything in my module works as I expect.

To test this, I brought in httpretty to mock calls to the API as it does the mocking for all my other calls.

In using httpretty though, the tagged field of the query string gets truncated down to just `python`. What gives?

{% highlight python %}
import requests
import httpretty

httpretty.enable()

httpretty.register_uri(httpretty.GET, "https://api.stackexchange.com/2.1/search", body='{"items":[]}')
resp = requests.get("https://api.stackexchange.com/2.1/search", params={"tagged":"python;ruby"})
httpretty_request = httpretty.last_request()
print(httpretty_request.querystring)

httpretty.disable()
httpretty.reset()

{% endhighlight %}

{% highlight bash %}
(semicolon) ~/code/HTTPretty$ python query_string_breakage.py
{u'tagged': [u'python']}
{% endhighlight %}

After some digging into httpretty's core code and a mini-session with ipdb, I find the culprit.

{% highlight python %}
In [1]: %run -d query_string_breakage.py
> /Users/rgbkrk/code/HTTPretty/query_string_breakage.py(5)<module>()
      4
----> 5 import requests
      6 import httpretty

ipdb> b httpretty/core.py:172
Breakpoint 1 at /Users/rgbkrk/code/HTTPretty/httpretty/core.py:172
ipdb> c
> /Users/rgbkrk/code/HTTPretty/httpretty/core.py(172)__init__()
    171         qstring = self.path.split("?", 1)[-1]
1-> 172         self.querystring = self.parse_querystring(qstring)
    173

ipdb> qstring
u'tagged=python%3Bruby'
ipdb> s
--Call--
> /Users/rgbkrk/code/HTTPretty/httpretty/core.py(185)parse_querystring()
    184
--> 185     def parse_querystring(self, qs):
    186         expanded = unquote_utf8(qs)

ipdb> n
> /Users/rgbkrk/code/HTTPretty/httpretty/core.py(186)parse_querystring()
    185     def parse_querystring(self, qs):
--> 186         expanded = unquote_utf8(qs)
    187         parsed = parse_qs(expanded)

ipdb> n
> /Users/rgbkrk/code/HTTPretty/httpretty/core.py(187)parse_querystring()
    186         expanded = unquote_utf8(qs)
--> 187         parsed = parse_qs(expanded)
    188         result = {}

ipdb> expanded
u'tagged=python;ruby'
ipdb> n
> /Users/rgbkrk/code/HTTPretty/httpretty/core.py(188)parse_querystring()
    187         parsed = parse_qs(expanded)
--> 188         result = {}
    189         for k in parsed:

ipdb> parsed
{u'tagged': [u'python']}
{% endhighlight %}

In my case, since I'm using Python 2.7 this is `parse_qs` from `urlparse` (`urllib.parse.parse_qs` in Python3).

Down to the smallest test case now:

{% highlight python %}
In [8]: urlparse.parse_qs("tagged=python;ruby")
Out[8]: {'tagged': ['python']}
{% endhighlight %}

My gut reaction, on poor sleep (I'm a Dad x 2!) was to immediately flip the table in front of me. I was also hoping it was just a simple fix in httpretty, one pull request away.

For shame.

Googling `parse_qs semicolon` immediately takes me to [a StackOverflow post asking "Why does Python's urlparse.parse_qs() split arguments on semicolon?"](http://stackoverflow.com/questions/5158565/why-does-pythons-urlparse-parse-qs-split-arguments-on-semicolon). The answer, of course, is to switch the asker's server to using commas instead of semicolons as the semicolon is equivalent to `&`. This doesn't solve it for me of course, as I can't change the StackExchange servers and API. Simply changing the request to use commas results in empty results.

What do I do now? Guess I'll write one of those blog things and come back to this in a second. Do I propose a hack for httpretty? Chat with the StackExchange folks?


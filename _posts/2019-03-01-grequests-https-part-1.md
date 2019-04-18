---
layout: post
title: The curious case of grequests and https - Part 1
tags:
- python
---


Table of Contents
=================

  * [Introduction](#introduction)
  * [Demo time!](#demo-time)
  * [Stage-0](#stage-0)
  * [Stage-1](#stage-1)
  * [Stage-2](#stage-2)
  * [Why is Stage-2 slow and Stage-1 fast with the same code?](#why-is-stage-2-slow-and-stage-1-fast-with-the-same-code)
  * [Monkey patching and re-patching.](#monkey-patching-and-re-patching)
  * [Profiling Stage-1 and Stage-2 code.](#profiling-stage-1-and-stage-2-code)
  * [Should I just uninstall pyopenssl and be fast again?](#should-i-just-uninstall-pyopenssl-and-be-fast-again)
  * [Summary](#summary)

<style type="text/css">
pre {
	width: 1000px;                          /* specify width  */
}
</style>

### Introduction

This post describes an interesting issue I ran into while using Python's grequests 
module.

If you have to download n pages of a site with the format ```https://site?page=n```
where n = 1 to 100 in python, the following snippet of code would be very slow as it would
download them one by one.

{% gist 6c904e3e6866c9c26e5f51d4883803dc %}

An obvious solution in the Python world is to use the [grequests](https://github.com/kennethreitz/grequests)
module, wherein one would write code similar to the following to get speed gains:

{% gist b08b8b3eb98432d786e4d6e712c0d4c9 %}

The above code runs faster because:

**grequests = [gevent](http://www.gevent.org/) + [requests](http://docs.python-requests.org/en/master/)**

where

**gevent = [greenlet](https://greenlet.readthedocs.io/en/latest/) + [libev](http://software.schmorp.de/pkg/libev.html)**

At a very high level something like this happens:

**When you use requests without gevent:**

1. Client opens a connection to the server
2. The client sends the request.
3. Client waits for server to respond.
4. Server responds.
5. Client goes back to Point 1. for next URL.

**When you use grequests:**

1. Client opens a connection to the server
2. The client sends the request.
3. The client does not wait for the server to respond (an IO wait), it returns control back to an event loop
   by registering a callback which is fired when the server responds.
4. Client does Point 1-3 for rest of the URLs.

**How does grequests accomplish this?**

1. grequests [imports gevent](https://github.com/kennethreitz/grequests/blob/98ff519f39d7456457a8b6e083766dd6d6f66e0b/grequests.py#L14)
   and uses gevent to [monkey patch](https://github.com/kennethreitz/grequests/blob/98ff519f39d7456457a8b6e083766dd6d6f66e0b/grequests.py#L21) the
   standard lib.
2. Now the underlying ```recv``` call is made non-blocking by gevent's library.

**What happens during monkey patching?**

<ol>
<li> Run the following code in your Python2.7 interpreter console: </li>

    {% highlight text %}
    >>> import inspect
    >>> import socket
    >>> inspect.getsourcefile(socket.ssl)
    '/usr/local/Cellar/python@2/2.7.15_3/Frameworks/Python.framework/Versions/2.7/lib/python2.7/socket.py'
    {% endhighlight %}

<li> Close and re-open your Python2.7 interpreter console and run similar code but after monkey patching through gevent: </li>
    {% highlight text %}
    >>> from gevent import monkey
    >>> monkey.patch_all()
    True
    >>> import inspect
    >>> import socket
    >>> inspect.getsourcefile(socket.ssl)
    '/usr/local/lib/python2.7/site-packages/gevent/_socket2.py'
    {% endhighlight %}

    As you noticed, <a href="https://github.com/gevent/gevent/blob/5f509a94382e9a04dd0dc1dbba63b14c499fb5f8/src/gevent/monkey.py#L975">monkey.patch_all</a> ensures 
    that if we reference any modules that gevent patches, we get their gevent variants instead of the vanilla ones.
</ol>

To understand **gevent** in more detail go through:

  - The awesome Kavya Joshi's amazing gevent depth talk [here](https://www.youtube.com/watch?v=GunMToxbE0E).
  - [This](http://blog.hownowstephen.com/post/50743415449/gevent-tutorial) equally excellent post.

Given this is how **gevent** works, using **grequests** to fetch 100 urls should do the trick. But as we will see in the
later sections, that depends on the Python modules installed on your system.

### Demo time!

Clone [this repo](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests) which 
has the sample setup. Run this [docker-compose command](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages#pre-requisites) to 
get the test environment up and running:

This will create the following containers on your system:

1. **https_server_1** - A local [golang https](https://github.com/mccutchen/go-httpbin) server 
   which mimics the very useful [httpbin.org](https://httpbin.org/), which provides
   the [/delay](https://httpbin.org/#/Dynamic_data/get_delay__delay_) endpoint to introduce
   latency in response.
2. **python27_n** where n = 1 to 4 - Python2.7 containers with different pip modules
   installed which show the situations in which the code is fast or slow.
3. **python37_n** where n = 1 to 4 - Same as above but for Python3.7 

We will go through these [stages](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages#stages) and see ```grequests``` behaviour.

Each stage is going to differ from the previous in the list of modules installed on the docker container. The code that we 
run in these stages is going to stay the same.

<a name="stage-0-main"></a>

### Stage-0

- Repo path: [here](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/00)

- Modules installed: Not applicable as we are not using virtualenv.

- Verdict: Python2.7 **may** run slow and Python3.7 **may** run fast.

- Experiment output 
  - [Python2.7 profiling output](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/00#fetch-urls-with-profiling)
  - [Python3.7 profiling output](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/00#fetch-urls-with-profiling-1)

- Reasoning: 
  As per the [disclaimer](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/00#disclaimer),
because this stage doesn't use virtualenv, the run time will depend on the Python modules you have installed on your system. 
We will uncover this behaviour in more depth in the coming stages.

<a name="stage-0-main"></a>

### Stage-1

- Repo path: [here](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/01)

- Modules installed (same for both):
  - [Python2.7 module list](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/01/python27#check-installed-modules)
  - [Python3.7 module list](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/01/python37#check-installed-modules)

- Verdict: [Python2.7](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/01/python27) runs fast and [ Python3.7](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/01/python37) runs fast.

- Experiment output 
  - [Python2.7 profiling output](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/01/python27#profile-code)
  - [Python3.7 profiling output](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/01/python37#profile-code)


<a name="stage-1-main"></a>

### Stage-2

- Repo path: [here](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/02)

- Modules installed (same for both):
  - [Python2.7 module list](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/02/python27#check-installed-modules)
  - [Python3.7 module list](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/02/python37#check-installed-modules)

- Verdict: [Python2.7](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/02/python27) runs **slow** and [ Python3.7](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/02/python37) runs **slow**.

- Experiment output 
  - [Python2.7 profiling output](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/02/python27#profile-code)
  - [Python3.7 profiling output](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/02/python37#profile-code)

### Why is Stage-2 slow and Stage-1 fast with the same code?

- We will show the reasoning for Python2.7 as it applies to Python3.7 also.
- The difference between [stage-2](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/02) and [stage-1](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/01)
  is that Stage-2 has the [pyopenssl](https://pypi.org/project/pyOpenSSL/) module installed. (compare the **Modules Installed** in above stages)
- Using pyopenssl does **something** which leads to the underlying socket class to change e.g. [Stage-1 Python2.7 socket class](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/01/python27#get-socket-class) 
  v/s [Stage-2 Python2.7 socket class](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/02/python27#get-socket-class)
- Stage-1 Python2.7 socket class = **gevent._sslgte279.SSLSocket** and Stage-2 Python2.7 socket class = **urllib3.contrib.pyopenssl.WrappedSocket**.
-  **grequests** internally uses [requests](https://github.com/kennethreitz/requests) which uses [urllib3](https://github.com/urllib3/urllib3)
- requests module, during initialization tries to include pyopenssl optionally as per [here.](https://github.com/kennethreitz/requests/blob/c4d76800e8917c3fda10cb1f2dee22c8219de3e6/requests/__init__.py#L95)
- If the user has **pyopenssl** installed, do ```pyopenssl.inject_into_urrllib3```. If the user doesn't have it installed, [this](https://github.com/urllib3/urllib3/blob/1e9ab5aee042ff0158d0f443bc600ef3a2e7bf9a/src/urllib3/contrib/pyopenssl.py#L46) call fails which is 
  caught by ```ImportError``` which does ```pass```, which made the Stage-1 ```ImportError``` silent.
- Using pyopenssl was introduced in [this](https://github.com/kennethreitz/requests/pull/1347/commits/18857a0eedcebbfc40bb1ae431daed935cd51d56) commit to take advantage of urllib3's [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) support.
- The side effect of using **pyopenssl** is that it **re-patches** the already monkey-patched [SSLContext](https://github.com/urllib3/urllib3/blob/1e9ab5aee042ff0158d0f443bc600ef3a2e7bf9a/src/urllib3/contrib/pyopenssl.py#L121) which denies us
  any gevent magic. [SSLContext](https://docs.python.org/3/library/ssl.html#ssl.SSLContext.wrap_socket) is used to add SSL capabilities
  to a socket.
- Let us see how this **re-patching** happened and why it led to slower code in the next 2 sections.

### Monkey patching and re-patching.

Let's use the already existing Stage-1 and Stage-2 Python interpreter console's to clear this out. Keep in mind that
Stage-2 had **pyopenssl** installed, while Stage-1 didn't. We will demo with Python2.7 as the same output applies to Python3.7 
also.

<ul>
  <li> Stage-1 Python2.7: </li>
  <ol>

  <li> Get the interpreter console: </li>

    {% highlight text %}
    ./docker-exec.sh test_grequests_python27_1 /usr/local/bin/python
    {% endhighlight %}

  <li> Import urllib3 and print the SSLContext: </li>

    {% highlight text %}
    >>> from __future__ import print_function
    >>> import urllib3
    >>> print(urllib3.util.ssl_.SSLContext)
    <class 'ssl.SSLContext'>
    {% endhighlight %}

  <li> Close and re-open the interpreter console and import urllib3 after monkey patching and then print the SSLContext:</li>

    {% highlight text %}
    >>> from __future__ import print_function
    >>> import gevent
    >>> from gevent import monkey
    >>> monkey.patch_all()
    True
    >>> import urllib3
    >>> print(urllib3.util.ssl_.SSLContext)
    <class 'gevent._sslgte279.SSLContext'>
    {% endhighlight %}

  <li> Close and re-open the interpreter console and import urllib3 after importing grequests then print the SSLContext:</li>

    {% highlight text %}
    >>> from __future__ import print_function
    >>> import grequests
    >>> import urllib3
    >>> print(urllib3.util.ssl_.SSLContext)
    <class 'gevent._sslgte279.SSLContext'>
    {% endhighlight %}

  </ol>
  Please note that in the last 2 cases, the SSLContext is provided by <b>gevent</b>.

  <li> Stage-2 Python2.7: </li>

  <ol>

  <li> Get the interpreter console: </li>

    {% highlight text %}
    ./docker-exec.sh test_grequests_python27_2 /usr/local/bin/python
    {% endhighlight %}

  <li> Import urllib3 and print the SSLContext: </li>

    {% highlight text %}
    >>> from __future__ import print_function
    >>> import urllib3
    >>> print(urllib3.util.ssl_.SSLContext)
    <class 'ssl.SSLContext'>
    {% endhighlight %}

  <li> Close and re-open the interpreter console and import urllib3 after monkey patching and then print the SSLContext:</li>

    {% highlight text %}
    >>> from __future__ import print_function
    >>> import gevent
    >>> from gevent import monkey
    >>> monkey.patch_all()
    True
    >>> import urllib3
    >>> print(urllib3.util.ssl_.SSLContext)
    <class 'gevent._sslgte279.SSLContext'>
    {% endhighlight %}

  <li> Close and re-open the interpreter console and import urllib3 after importing grequests then print the SSLContext:</li>

    {% highlight text %}
    >>> from __future__ import print_function
    >>> import grequests
    >>> import urllib3
    >>> print(urllib3.util.ssl_.SSLContext)
    <class 'urllib3.contrib.pyopenssl.PyOpenSSLContext'>

    {% endhighlight %}

  </ol>

  Please note that in the last case, the SSLContext is no longer provided by gevent because the presence 
  of <b>pyopenssl</b> triggered the repatching. This does not happen in Stage-1 - now let us see how this 
  impacts the run time.
</ul>

### Profiling Stage-1 and Stage-2 code.

1. [Stage-1 Python2.7 profiling output](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/01/python27#profile-code)
1. [Stage-2 Python2.7 profiling output](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/02/python27#profile-code)

If you observe the output for Stage-2 you will see that there is a call to **wait**:

{% highlight text %}
30    0.001    0.000   10.042    0.335 /usr/local/lib/python2.7/site-packages/urllib3/util/wait.py:99(do_poll)
{% endhighlight %}

The source of ```/usr/local/lib/python2.7/site-packages/urllib3/util/wait.py:99(do_poll)```:

{% highlight text %}
def do_poll(t):
    if t is not None:
        t *= 1000
    return poll_obj.poll(t)

return bool(_retry_on_intr(do_poll, timeout))
{% endhighlight %}

We see that there is an explicit wait of 1 second (1000 ms) happening on when the socket is blocked. One more interesting observation is 
that for Stage-2 the **recv** socket call is made from **pyopenssl.py.** which leads to polled waiting:

{% highlight text %}
20/10    0.002    0.000   10.030    1.003 /usr/local/lib/python2.7/site-packages/urllib3/contrib/pyopenssl.py:271(recv)
{% endhighlight %}

 For Stage-1 the **recv** socket call is made from **gevent** which doesn't get blocked on IO and moves on to processing the
 next request:

{% highlight text %}
10/1    0.001    0.000    1.294    1.294 /usr/local/lib/python2.7/site-packages/gevent/_sslgte279.py:448(recv)
{% endhighlight %}

Hence there is no polled wait.

### Should I just uninstall pyopenssl and be fast again?

You can do that and get the speed gains. But as per [this](https://pyopenssl.org/en/stable/introduction.html#history):

{% highlight text %}
pyOpenSSL was originally created by Martin Sjögren because the SSL support in the standard library in Python 2.1 
(the contemporary version of Python when the pyOpenSSL project was begun) was severely limited. Other OpenSSL wrappers 
for Python at the time were also limited, though in different ways.
{% endhighlight %}

In my case, I wanted to do HTTPS verification (verify=True flag in requests). The latest [ssl](https://docs.python.org/3/library/ssl.html) library has these features.

{% highlight text %}
$ python2
>>> import ssl
>>> ssl.HAS_SNI
True
>>> import requests
>>> requests.get('https://google.com', verify=True)
<Response [200]>
{% endhighlight %}

But if you are using Python older than 2.7.9, then you have to use **pyopenssl** for the above features:

The following commands are run on Python 2.7.8 interpreter without **pyopenssl**:

{% highlight text %}
$ ~/.pyenv/versions/2.7.8/bin/python
>>> import ssl
>>> ssl.HAS_SNI
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'module' object has no attribute 'HAS_SNI'
>>> import requests
>>> requests.get('https://google.com', verify=True)
~/.pyenv/versions/2.7.8/lib/python2.7/site-packages/urllib3/util/ssl_.py:354: SNIMissingWarning: An HTTPS request has been made, but the SNI (Server Name Indication) extension to TLS is not available on this platform. This may cause the server to present an incorrect TLS certificate, which can cause validation failures. You can upgrade to a newer version of Python to solve this. For more information, see https://urllib3.readthedocs.io/en/latest/advanced-usage.html#ssl-warnings
  SNIMissingWarning
~/.pyenv/versions/2.7.8/lib/python2.7/site-packages/urllib3/util/ssl_.py:150: InsecurePlatformWarning: A true SSLContext object is not available. This prevents urllib3 from configuring SSL appropriately and may cause certain SSL connections to fail. You can upgrade to a newer version of Python to solve this. For more information, see https://urllib3.readthedocs.io/en/latest/advanced-usage.html#ssl-warnings
  InsecurePlatformWarning
~/.pyenv/versions/2.7.8/lib/python2.7/site-packages/urllib3/util/ssl_.py:150: InsecurePlatformWarning: A true SSLContext object is not available. This prevents urllib3 from configuring SSL appropriately and may cause certain SSL connections to fail. You can upgrade to a newer version of Python to solve this. For more information, see https://urllib3.readthedocs.io/en/latest/advanced-usage.html#ssl-warnings
  InsecurePlatformWarning
<Response [200]>
{% endhighlight %}

If we install **pyopenssl** and retry verification, it goes through:

{% highlight text %}
$ ~/.pyenv/versions/2.7.8/bin/pip install PyOpenSSL
$ ~/.pyenv/versions/2.7.8/bin/python
>>> import ssl
>>> ssl.HAS_SNI
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'module' object has no attribute 'HAS_SNI'
>>> import requests
>>> requests.get('https://google.com', verify=True)
<Response [200]>
{% endhighlight %}

But what if you need to use other features of **pyopenssl** related to certificate management which may or may not be
in the core **ssl** library? [gevent-openssl](https://github.com/mjs/gevent_openssl) has got you covered. It is a 
**gevent** wrapper over **pyopenssl** so that you can use **pyopenssl** and prevent the re-patching scenario that we 
ran into.

However, installing **gevent-openssl** solves the problem of running fast with **pyopenssl** on Python2.7 but it doesn't
do so for Python3.7. We will learn more about the same in my [next post](http://saurabh-hirani.github.io/writing/2019/04/01/grequests-https-part-2) as we explore [Stage-3](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03) and [Stage-4](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/04).

### Summary

- [Stage-0](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/00) is **unpredictable** because we are not using virtualenv.
- [Stage-1](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/01) verifies that the absence of **pyopenssl** makes both Python 2.7 and Python 3.7 **fast**.
- [Stage-2](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/02) verifies that the presence of **pyopenssl** makes both Python 2.7 and Python 3.7 **slow**.
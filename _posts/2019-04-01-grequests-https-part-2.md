---
layout: post
title: The curios case of grequests and https - Part 2
tags:
- python
---


Table of Contents
=================

  * [Introduction](#introduction)
  * [Quick recap](#quick-recap)
  * [grequests with gevent-openssl code change:](#grequests-with-gevent-openssl-code-change)
  * [Stage-3](#stage-3)
  * [Stage-3 - Why is Python27 fast and Python3.7 slow?](#stage-3---why-is-python27-fast-and-python37-slow)
  * [Stage-4](#stage-4)
  * [Stage-4 - What did we do to make Python3.7 fast?](#stage-4---what-did-we-do-to-make-python37-fast)
  * [Summary](#summary)

<style type="text/css">
pre {
	width: 1000px;                          /* specify width  */
}
</style>

### Introduction

This is the second part of a two part post describing how the Python modules installed
on your system can impact run time of a **grequests** based program.

Read Part-I [here](http://saurabh-hirani.github.io/writing/2019/03/01/grequests-https-part-1).

### Quick recap


- Problem: Doing **GET** type calls on 100s of urls using [requests](http://docs.python-requests.org/en/master/) is slow due to serial processing.
- Potential solution: Using [grequests](https://github.com/kennethreitz/grequests) which leverages [gevent](https://github.com/gevent/gevent) helps us make concurrent calls.
- Issue: If you have [pyopenssl](https://pyopenssl.org/en/stable/) installed on your system, [urllib3](https://urllib3.readthedocs.io/en/latest/) re-patches the **SSLContext** and makes your 
  code run slower. This was demoed by [stage-1](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/01) and [stage-2](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/02).
- Work around solution: 
  - Don't use **pyopenssl** OR 
  - Use [gevent-openssl](https://github.com/mjs/gevent_openssl) which was created to make **pyopenssl** gevent compatible.

Using **gevent-openssl** will certainly make Python2.7 code run faster but it will not give you
speed gains for Python3.7. Find out more in the coming sections which explore Stage-3 and Stage-4.

### grequests with gevent-openssl code change:

<ul>
<li>As per <a href="https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/blob/master/bin/test_grequests_v2.py#L27#L31">bin/test_grequests_v2.py</a>, if you
  plan to use <b>gevent_openssl</b>, you have to add the following code before importing grequests: </li>

  {% highlight text %}
  try:
    import gevent_openssl
    gevent_openssl.monkey_patch()
  except ImportError as ex:
    pass
  {% endhighlight %}

<li>We are letting the absence of <b>gevent_openssl</b> exception pass because we want to use the same code throughout our stages.</li>
</ul>

### Stage-3

- Repo path: [here](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03)

- Modules installed (same for both):
  - [Python2.7 module list](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python27#check-installed-modules)
  - [Python3.7 module list](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python37#check-installed-modules)

- Verdict: [Python2.7](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python27) runs **slow** and [ Python3.7](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python37) runs **slow**.

- Experiment output 
  - [Python2.7 profiling output](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python27#profile-code)
  - [Python3.7 profiling output](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python37#profile-code)

### Stage-3 - Why is Python27 fast and Python3.7 slow?

<ul>
  <li><a href="https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python27#get-socket-class">Python 2.7 socket class</a> and <a href="https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python37#get-socket-class">Python 3.7 socket class</a> are the same - <b>urllib3.contrib.pyopenssl.WrappedSocket</b>, which means that the <b>gevent</b> issue that we saw in earlier stages is not at play.</li>
  <li> There is one subtle difference in <a href="https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python27#profile-code">Python 2.7 profiling output</a> and <a href="https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python37#profile-code">Python 3.7 profiling output.</a></li>

  <ol>
    <li> Python2.7 profile output shows calls to <b>pyopenssl recv</b>: </li>
        {% highlight text %}
        /usr/local/lib/python2.7/site-packages/urllib3/contrib/pyopenssl.py:271(recv)
        {% endhighlight %}
    <li> Python3.7 profile output shows calls to <b>pyopenssl recv_into</b>: </li>
        {% highlight text %}
        /usr/local/lib/python3.7/site-packages/urllib3/contrib/pyopenssl.py:292(recv_into)
        {% endhighlight %}
    <li> You can also trace <a href="https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python27#trace-code">Python 2.7 function calls</a> and <a href="https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python37#trace-code">Python 3.7 function calls</a> to verify the same.</li>
  </ol>

  <li> Checking <a href="https://github.com/mjs/gevent_openssl/blob/645ded94710d886bce671c2f001d30643242b3cd/gevent_openssl/SSL.py#L61">gevent_openssl/SSL.py:recv</a>, we see that it overrides <a href="https://github.com/urllib3/urllib3/blob/1e9ab5aee042ff0158d0f443bc600ef3a2e7bf9a/src/urllib3/contrib/pyopenssl.py#L277">urllib3/contrib/pyopenssl.py:recv</a> </li>
  <li> There are no functions in <a href="https://github.com/mjs/gevent_openssl/blob/645ded94710d886bce671c2f001d30643242b3cd/gevent_openssl/SSL.py">gevent_openssl/SSL.py</a> to override <a href="https://github.com/urllib3/urllib3/blob/1e9ab5aee042ff0158d0f443bc600ef3a2e7bf9a/src/urllib3/contrib/pyopenssl.py#L302">urllib3/contrib/pyopenssl.py:recv_into</a> </li>
  <li> Python 2.7 uses <b>recv</b> call and can leverage gevent-openssl's patched <b>recv</b> function, while Python 3.7 uses <b>recv_into</b> which has no corresponding gevent-openssl function and hence it falls back to <a href="https://github.com/urllib3/urllib3/blob/1e9ab5aee042ff0158d0f443bc600ef3a2e7bf9a/src/urllib3/contrib/pyopenssl.py#L302">urllib3/contrib/pyopenssl.py:recv_into</a>, which is slow. </li>

</ul>

### Stage-4

- Repo path: [here](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/04)

- Modules installed (same for both):
  - [Python2.7 module list](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/04/python27#check-installed-modules)
  - [Python3.7 module list](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/04/python37#check-installed-modules)

- Verdict: [Python2.7](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/04/python27) runs **slow** and [ Python3.7](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python37) runs **slow**.

- Experiment output 
  - [Python2.7 profiling output](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/04/python27#profile-code)
  - [Python3.7 profiling output](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/04/python37#profile-code)

### Stage-4 - What did we do to make Python3.7 fast?

<ul>

  <li>We patched <a href="https://github.com/mjs/gevent_openssl/blob/c9e2f094b33fc70b4007331c3311c42c85184a24/gevent_openssl/SSL.py">gevent_openssl/SSL.py local copy</a> to add support to override <a href="https://github.com/urllib3/urllib3/blob/1e9ab5aee042ff0158d0f443bc600ef3a2e7bf9a/src/urllib3/contrib/pyopenssl.py#L302">urllib3/contrib/pyopenssl.py:recv_into</a></li>
  <li>The <a href="https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/blob/master/patches/gevent_openssl_ssl.patch">patch</a> adds the following function:</li>

    {% highlight text %}
    def recv_into(self, buffer, nbytes=None, flags=None):
        try:
            return self.__iowait(self._connection.recv_into, buffer, nbytes, flags)
        except OpenSSL.SSL.ZeroReturnError:
            return ''
        except OpenSSL.SSL.SysCallError as e:
            if e[0] == -1 and 'Unexpected EOF' in e[1]:
                # errors when reading empty strings are expected and can be
                # ignored
                return ''
            raise
    {% endhighlight %}

  <li> It is an similar to the existing <a href="https://github.com/mjs/gevent_openssl/blob/c9e2f094b33fc70b4007331c3311c42c85184a24/gevent_openssl/SSL.py#L61">gevent_openssl/SSL.py:recv</a> function. </li>
  <li> We are wrapping <b>self._connection.recv_into</b> by <b>self.__iowait</b>, which as per <a href="https://github.com/mjs/gevent_openssl/blob/c9e2f094b33fc70b4007331c3311c42c85184a24/gevent_openssl/SSL.py#L24">gevent_openssl/SSL.py:__iowait</a> does the following: </li>

    {% highlight text %}
    def __iowait(self, io_func, *args, **kwargs):
        fd = self._sock.fileno()
        timeout = self._sock.gettimeout()
        while True:
            try:
                return io_func(*args, **kwargs)
            except (OpenSSL.SSL.WantReadError, OpenSSL.SSL.WantX509LookupError):
                wait_read(fd, timeout=timeout)
            except OpenSSL.SSL.WantWriteError:
                wait_write(fd, timeout=timeout)
    {% endhighlight %}

  <li> It calls the passed <b>io_func</b> e.g recv, recv_into, etc. and as soon as it gets <b>OpenSSL.SSL.WantReadError</b>, which is raised by <a href="https://github.com/pyca/pyopenssl/blob/a42c5c9a91639c1d4405e67316046cb6b939ac84/src/OpenSSL/SSL.py#L1625">OpenSSL/SSL.py:_raise_ssl_error</a> it returns control back to <a href="https://github.com/gevent/gevent/blob/422a71266e2f0551f68a1b339a4d640424614025/src/gevent/__hub_primitives.pxd#L70">gevent/_hub_primitives.pxd</a> </li>
  <li> A <a href="https://cython.readthedocs.io/en/latest/src/tutorial/pxd_files.html">pxd</a> file is like a C-header file, which means when we call <b>wait_read</b> we wade into C-extensions created by gevent, which makes our code faster.</li>
  <li> You can <a href="https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/04/python37#toggle-patch-to-verify">toggle the patching</a> and see a corresponding impact on the execution time. </li>

</ul>

I have opened a [Pull request](https://github.com/mjs/gevent_openssl/pull/15) to add this functionality to gevent_openssl.


### Summary

- [Stage-0](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/00) is **unpredictable** because we are not using virtualenv.
- [Stage-1](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/01) verifies that the absence of **pyopenssl** makes both Python 2.7 and Python 3.7 **fast**.
- [Stage-2](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/02) verifies that the presence of **pyopenssl** makes both Python 2.7 and Python 3.7 **slow**.
- [Stage-3](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03) verifies that the presence of **gevent_openssl** with **pyopenssl** makes Python 2.7 **fast** but Python 3.7 **slow**.
- [Stage-3](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/04) patches the existing **gevent_openssl** to make Python 3.7 **fast**.
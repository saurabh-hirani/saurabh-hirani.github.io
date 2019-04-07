---
layout: post
title: The curios case of grequests and https - Part 2
tags:
- python
---



<style type="text/css">
pre {
	width: 1000px;                          /* specify width  */
}
</style>

### Introduction

In the previous [post](http://saurabh-hirani.github.io/writing/2019/03/01/grequests-https-part-1) we looked at
how the absence/presence of **pyopenssl** can make the same **grequests** driven code faster or slower.

Before we go further - a quick recap:

- Problem: Doing **GET** type calls on 100s of urls using [requests](http://docs.python-requests.org/en/master/) is slow due to serial processing.
- Potential solution: Using [grequests](https://github.com/kennethreitz/grequests) which leverages [gevent](https://github.com/gevent/gevent) helps us make concurrent calls.
- Issue: If you have [pyopenssl](https://pyopenssl.org/en/stable/) installed on your system, [urllib3](https://urllib3.readthedocs.io/en/latest/) re-patches the **SSLContext** and makes your 
  code run slower. This was demoed by [stage-1](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/01) and [stage-2](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/02).
- Work around solution: 
  - Don't use **pyopenssl** OR 
  - Use [gevent-openssl](https://github.com/mjs/gevent_openssl) which was created to make **pyopenssl** gevent compatible.

Using **gevent-openssl** will certainly make Python2.7 code run faster but it will not give you
speed gains for Python3.7. Let us try out Stage-3 and Stage-4 to compare.


### Stage-3

- Repo path: [here](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03)

- Verdict: Python2.7 runs fast and Python3.7 runs **slow**.

- Experiment output - [here.](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03)

- Modules installed (same for both):
  - [Python2.7](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python27#check-installed-modules)
  - [Python3.7](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/tree/master/stages/03/python37#check-installed-modules)

### Stage-3 - Why is Python27 fast and Python3.7 slow?
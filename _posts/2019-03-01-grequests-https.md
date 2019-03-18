---
layout: post
title: The curios case of grequests and https
tags:
- python
---

This post describes an interesting issue I ran into while using Python's grequests 
module.

<style type="text/css">
pre {
	width: 1000px;                          /* specify width  */
}

</style>

### Introduction

If you have to download n pages of a site with the format ```https://site?page=n```
where n = 1 to 100, the following snippet of code would be very slow as it would
download them one by one.

{% gist 6c904e3e6866c9c26e5f51d4883803dc %}

An obvious solution in the Python world is to use the [grequests](https://github.com/kennethreitz/grequests)
module, wherein one would write code similar to the following:

{% gist b08b8b3eb98432d786e4d6e712c0d4c9 %}

**grequests = [gevent](http://www.gevent.org/) + [requests](http://docs.python-requests.org/en/master/)**

where

**gevent = greenlet + libev**

I won't even try to explain what gevent does in detail because the amazing 
Kavya Joshi has explained it in depth in [this](https://www.youtube.com/watch?v=GunMToxbE0E) 
awesome talk.

[This](http://blog.hownowstephen.com/post/50743415449/gevent-tutorial) equally excellent post walks
through examples to explain how you can use gevent and explains the importance of [monkey-patching](https://en.wikipedia.org/wiki/Monkey_patch)
in gevent.

But at a very high level something like this happens:

Life before gevent:

1. Client opens a connection to the server
2. The client sends the request.
3. Client waits for server to respond.
4. Server responds.
5. Client goes back to Point 1. for next URL.

With gevent the flow becomes:

1. Client opens a connection to the server
2. The client sends the request.
3. The client does not wait for the server to respond (an IO wait), it returns control back to an event loop
   by registering a callback which is fired when the server responds.
4. Client does Point 1-3 for rest of the URLs.

So everthing is sorted, right? If it was then I wouldn't be writing this post.

What if I told you your program **might** do serial URL calls under the hood even if you use
**grequests**? You would say - talk is cheap. Show me the code.

Clone [this repo](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests) which 
has the sample setup. Follow these steps to get the test environment up and running:

1. ```docker-compose --project-name test_grequests up```

This will create the following containers on your system:

1. **https_server_1** - A local [golang https](https://github.com/mccutchen/go-httpbin) server 
   which mimics the very useful [httpbin.org](https://httpbin.org/), which provides
   the [/delay](https://httpbin.org/#/Dynamic_data/get_delay__delay_) endpoint to introduce
   latency in response.
2. **python27_n** where n = 1 to 4 - Python 2.7 containers with different pip modules
   installed which show the situations in which the code is fast or slow.
3. **python37_n** where n = 1 to 4 - Same as above but for Python 3.7

We will go through the following stages and see ```grequests``` behaviour:

<a name="all-stages">

1. [Stage-1 Python 2.7](#stage-1-python27): runs fast.
2. [Stage-1 Python 3.7](#stage-1-python37): runs fast.
3. [Stage-2 Python 2.7](#stage-2-python27): runs slow.
4. [Stage-2 Python 3.7](#stage-2-python37): runs slow.
5. [Stage-3 Python 2.7](#stage-3-python27): runs fast.
6. [Stage-3 Python 3.7](#stage-3-python37): runs slow.
7. [Stage-4 Python 2.7](#stage-4-python27): runs fast. 
8. [Stage-4 Python 3.7](#stage-4-python37): runs fast or slow depending on a patch applied to a Python library.

Each stage is going to differ from the previous in the list of modules installed on the docker container. The
code that we run against these Python environments is going to stay the same.

<a name="stage-1-main"></a>

### Stage-1 - Python 2.7 and Python 3.7 run fast

<a name="stage-1-python27"></a>

#### Stage-1 - Python 2.7 - runs fast

[Back to all stages](#all-stages)

Run the following command and watch what happens:

{% highlight text %}
# ./docker-exec.sh test_grequests_python27_1 \
  '/usr/local/bin/python /app/test_grequests_v2.py \
  --url https://https_server:8081/delay/1 --url-count 10'
{% endhighlight %}

```./docker-exec.sh``` matches the container(s) specified in the first argument
and runs the command specified in the second argument on those container(s).

The above command gives an output similar to the following:

{% highlight text %}
================================
--------------------------------
urllib3==1.24.1
requests==2.21.0
gevent==1.4.0
grequests==0.3.0
--------------------------------
+ docker exec -it test_grequests_python27_1 /usr/local/bin/python /app/test_grequests_v2.py --url https://https_server:8081/delay/1 --url-count 10
2019-03-17 15:30:54,977 - python27_1 - START
2019-03-17 15:30:56,015 - python27_1 - len(all_page_ids) = 10
2019-03-17 15:30:56,016 - python27_1 - len(valid_page_ids) = 10
2019-03-17 15:30:56,016 - python27_1 - len(invalid_page_ids) = 0
2019-03-17 15:30:56,019 - python27_1 - total_time=0:00:01.048203
2019-03-17 15:30:56,020 - python27_1 - END
+ set +x
================================
{% endhighlight %}

It prints the [modules installed for that container](https://github.com/saurabh-hirani/grequests-https-python-27-37-tests/blob/master/stages/01/python27/requirements.txt) followed by the command output. 
It clearly shows that concurrent requests are being made and a GET on 
10 URLs is getting completed in around 1 second.

<a name="stage-1-python37"></a>

#### Stage-1 - Python 3.7 - runs fast

[Back to all stages](#all-stages)

{% highlight text %}
# ./docker-exec.sh test_grequests_python37_1 \
  '/usr/local/bin/python /app/test_grequests_v2.py \
  --url https://https_server:8081/delay/1 --url-count 10'
{% endhighlight %}

The above command gives an output similar to the following:

{% highlight text %}
================================
--------------------------------
urllib3==1.24.1
requests==2.21.0
gevent==1.4.0
grequests==0.3.0
--------------------------------
+ docker exec -it test_grequests_python37_1 /usr/local/bin/python /app/test_grequests_v2.py --url https://https_server:8081/delay/1 --url-count 10
2019-03-17 15:33:34,920 - python37_1 - START
2019-03-17 15:33:35,973 - python37_1 - len(all_page_ids) = 10
2019-03-17 15:33:35,973 - python37_1 - len(valid_page_ids) = 10
2019-03-17 15:33:35,973 - python37_1 - len(invalid_page_ids) = 0
2019-03-17 15:33:35,975 - python37_1 - total_time=0:00:01.060971
2019-03-17 15:33:35,976 - python37_1 - END
+ set +x
================================
{% endhighlight %}

No surprises here. Similar run time.

<a name="stage-2-main"></a>

### Stage-2 - Python 2.7 and Python 3.7 run slow

<a name="stage-2-python27"></a>

#### Stage-2 - Python 2.7 - runs slow

[Back to all stages](#all-stages)

{% highlight text %}
# ./docker-exec.sh test_grequests_python27_2 \
  '/usr/local/bin/python /app/test_grequests_v2.py \
  --url https://https_server:8081/delay/1 --url-count 10'
{% endhighlight %}


{% highlight text %}
================================
--------------------------------
urllib3==1.24.1
requests==2.21.0
gevent==1.4.0
grequests==0.3.0
pyopenssl==19.0.0
--------------------------------
+ docker exec -it test_grequests_python27_2 /usr/local/bin/python /app/test_grequests_v2.py --url https://https_server:8081/delay/1 --url-count 10
2019-03-17 15:35:36,704 - python27_2 - START
2019-03-17 15:35:46,805 - python27_2 - len(all_page_ids) = 10
2019-03-17 15:35:46,806 - python27_2 - len(valid_page_ids) = 10
2019-03-17 15:35:46,806 - python27_2 - len(invalid_page_ids) = 0
2019-03-17 15:35:46,807 - python27_2 - total_time=0:00:10.107268
2019-03-17 15:35:46,808 - python27_2 - END
+ set +x
================================
{% endhighlight %}

There are 2 major differences from [Stage-1 Python 2.7](#stage-1-python27):

1. We installed the ```pyopenssl``` module.
2. The same code's run time spiked from around 1 sec to 10 seconds i.e. around 1 second for each URL.

**What happened?**

Let's enable some more debugging by adding the flag ```--log-level DEBUG``` which 
prints the ```requests``` module connection calls also. 

{% highlight text %}
# ./docker-exec.sh test_grequests_python27_2 \
  '/usr/local/bin/python /app/test_grequests_v2.py \
  --url https://https_server:8081/delay/1 --url-count 10 --log-level DEBUG'
{% endhighlight %}

{% highlight text %}
================================
--------------------------------
urllib3==1.24.1
requests==2.21.0
gevent==1.4.0
grequests==0.3.0
pyopenssl==19.0.0
--------------------------------
+ docker exec -it test_grequests_python27_2 /usr/local/bin/python /app/test_grequests_v2.py --url https://https_server:8081/delay/1 --url-count 10 --log-level DEBUG
2019-03-18 12:17:31,782 - python27_2 - START
2019-03-18 12:17:31,790 - python27_2 - Starting new HTTPS connection (1): https_server:8081
2019-03-18 12:17:31,798 - python27_2 - Starting new HTTPS connection (1): https_server:8081
2019-03-18 12:17:31,800 - python27_2 - Starting new HTTPS connection (1): https_server:8081
2019-03-18 12:17:31,802 - python27_2 - Starting new HTTPS connection (1): https_server:8081
2019-03-18 12:17:31,803 - python27_2 - Starting new HTTPS connection (1): https_server:8081
2019-03-18 12:17:31,805 - python27_2 - Starting new HTTPS connection (1): https_server:8081
2019-03-18 12:17:31,808 - python27_2 - Starting new HTTPS connection (1): https_server:8081
2019-03-18 12:17:31,811 - python27_2 - Starting new HTTPS connection (1): https_server:8081
2019-03-18 12:17:31,813 - python27_2 - Starting new HTTPS connection (1): https_server:8081
2019-03-18 12:17:31,814 - python27_2 - Starting new HTTPS connection (1): https_server:8081
2019-03-18 12:17:32,830 - python27_2 - https://https_server:8081 "GET /delay/1?page=6 HTTP/1.1" 200 308
2019-03-18 12:17:33,841 - python27_2 - https://https_server:8081 "GET /delay/1?page=5 HTTP/1.1" 200 308
2019-03-18 12:17:34,849 - python27_2 - https://https_server:8081 "GET /delay/1?page=4 HTTP/1.1" 200 308
2019-03-18 12:17:35,856 - python27_2 - https://https_server:8081 "GET /delay/1?page=3 HTTP/1.1" 200 308
2019-03-18 12:17:36,866 - python27_2 - https://https_server:8081 "GET /delay/1?page=2 HTTP/1.1" 200 308
2019-03-18 12:17:37,876 - python27_2 - https://https_server:8081 "GET /delay/1?page=1 HTTP/1.1" 200 308
2019-03-18 12:17:38,885 - python27_2 - https://https_server:8081 "GET /delay/1?page=0 HTTP/1.1" 200 308
2019-03-18 12:17:39,898 - python27_2 - https://https_server:8081 "GET /delay/1?page=9 HTTP/1.1" 200 308
2019-03-18 12:17:40,911 - python27_2 - https://https_server:8081 "GET /delay/1?page=7 HTTP/1.1" 200 308
2019-03-18 12:17:41,919 - python27_2 - https://https_server:8081 "GET /delay/1?page=8 HTTP/1.1" 200 308
2019-03-18 12:17:41,921 - python27_2 - len(all_page_ids) = 10
2019-03-18 12:17:41,921 - python27_2 - len(valid_page_ids) = 10
2019-03-18 12:17:41,921 - python27_2 - len(invalid_page_ids) = 0
2019-03-18 12:17:41,922 - python27_2 - total_time=0:00:10.144756
2019-03-18 12:17:41,922 - python27_2 - END
+ set +x
================================
{% endhighlight %}

This shows that each url was being called after the previous one finished. Let's get some more detail
by profiling the code and seeing what is taking time by using the flags ```--profile-code``` and 
```--profile-stats-count 20``` to show the top 20 time consuming calls.

{% highlight text %}
# ./docker-exec.sh test_grequests_python27_2 \
  '/usr/local/bin/python /app/test_grequests_v2.py \
  --url https://https_server:8081/delay/1 --url-count 10 \
  --profile-code --profile-stats-count 20'
{% endhighlight %}

{% highlight text %}
===============================                                                                                                                                                                                                    [5/5914]
--------------------------------
urllib3==1.24.1
requests==2.21.0
gevent==1.4.0
grequests==0.3.0
pyopenssl==19.0.0
--------------------------------
+ docker exec -it test_grequests_python27_2 /usr/local/bin/python /app/test_grequests_v2.py --url https://https_server:8081/delay/1 --url-count 10 --profile-code --profile-stats-count 20
2019-03-18 12:19:37,223 - python27_2 - START
2019-03-18 12:19:47,699 - python27_2 - len(all_page_ids) = 10
2019-03-18 12:19:47,699 - python27_2 - len(valid_page_ids) = 10
2019-03-18 12:19:47,699 - python27_2 - len(invalid_page_ids) = 0
         23563 function calls (23266 primitive calls) in 10.472 seconds

   Ordered by: cumulative time
   List reduced from 535 to 20 due to restriction <20>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000   10.473   10.473 /usr/local/lib/python2.7/site-packages/grequests.py:103(map)
     10/1    0.000    0.000   10.472   10.472 /usr/local/lib/python2.7/site-packages/grequests.py:60(send)
     10/1    0.000    0.000   10.472   10.472 /usr/local/lib/python2.7/site-packages/requests/sessions.py:466(request)
     10/1    0.001    0.000   10.447   10.447 /usr/local/lib/python2.7/site-packages/requests/sessions.py:617(send)
     10/1    0.001    0.000   10.445   10.445 /usr/local/lib/python2.7/site-packages/requests/adapters.py:394(send)
     10/1    0.001    0.000   10.436   10.436 /usr/local/lib/python2.7/site-packages/urllib3/connectionpool.py:446(urlopen)
     10/1    0.002    0.000   10.433   10.433 /usr/local/lib/python2.7/site-packages/urllib3/connectionpool.py:319(_make_request)
       10    0.000    0.000   10.034    1.003 /usr/local/lib/python2.7/httplib.py:1084(getresponse)
       10    0.001    0.000   10.033    1.003 /usr/local/lib/python2.7/httplib.py:431(begin)
       26    0.001    0.000   10.008    0.385 /usr/local/lib/python2.7/site-packages/urllib3/util/wait.py:139(wait_for_read)
       26    0.001    0.000   10.007    0.385 /usr/local/lib/python2.7/site-packages/urllib3/util/wait.py:87(poll_wait_for_socket)
       70    0.008    0.000   10.005    0.143 /usr/local/lib/python2.7/socket.py:410(readline)
       27    0.001    0.000   10.003    0.370 /usr/local/lib/python2.7/site-packages/urllib3/util/wait.py:45(_retry_on_intr)
       26    0.001    0.000   10.002    0.385 /usr/local/lib/python2.7/site-packages/urllib3/util/wait.py:99(do_poll)
       27   10.001    0.370   10.001    0.370 {built-in method poll}
       10    0.001    0.000    9.994    0.999 /usr/local/lib/python2.7/httplib.py:392(_read_status)
    20/10    0.002    0.000    9.991    0.999 /usr/local/lib/python2.7/site-packages/urllib3/contrib/pyopenssl.py:271(recv)
     10/1    0.001    0.000    9.426    9.426 /usr/local/lib/python2.7/site-packages/urllib3/connectionpool.py:831(_validate_conn)
     10/1    0.002    0.000    9.426    9.426 /usr/local/lib/python2.7/site-packages/urllib3/connection.py:299(connect)
     10/1    0.000    0.000    9.415    9.415 /usr/local/lib/python2.7/site-packages/urllib3/connection.py:145(_new_conn)
2019-03-18 12:19:47,712 - python27_2 - total_time=0:00:10.495121
2019-03-18 12:19:47,713 - python27_2 - END
+ set +x
{% endhighlight %}

There following entries stand out:

{% highlight text %}
... 10.008    0.385 /usr/local/lib/python2.7/site-packages/urllib3/util/wait.py:139(wait_for_read)
... 10.007    0.385 /usr/local/lib/python2.7/site-packages/urllib3/util/wait.py:87(poll_wait_for_socket)
... 10.002    0.385 /usr/local/lib/python2.7/site-packages/urllib3/util/wait.py:99(do_poll)
... 9.991    0.999 /usr/local/lib/python2.7/site-packages/urllib3/contrib/pyopenssl.py:271(recv)
{% endhighlight %}

The socket when it is blocked on ```recv``` is stuck on ```wait_for_read``` and is doing a
```do_poll``` to wait it out.

If we look at the source of 
{% highlight text %}
/usr/local/lib/python2.7/site-packages/urllib3/util/wait.py:99(do_poll)
{% endhighlight %}


{% highlight text %}
def do_poll(t):
    if t is not None:
        t *= 1000
    return poll_obj.poll(t)

return bool(_retry_on_intr(do_poll, timeout))
{% endhighlight %}

We see that there is an explicit wait of 1 second (1000 ms) happening on when the
socket is blocked. One more interesting observation is that the ```recv``` socket call is made
from ```pyopenssl.py```

{% highlight text %}
... 9.991    0.999 /usr/local/lib/python2.7/site-packages/urllib3/contrib/pyopenssl.py:271(recv)
{% endhighlight %}

We have proof that code in [Stage-2 Python 2.7](#stage-2-python27) runs slow. But why did 
the previous case run fast and what does the presence of ```pyopenssl``` have to do with it?

To understand this, let us profile [Stage-1 Python 2.7](#stage-1-python27):

{% highlight text %}
# ./docker-exec.sh test_grequests_python27_1 \
  '/usr/local/bin/python /app/test_grequests_v2.py \
  --url https://https_server:8081/delay/1 --url-count 10 \
  --profile-code --profile-stats-count 20'
{% endhighlight %}


{% highlight text %}
================================
--------------------------------
urllib3==1.24.1
requests==2.21.0
gevent==1.4.0
grequests==0.3.0
--------------------------------
+ docker exec -it test_grequests_python27_1 /usr/local/bin/python /app/test_grequests_v2.py --url https://https_server:8081/delay/1 --url-count 10 --profile-code --profile-stats-count 20
2019-03-18 12:35:48,228 - python27_1 - START
2019-03-18 12:35:49,595 - python27_1 - len(all_page_ids) = 10
2019-03-18 12:35:49,596 - python27_1 - len(valid_page_ids) = 10
2019-03-18 12:35:49,596 - python27_1 - len(invalid_page_ids) = 0
         19256 function calls (18825 primitive calls) in 1.364 seconds

   Ordered by: cumulative time
   List reduced from 474 to 20 due to restriction <20>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    1.364    1.364 /usr/local/lib/python2.7/site-packages/grequests.py:103(map)
     10/1    0.000    0.000    1.364    1.364 /usr/local/lib/python2.7/site-packages/grequests.py:60(send)
     10/1    0.000    0.000    1.364    1.364 /usr/local/lib/python2.7/site-packages/requests/sessions.py:466(request)
     10/1    0.001    0.000    1.335    1.335 /usr/local/lib/python2.7/site-packages/requests/sessions.py:617(send)
     10/1    0.001    0.000    1.333    1.333 /usr/local/lib/python2.7/site-packages/requests/adapters.py:394(send)
     10/1    0.001    0.000    1.325    1.325 /usr/local/lib/python2.7/site-packages/urllib3/connectionpool.py:446(urlopen)
     10/1    0.001    0.000    1.322    1.322 /usr/local/lib/python2.7/site-packages/urllib3/connectionpool.py:319(_make_request)
     10/1    0.001    0.000    1.322    1.322 /usr/local/lib/python2.7/httplib.py:1084(getresponse)
     10/1    0.001    0.000    1.322    1.322 /usr/local/lib/python2.7/httplib.py:431(begin)
     70/7    0.006    0.000    1.320    0.189 /usr/local/lib/python2.7/socket.py:410(readline)
     10/1    0.002    0.000    1.319    1.319 /usr/local/lib/python2.7/httplib.py:392(_read_status)
     10/1    0.000    0.000    1.319    1.319 /usr/local/lib/python2.7/site-packages/gevent/_sslgte279.py:448(recv)
     10/1    0.986    0.099    1.299    1.299 /usr/local/lib/python2.7/site-packages/gevent/_sslgte279.py:298(read)
       10    0.001    0.000    0.097    0.010 /usr/local/lib/python2.7/site-packages/requests/sessions.py:426(prepare_request)
       10    0.001    0.000    0.043    0.004 /usr/local/lib/python2.7/site-packages/requests/models.py:307(prepare)
       10    0.000    0.000    0.042    0.004 /usr/local/lib/python2.7/site-packages/requests/adapters.py:292(get_connection)
       70    0.003    0.000    0.042    0.001 /usr/local/lib/python2.7/site-packages/requests/sessions.py:49(merge_setting)
     10/1    0.001    0.000    0.040    0.040 /usr/local/lib/python2.7/site-packages/gevent/_socketcommon.py:382(_resolve_addr)
     20/1    0.001    0.000    0.040    0.040 /usr/local/lib/python2.7/site-packages/gevent/_socketcommon.py:179(getaddrinfo)
       10    0.000    0.000    0.038    0.004 /usr/local/lib/python2.7/site-packages/urllib3/poolmanager.py:267(connection_from_url)



2019-03-18 12:35:49,607 - python27_1 - total_time=0:00:01.385698
2019-03-18 12:35:49,608 - python27_1 - END
+ set +x
================================
{% endhighlight %}

There are 2 important observations here:

1. There are no **wait** calls! 
2. The ```recv``` call is made from ```gevent``` and not ```pyopenssl```

{% highlight text %}
.... 1.319    1.319 /usr/local/lib/python2.7/site-packages/gevent/_sslgte279.py:448(recv)
{% endhighlight %}

The ```recv``` call is tied to the socket - The only way 2 different ```recv``` calls
are being made is if in both cases, even though the code is the same, something is causing
the socket class to change.





To be continued....
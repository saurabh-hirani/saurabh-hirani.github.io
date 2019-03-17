---
layout: post
title: The curios case of grequests and https
tags:
- python
---

This post describes as interesting issue I ran into while using Python's grequests 
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

### Scenario-1: Python 2.7 and Python 3.7 run fast

#### Scenario-1: Python 2.7

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

#### Scenario-1: Python 3.7

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

### Scenario-2: Python 2.7 and Python 3.7 run slow

#### Scenario-2: Python 2.7

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

There are 2 major differences from Scenario-1:

1. We installed the ```pyopenssl``` module.
2. The same code's run time spiked from around 1 sec to 10 seconds i.e. 1 second for each URL.



To be continued....
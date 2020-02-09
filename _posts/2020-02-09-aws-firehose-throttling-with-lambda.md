---
layout: post
title: AWS Kinesis Firehose throttling with transformation Lambda
tags:
- AWS
---

In my [previous post](http://saurabh-hirani.github.io/writing/2020/02/08/terraform-aws-firehose-elasticsearch) I wrote about a Terraform module which can be used to setup a
logging pipeline with AWS Kinesis Firehose and AWS Elasticsearch. Following is the architecture diagram of the setup:

<div class='pull-left' style="border: 0px solid black;">
{% include figure.html path="blog/aws-firehose-throttling-with-lambda/aws-firehose-elasticsearch-arch.png" alt="slide preview problem" url=url_with_ref %}
</div>

In this post, I want to show a method we used to throttle the flow between AWS Kinesis Firehose and AWS Elasticsearch using the transformation Lambda.

As per the above diagram the data flow is:

1. Sender AWS account has an AWS Lambda which send logs to the receiver AWS account.
2. Receiver AWS account trusts the sender AWS account and accepts logs at the AWS Kinesis Firehose.
3. AWS Kinesis Firehose validates the incoming records and does any data transformation through AWS Kinesis transformation Lambda.
4. AWS Kinesis Firehose backs up a copy of the incoming records to a backup AWS S3 bucket.
5. Valid records are delivered to AWS Elasticsearch.
6. Invalid records (invalid json, invalid base64 encoding) and Elasticsearch reject records (Elasticsearch mapping exceptions, etc.) are backed up to processing-failed and elasticsearch-failed buckets respectively.
7. The user can view valid logs on the AWS Elasticsearch Kibana UI.

Restating a central idea for the post as per point 3 - AWS Kinesis Firehose can use a Lambda to validate incoming records and do data transformation
on them. This Lambda is optional but we will see how it can serve a very important purpose.

Following are the points of entry of data which might need rate limting:

1. Between sender AWS account Lambda and receiver AWS account Kinesis Firehose.
2. Between receiver AWS account Kinesis Firehose and receiver AWS account Elasticsearch.

### Sender Lambda -> Receiver Firehose rate limting

Although AWS Kinesis Firehose does have [buffer size and buffer interval](https://aws.amazon.com/kinesis/data-firehose/faqs/), which help to batch
and send data to the next stage, it does not have explicit rate limiting for the incoming data. You can rate limit indirectly by working with AWS
support to tweak [these](https://docs.aws.amazon.com/firehose/latest/dev/limits.html) limits. There is no UI or config to change them, so account
for AWS support turnaround time and ensure that you set them to high numbers depending on your incoming data rate.

### Receiver Firehose -> Receiver Elasticsearch rate limiting

There is no support to do rate limit on this flow. You cannot say - ensure that AWS Kinesis Firehose throttles and sends data at max rate of X MB/sec to
the backend AWS Elasticsearch. Rate limting this flow is useful if you are sending data from multiple AWS accounts into one Firehose destined for different
indices in AWS Elasticsearch to ensure that one sender does not overwhelm the Firehose and subsequently Elasticsearch for other tenants.

This is where the transformation Lambda comes in handy. As it is used to validate/transform incoming data, setting the right level of concurrency can help
you do rate limiting. Let's understand some AWS Lambda basics before we tackle this problem.

### AWS Lambda concurrency basics

As per [this](https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html) post,

```sh
Concurrency is the number of requests that your function is
serving at any given time.
```

this means that a concurrency of **1** means that at any given time only one instance of Lambda can run.

If Lambda has 200 ms runtime, in 1 second - you can run this Lambda - 5 times (1 / 0.2 sec) with a concurrency of 1 - because we cannot run more than 1 instance
of the Lambda at any given time.

| Concurrency  | Invocations per sec  | Invocations per min  |
| :----------: | :------------------: | :------------------: |
|      1       | 5 * 1 = 5            | 5 * 1 * 60 = 300     |
|      2       | 5 * 2 = 10           | 5 * 2 * 60 = 600     |
|      3       | 5 * 3 = 15           | 5 * 3 * 60 = 900     |
|      n       | 5 * n = 5n           | 5 * n * 60 = 300n    |


<br/>
Attempting to invoke at a higher rate than the concurrency leads to [throttling](https://www.bluematador.com/docs/troubleshooting/aws-lambda-throttles).

If the AWS Lambda processes 2 KB in each invocation and it takes 200 ms for each run then:

| Concurrency  | KB processed/sec       | KB processed/min       |
| :----------: | --------------------:  | :--------------------: |
|      1       | 5 * 1 * 2 = 10         | 5 * 1 * 2 * 60 = 600   |
|      2       | 5 * 2 * 2 = 20         | 5 * 2 * 2 * 60 = 1200  |
|      3       | 5 * 3 * 2 = 30         | 5 * 3 * 2 * 60 = 1800  |
|      n       | 5 * n * 2 = 10n        | 5 * n * 2 * 60 = 600n  |

and so on.

### Derive concurrency given processing rate

Given this knowledge, let us assume that we want to ensure that Elasticsearch does not receive > 1.5 TB / day

1.5 TB / day
<br/>
= 1649267441664 bytes / day
<br/>
= 1649267441664 bytes / (24 * 60) min
<br/>
= 1092.26 MB / min
<br/>
=~ 1 GB / min for easier calculation.

Assuming that we want to process 1 GB / minute => what should be the Lambda concurrency?

For this we will have to assume the Lambda run time and Lambda payload per call.

As explained above, AWS Kineis Firehose has [buffer size and buffer interval](https://aws.amazon.com/kinesis/data-firehose/faqs/) which mean that,
AWS Kinesis Firehose will buffer the data for X MBs or for X minutes before sending it ahead for processing. Let us stick with buffer size
for now. Let us assume buffer size = 1 MB. Please note that AWS Lambda also has [limits](https://docs.aws.amazon.com/lambda/latest/dg/limits.html) i.e.
as of this writing, it can have max invocation payload of 6 MB.

Lambda payload per call = 1 MB.

Let us assume runtime of 200 ms.

Invocations per sec = 1 / (0.2 sec) = 5.

Data processed per call = 1 MB.

Date processed per sec = 5 MB.

Data processed per min = 300 MB.

We want data processed per min = 1024 MB.

Concurrency = 1 => 300 MB per min

Concurrency = 2 => 600 MB per min

Concurrency = 3 => 900 MB per min

Concurrency = 4 => 1200 MB per min

As we cannot set concurrency in floating point - we should use concurrency as **4** which will give us 1200 MB per min => 1.64 TB per day - which is close
to our goal. We can also go lower to concurrency of **3** to get 900 MB per min => 1.23 TB per day.

The above calculation can easily be converted to a script, which I have added [here](https://github.com/saurabh-hirani/bin/blob/master/aws-lambda-calc-concurrency).

It can be invoked as

```sh
# ./aws-lambda-calc-concurrency avg_duration_sec avg_payload_bytes limit_bytes_per_mind
./aws-lambda-calc-concurrency 0.2 1048576 1073741824
3.41
```

where

Lambda runtime = 0.2 sec = 200 ms

Lambda payload size = 1048576 = 1 MB

Required data processing amount per min = 1073741824 = 1 GB

Instead of remembering the MB to bytes and GB to bytes conversion, you could also call the above command as

```sh
./aws-lambda-calc-concurrency 0.2 $(echo 1 | MB-to-bytes) $(echo 1 | GB-to-bytes)
```

where ```MB-to-bytes``` and ```GB-to-bytes``` are simple MB and GB to byes converters present in the same [bin](https://github.com/saurabh-hirani/bin) repo.

### Caveat

The above calculation assumes uniform distribution of data and it can afford to do so because the transformation Lambda is fed by the Kinesis Firehose
which needs to have a fixed buffer rate. But limiting the concurrency does not tell the AWS Firehose to invoke the Lambda X times. That is precisely what
we want to protect against.

What happens when you get a sudden spike which makes Firehose invoke more Lambdas than the concurrency? => concurreny limit breached => Throttling => Add headroom in concurrency numbers. Because Firehose does not know/care about Lambda concurrency.

### Derive concurrency given processing rate

After understanding the above calculation, we can check a running Lambda, look at its concurrency, avg runtime, avg payload size and check the
amount of data it can process.

```sh
./aws-lambda-calc-bytes 4 0.2 $(echo 1 | MB-to-bytes)
1258291200.00

echo 1258291200.00 | bytes-to-GB
1.17
```

which means that with

Lambda concurrency = 4

Lambda runtime = 0.2 sec = 200 ms

Payload size = 1 MB

you can process 1.17 GB per minute.

That's about it. Understand these limits. Don't use unlimited concurrency to ensure the right amount of throttling.

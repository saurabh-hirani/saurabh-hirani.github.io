---
layout: post
title: Pipelines >> Custom scripts - Part-1
tags:
- Linux
- cli
---

Problem-1:

1. Find out all EC2 instances which are in a public subnet.

Few possible solutions in increasing order of developer productivity:

1. Go to the AWS console, find public subnet ids, and filter AWS EC2 instances by those ids.

2. Write a Python program using [boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)

    ```python
    import boto3

    ec2 = boto3.resource("ec2")

    for instance in ec2.instances.all():
        public_ip = instance.public_ip_address
        if public_ip:
            print(instance.id, public_ip, "public")
    ```

3. Run a [jq](https://github.com/stedolan/jq) command

    ```sh
    aws ec2 describe-instances | jq -rc '.Reservations[].Instances[] |
                                         [.InstanceId, (.PublicIpAddress | if .==null then "Private" else "Public" end)] | @csv'
    ```

- Observations:
  - Searching on the UI is clunky and prone to human error.
  - Writing a python program is better than using the AWS console, but is mostly an overkill for a problem this trivial.
  - jq cli seems to fit in the sweet spot of just enough effort to get the desired output.

One might say that in this case the Python program is simpler than the jq cli. However, getting
used to jq and its myriad filters is useful for getting quick answers. These kind of problems serve
as opportunities to build that muscle memory.

This post is about the 4th approach - take the dataset which answers your current query and see if it hides some other queries worth answering.
If it does, solve for making the dataset queryable rather than writing one-off solutions.

Problem-2:

1. Find out all EC2 instances which were launched before Jan 1st 2021.

An interesting problem because, knowing this would help us clean up older instances. Armed with `boto3` and `jq`, it is only natural
to jump at the problem and solve it using these tools. However, before we solve it, let us think of what are the extra things we will need
to learn:

1. Find which key decides the launch time of EC2 instance in `aws ec2 describe-instances` call. Answer: `LaunchTime`

2. Once we find the date, google for the millionth time on how to compare dates OR if you want to be clever, split the string - "2017-02-16T09:11:35.000Z"
   on **T** and `sed` delete the hyphen and do a numerical comparison with 20210101

Learning-2 is not worth relearning all the times and yet, Google runs on search queries of cute kittens, how to exit vim and how to compare dates.

This is where [q](http://harelba.github.io/q/) comes in - all guns blazing.

A solution, which decouples data extraction and data analysis:

  ```sh
  ./aws-ec2-describe-instances-csv | \
  q -O -H -d',' 'select launch_time from - where strftime("%Y%m%d", launch_time) < "20210101"'
  ```

  where [./aws-ec2-describe-instances-csv](https://github.com/saurabh-hirani/bin/blob/master/aws-ec2-describe-instances-csv) will keep on growing with
  comma separated fields as I keep on encountering more SQL-like problem scenarios.

But you did have to learn about **strftime** - didn't you? - this question might be posed by the perceptive reader. Yes - but the point is that instead of
employing logic of what to do with the data in the tool that queries the data, by dumping the output in CSV, we have made the data queryable in a lot
of ways. The same CSV dump can answer queries like:

1. Give me all the instances whose tags match 'kuber%'

    ```sh
    ./aws-ec2-describe-instances-csv | \
    q -O -H -d',' 'select * from - where tags like "kuber%"'
    ```

2. Give me all the instances in the public subnet and in ap-south\* AZ

    ```sh
    ./aws-ec2-describe-instances-csv | \
      q -O -H -d',' 'select * from - where public_or_private == "Public" and az like "ap-south%"'
    ```

and so on. The important thing is that now the cognitive translation (I just made that up) between the query you want to ask and the code you **have** to write to
answer it has reduced.

But the output is an ugly mess of a CSV!!

This is where I shamelessly plug in a wrapper I wrote over the amazing [PrettyTable](https://pypi.org/project/prettytable/) module which made my terminal CSV cleaner in a lot of ways - [csv2table](https://github.com/saurabh-hirani/bin/blob/master/csv2table). This has grown quite a bit since its [last appearance](http://saurabh-hirani.github.io/writing/2021/04/23/what-happens-on-cli)

  ```sh
  ./aws-ec2-describe-instances-csv | \
   q -O -H -d',' 'select * from - where public_or_private == "Public" and az like "ap-south%"'
  ```

  [output-1](https://gist.github.com/saurabh-hirani/6e3120d5ebd6fe719a798398971f44d9)

  ```sh
  ./aws-ec2-describe-instances-csv | \
   q -O -H -d',' 'select * from - where public_or_private == "Public" and az like "ap-south%"' |\
   TABLE_HAS_HEADER=1 TABLE_ORIENT=v csv2table
  ```

  [output-2](https://gist.github.com/saurabh-hirani/f2e787f44e4d6847a59b09a2a0c02379)

  Cleaner and more readable. Flip the `TABLE_ORIENT=h` to view the output in horizontal format. You could also do a selective select instead of '\*' to keep the tags from cluttering the output.

- Learnings:

1. `aws-ec2-describe-instances-csv` dumps the AWS output in CSV. Does not take the responsibility of processing.

2. `q` allows the user to write SQL like familiar syntax to extract data. Does not take the responsibility of displaying the data nicely.

3. `csv2table` takes CSV and displays in a clean terminal tabular format.


Combining these 3 tools with the ever-powerful Linux pipeline can make for some interesting patterns for extracting data.

In closing, as a shout out to the giants whose shoulders we stand upon, this post demonstrates a simple example of the following two principles from the [Unix philosophy](https://en.wikipedia.org/wiki/Unix_philosophy):

1. Make each program do one thing well.
2. Expect the output of every program to become the input to another, as yet unknown, program.

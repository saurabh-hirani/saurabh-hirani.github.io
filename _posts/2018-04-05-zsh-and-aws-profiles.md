---
layout: post
title: Switching between AWS profiles using zsh completion
tags:
- aws
- zsh
---

This blog post shows you a simple way to switch between different AWS profiles
on the command line.

**Pre-requisites:**

1. [aws cli](https://aws.amazon.com/cli/)
2. You are using [zsh](http://ohmyz.sh/) as your shell.

**Scenario:**

If you have worked with multiple AWS accounts and use the ```~/.aws/credentials```
file to store your profiles, you will have a file full of the following statements:

<script src="https://gist.github.com/saurabh-hirani/16a63ff1a9163547ffc66c8665d33b67.js"></script>

To set an AWS profile on the command line via the environment variable, you do
the following:

{% highlight text %}
# look up the right profile by grepping it or
# doing a cat on ~/.aws/credentials

$ export AWS_PROFILE=profile_1

# do some work and when the time comes to switch
# your profile - do the same activity again

$ export AWS_PROFILE=profile_2
{% endhighlight %}

or if you want to set the access key and secret key

{% highlight text %}
# look up the right profile by grepping it or
# doing a cat on ~/.aws/credentials

$ export AWS_ACCESS_KEY_ID=profile_1_access_key
$ export AWS_SECRET_ACCESS_KEY=profile_1_secret_key

{% endhighlight %}

**Problem:**

The problem with the above approach is:

1. You have to manually look up the ```~/.aws/credentials``` file and copy paste
the right values.

2. When the no. of AWS accounts increase, the profiles associated with them increase
and you have to grep for the profile name in the file.

**Requirement:**

You should have a simple way to set the AWS_PROFILE on the command line with some
sort of autocompletion. Without doing too much work.

**Solution:**

zsh has a very cool feature called [zsh completions](http://mads-hartmann.com/2017/08/06/writing-zsh-completion-scripts.html)
which can help us in this case.

Carry out the following steps to enable zsh commands which help you list and set
your AWS profiles and keys with autocompletion:


1. Create the completions directory:
   ```
   # mkdir -p ~/.oh-my-zsh/completions
   ```
2. Change dir to the completions directory:
   ```
   # cd ~/.oh-my-zsh/completions </pre>
   ```
3. Create ```~/.oh-my-zsh/completions/_set-aws-profile``` file (**without** the .sh extension):
   <script src="https://gist.github.com/saurabh-hirani/8dd62d4521070ed4166734d4fb3678db.js"></script>

4. Create ```~/.oh-my-zsh/completions/_set-aws-keys``` file (**without** the .sh extension):
   <script src="https://gist.github.com/saurabh-hirani/4a6a1559ffb7ab890a34977887cd8c20.js"></script>

5. Add the supporting commands to your ```~/.zshrc``` file:
   <script src="https://gist.github.com/saurabh-hirani/61d02a3d836f1321dae6d359fc492de1.js"></script>

6. Open a new terminal and source your new config:
   ```
   # source ~/.zshrc
   ```
7. Give it a spin:

{% include figure.html path="blog/zsh-aws-autocomplete/autocomplete-demo.gif" alt="zsh aws cmdline autocomplete gif" url=url_with_ref %}

---
layout: post
title: Remote Profiling Elixir Over SSH
tags: [elixir, devops, ssh, erlang]
image:
  background: triangular.png
---

Erlang comes with an awesome tool called observer, which is readily available to the Elixir user.

If you want to use it, simple run `:observer.start` from an IEx session.

An interesting feature is that you can then connect to another node and observer its details.  Getting that working when the other node is on a different machine accessible only via SSH is another story.  This is that story!

## A brief trip into the world of SSH

SSH is something many developers use on a daily basis.  The way we _normally_ use it is to provide us with a secure connection to a remote server, allowing us to run commands as if we were in the same room.

Another thing we can to is to forward local ports to remote machines.  Let me explain with an example.

{% highlight bash linenos %}
ssh -N -L 9001:localhost:9001 root@192.168.1.12
{% endhighlight %}

If we now do anything that involves connection to port 9001, it will be sent to the remote machine.  This gets kind of cool and allows you to use local instance of utilities like `psql`, but connected to a remote machine!  This means your interactions are as fast as local, you don't get that SSH typing delay that (for me at least) causes me to make lots of typos because I'm second guessing myself!

The -N flag means we don't want to interact directly with the session, so it will just sit there and wait.

This port forwarding becomes important because, although Erlang / Elixir can happily communicate with other instances over a local network, it is not happy to do so over SSH.

Let's fix that.

## Starting IEx with some restrictions

By default the Erlang VM will use a bunch of different ports for distributed communication.  We're going to force its hand, enabling us to guarantee which ports we should forward.

On your remote machine, run

{% highlight bash linenos %}
$ epmd -names
epmd: up and running on port 4369 with data:
{% endhighlight %}

You should see something similar to the output above.

EPMD is the Erlang Port Mapper Daemon.  It is kind of like a DNS server for Erlang nodes.

On our remote machine we can start IEx with a specific port on which to listen, rather than a _range_ of ports.

In this example I wanted to profile a Phoenix application.

{% highlight bash linenos %}
iex --name bench@127.0.0.1 --cookie 123 --erl "-kernel inet_dist_listen_min 9001 inet_dist_listen_max 9001" -S mix phoenix.server
{% endhighlight %}

Let's break down what's been done here.

`--name` gives our node a name, in this instance I'm explicitly stating it's on localhost.  The alternative `--sname` didn't seem to play nice, but that may be operator error!

`--cookie` Is where we set a shared secret that gives nodes permission to talk to each other.  In this instance I didn't care about making it a difficult cookie as I'm on a local network with non-production systems.

`--erl` Allows us to pass options to the Erlang VM itself.

`-kernel inet_dist_listen_min 9001 inet_dist_listen_max 9001` Is essentially saying, to the VM, you may only listen on port 9001.

## Setting up as SSH tunnel

We now know that ports 4369 and 9001 need to be linked.

{% highlight bash linenos %}
ssh -N -L 9001:localhost:9001 -L 4369:localhost:4369 root@192.168.1.12
{% endhighlight %}

Here we're linking two ports at the same time to the remote machine.

You may be prompted for a password at this point, depending on whether you're set up key authentication or not.

## Seeing all the nodes

In another local terminal session, run `epmd -names`, you should see something like this.

{% highlight bash linenos %}
epmd: up and running on port 4369 with data:
name bench at port 9001
{% endhighlight %}

The fact that it shows `name bench at port 9001` tells us that we now have the ability to connect to that node.

## Connecting the Erlang observer

We'll now fire up and Erlang Shell running the observer.

{% highlight bash linenos %}
erl -name debug@127.0.0.1 -setcookie 123 -run observer
{% endhighlight %}

Note that the syntax is slightly different, but we give it a name different to the remote node, but the *same* cookie.  This will now start the Erlang observer.

![Erlang observer]({{ site.url }}/images/observer.jpg)

Once this is running, click on the Nodes option on the menu bar and select `Connect node`, enter the name we used earlier `bench@127.0.0.1`.  

![Select Node]({{ site.url }}/images/connect_node.jpg)

If all is well, you will now see your remote application's profiling data.  Note that in this example, I've connect to a node running on a Raspberry PI, so we've got 4 cores on an ARM processor.

![Remote Node]({{ site.url }}/images/remote_node.jpg)

I'll just throw a bit of load at it ... and

![Node Under Load]({{ site.url }}/images/node_under_load.jpg)


## Wrapping up

Being able to communicate with a running node, remotely, over SSH gives us the ability to do lots of interesting experiments.  Load testing, for example.  The really nice thing about this is that it's much closer to real world usage than running benchmarks locally.

In an upcoming post I'll share what I've learned from running Phoenix on a Raspberry Pi.

# Shameless plug

I'm a consultant application developer with [ThoughtWorks](http://www.thoughtworks.com) and if you would like us to work with you and your company, reach out at <mailto:mfeckie@thoughtworks.com>

In the meantime, enjoy profiling all of the things :)


---
layout: wide-post
title: Phoenix in Production with systemd
tag: [elixir production phoenixframework phoenix systemd]
image:
background: triangular.png
---
<p>
  In this post, I'm going to explore how we can leverage <strong>systemd</strong> to ensure our apps are kept running even following a power outage or whole system crash.
</p>
<h1>Running Elixir in production</h1>
<p>
  For those not familiar with running an Elixir application there are two main ways that people use in the community, one is to use a straightforward <code>PORT=4001 MIX_ENV=prod mix phoenix.server</code>.  This approach is very simple and straightforward, but has some requirements that may not suit everyone, the server, for example must have Erlang and Elixir installed.
</p>
<p>
  The other major method is to leverage Erlang releases, this what we'll be doing here.
</p>
<h1>Why do we care about releases?</h1>
<p>
  Much of the world has gone a little crazy about Docker, using self contained, immutable containers for packaging their apps.  Interestingly, Erlang has long supported releases, which does a significant subset of what folks are trying to achieve with Docker.  You can think of a release in the same category as a self contained binary - it brings with it everything it needs to run.
</p>
<p>
  Generally a release contains the Erlang runtime, though like most things, it's configurable.  This is important because it can be troublesome if you build a release on a different platform to where it will run.  If I build a release on OSX, it won't run on Ubuntu.  Some folks get frustrated by this, but I think it's pretty reasonable.

  A release is essentially a collection of parts built into your application.  If your factory built a car with a petrol engine, we shouldn't be surprised if it doesn't run on diesel!
</p>
<h1>The first great struggle</h1>
<p>
  Many of us have embraced <a href="https://12factor.net/">12 Factor</a> as a way of ensuring our applications are readily configurable and scalable.  One aspect of this is using environment variables to configure aspects of our applications at runtime.
</p>
<p>
  It often comes as a surprise to folks new to the deploying Elixir applications, that their environment variables are not respected when they come to run a release.  The issue is that config files are evaluated at <strong>compile</strong> time.  I'm not going to rehash the good information available at plataformatec, so <a href="http://blog.plataformatec.com.br/2016/05/how-to-config-environment-variables-with-elixir-and-exrm">read for yourself.</a>
</p>
<h1>Environment variables and edeliver</h1>
<p>
  I've been relying very heavily on <a href="https://github.com/boldpoker/edeliver">edeliver</a> for our deploys.  Edeliver leans heavily on relx and provides the deployment part of releases.  It's a great and very straightforward tool.
 </p>
 <p>
   In the process of getting our application deployable I found a big challenge making environment variables available to the system.
 </p>
<p>
  The challenge comes because environment variables you may have exported in, say, <code>.bashrc</code>, aren't available when edeliver executes commands for you. One solution to this is to add your environment variables to <code>.profile</code> , which will make them available when edeliver executes commands.  For more information have a look at the <a href="https://github.com/boldpoker/edeliver#configuration">configuration</a> section of the edeliver documentation.
</p>
<h1>The trouble with upstart</h1>
<p>
  Trying to deal with jobs that require environment variables is a non trivial problem with upstart and there's not a clear and agreed convention to resolve the matter.  There's a ton of advice, hacks and strategies to get it working, but none of them made me feel warm and fuzzy.
</p>
<p>
  As it stands currently, the Phoenix documentation only discusses upstart.  They do share one way of dealing with environment variables.
  Below is the example provided.
</p>
<div class="highlight">
  <pre>description <span class="s2">&quot;hello_phoenix&quot;</span>

  <span class="c">## Uncomment the following two lines to run the</span>
  <span class="c">## application as www-data:www-data</span>
  <span class="c">#setuid www-data</span>
  <span class="c">#setgid www-data</span>

start on runlevel <span class="o">[</span>2345<span class="o">]</span>
stop on runlevel <span class="o">[</span>016<span class="o">]</span>

expect stop
respawn

env <span class="nv">MIX_ENV</span><span class="o">=</span>prod
<span class="nb">export </span>MIX_ENV

  <span class="c">## Uncomment the following two lines if we configured</span>
  <span class="c">## our port with an environment variable.</span>
  <span class="c">#env PORT=8888</span>
  <span class="c">#export PORT</span>

  <span class="c">## Add app HOME directory.</span>
env <span class="nv">HOME</span><span class="o">=</span>/app
<span class="nb">export </span>HOME

pre-start <span class="nb">exec</span> /bin/sh /app/bin/hello_phoenix start

post-stop <span class="nb">exec</span> /bin/sh /app/bin/hello_phoenix stop
</pre></div>
<p>
  As you can see, it's possible to provide environment variables, but often we'd want this configuration to be part of an Ansible script or some other version controlled system configuration tool.  This makes handling of sensitive data, like passwords, troublesome.
</p>
<h1>A real world problem</h1>
<p>
  I recently dealt with an odd and intermittent fault with one application I work on.  The problem presented with the Postgrex component of our application being unable to find our database hosted on AWS.  The issue was difficult to reproduce and was always resolved by simply stopping the release and starting it again.  No other intervention necessary, no server reboots, nothing.
</p>
<p>
  The error showing was <code>Postgrex.Protocol (#PID<0.1511.0>) failed to connect: ** (DBConnection.ConnectionError) tcp connect: non-existing domain - :nxdomain</code>.  This error seemed like something Erlang / Elixir should be able to recover from, but alas it did not.
</p>
<p>
  On trawling the system logs we discovered that the problem only presented itself on reboot, which was a great clue.  It seemed like a kind of race condition sometimes exists where the Elixir application is started before all of the networking aspects the OS have been initialized.  This caused problems because our databse is not hosted on the same machine and is located via AWS DNS.
</p>
<h1>Systemd to the rescue</h1>
<p>
  One of the nice things about systemd is you can specify the order in which things should happen in a declarative manner.
</p>
<p>
  This file is placed at <code>/etc/systemd/system/my_phoenix_app.service</code>
</p>
<div class="highlight">
<pre><span class="k">[Unit]</span>
<span class="na">Description</span><span class="o">=</span><span class="s">Runner for My Phoenix App</span>
<span class="na">After</span><span class="o">=</span><span class="s">network.target</span>

<span class="k">[Service]</span>
<span class="na">WorkingDirectory</span><span class="o">=</span><span class="s">/opt/path_to_my_phoenix_app</span>
<span class="na">EnvironmentFile</span><span class="o">=</span><span class="s">/etc/default/my_phoenix_app.env</span>
<span class="na">ExecStart</span><span class="o">=</span><span class="s">/opt/path_to_my_phoenix_app/bin/my_phoenix_app start</span>
<span class="na">ExecStop</span><span class="o">=</span><span class="s">/opt/path_to_my_phoenix_app/bin/my_phoenix_app stop</span>
<span class="na">User</span><span class="o">=</span><span class="s">ubuntu</span>
<span class="na">RemainAfterExit</span><span class="o">=</span><span class="s">yes</span>

<span class="k">[Install]</span>
<span class="na">WantedBy</span><span class="o">=</span><span class="s">multi-user.target</span>
</pre>
</div>

<p>
  Let just call out a few interesting things here.
  <ol>
    <li>In our <code>[Unit]</code> definition we can pass the <code>After</code> directive, ensuring our network is ready to go before we try to run</li>
    <li>We use the <code>EnvironmentFile</code> directive to setup all our environment variables for the service</li>
    <li>We can specify which <code>User</code> the code is executed with</li>
    <li>We use the <code>RemainAfterExit</code>, which means we don't have to run the applications using <code>foreground</code> rather than <code>start</code></li>
  </ol>
</p>
<p>
  The particularly nice thing, in my opinion, is the ability to put our environment variables into a file.  This gives the ability to securely store credentials on the server without checking them into version control.  Alternatively, it provides a nice single source of truth for committing to whatever credential management system / tool / package you're using.
</p>
<p>
  In order to recognize the service we must execute <code>systemctl daemon-reload</code> after creating the file.
</p>
<p>
  Once this has been done we can do a one off start <code>systemctl start my_phoenix_app.service</code>, or start it permanently with <code>systemctl enable my_phoenix_app.service</code>
</p>

<p>
  Here's an example of a .env file that might be used by a phoenix / ecto application.
  Take particular note that we set <code>RELX_REPLACE_OS_VARS=true</code>, without this our release would not try to evaluate the environment variables.
  <em>Update:</em> Thanks to David Kuhta for pointing out that if you're using Distillery, rather than exrm, use <code>REPLACE_OS_VARS=true</code> instead
</p>
<p>
  This file lives at <code>/etc/default/my_phoenix_app.env</code> and is reference from the service definition.
</p>

<div class="highlight"><pre><span class="na">HOME</span><span class="o">=</span><span class="s">/path_to_release</span>

<span class="na">SECRET_KEY_BASE</span><span class="o">=</span>
<span class="na">DB_PASSWORD</span><span class="o">=</span>
<span class="na">SENDGRID_API_KEY</span><span class="o">=</span>
<span class="na">DB_NAME</span><span class="o">=</span>
<span class="na">DB_USER</span><span class="o">=</span>
<span class="na">DB_HOST</span><span class="o">=</span>
<span class="na">DB_PASSWORD</span><span class="o">=</span>
<span class="na">HONEYBADGER_API_KEY</span><span class="o">=</span>
<span class="na">RELX_REPLACE_OS_VARS</span><span class="o">=</span><span class="s">true</span>
</pre></div>

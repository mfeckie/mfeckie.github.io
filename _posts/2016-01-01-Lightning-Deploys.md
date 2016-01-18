---
layout: post
title: Lightning Deploys with Phoenix and Ember
tags: [ember, elixir, phoenix, devops]
image:
  background: triangular.png
---

I have been really interested in the discussions in the Rails and Ember communities on the topic of Lightning Deploys.  Here are some things I've tried, failed, learned and succeeded with!

# What's a lightning deploy anyway?

One of the issues that many folks find when they start pushing their Rails applications into a more API / Client architecture is that deploying front-end assets can become painful.

Any Rails app of a decent size takes a non-trivial amount of time to boot, so there's a risk of downtime.  If you don't want downtime, then folks often look to do things like warming up the latest version then swapping over a load balancer to point at the new one.

An alternative strategy that has been talked about is the Lightning Deploy

<iframe width="560" height="315" src="https://www.youtube.com/embed/QZVYP3cPcWQ" frameborder="0" allowfullscreen></iframe>

Now in the Rails world, this is a great improvement and there's some tooling been build for [Ember-CLI](https://www.npmjs.com/package/ember-lightning) to support it.

So I set off on a path of implementing something similar in Phoenix.

# Standing on the shoulders of others 

To start with I read a bunch of blog posts on the topic and looked at various libraries, small selection here.

<http://blog.abuiles.com/blog/2014/07/08/lightning-fast-deployments-with-rails/>

<http://neo.com/2014/06/04/framework-agnostic-zero-downtime-javascript-app-deployment/>

<https://github.com/ember-cli/ember-cli-deploy/>

<https://github.com/ember-cli-deploy/ember-cli-deploy-lightning-pack>

<https://github.com/seanpdoyle/ember-cli-rails-deploy-redis>

They all look interesting, and I the set about trying to implement some kind of asset caching / deploying solution using Phoenix and Redis.  This was very much influenced by the stuff I read and saw.

# The first cut is the deepest

As I got close to finishing a solution using a generated manifest file and a way to connect Phoenix to a Redis instance, I had this moment where I'm like, why am I doing this?  What's the difference between an Erlang ETS table or even Mnesia?  Do I really need to install another dependency just to serve some HTML, JS and CSS?

This lead to the widespread deletion of code and another exploration!

# Leaning on Erlang

So I went off on another exploratory journey into the world of ETS.  It was very clear that it would do everything a Redis instance would, but without any additional dependencies.  Now I know there are those who may say, Redis is faster, has x,y,z feature, but for the purposes I was looking for, ETS is plenty fast enough and not needed to lean on third party libraries and the additional latency of making requests to Redis, I was OK with the choice.

So I begun the implementation and wrote a bunch of code.

# If you don't need it, don't implement it

After a while coursing down this path, I'm like, "Hmm, this feels an awful lot like a static file server ...."

Ember-CLI already spits out assets that are compressed, minified and fingerprinted, why not just serve them?

# It's turtles all the way down

It's not really turtles, it's Plug.  Plug has a module called `Plug.Static`, as you may have guessed, it serves static files!

So here's the the method I ended up with.

Ember spits out compiled and fingerprinted assets, in that, the `index.html` controls which assets should be served via their fingerprint.  All that needs to be done is to copy across the production built files.

In order to get this working in Phoenix we need to ensure that the index is served and that users are able to access the assets.

Here's the implementation.

## Configure Ember to use specific URL prefixes for assets

In order to provide a separate place for the app assets to be served I prefix them to be in a `lightning` folder.

Update your `ember-cli-build.js` file to include the fingerprinting configuration.

I personally prefer to service images from the static folder in Phoenix as for my use, they don't need a fingerprint and change almost never.  Drop the exclude line if that doesn't suit you.

{% highlight js linenos %}
module.exports = function(defaults) {
  const app = new EmberApp(defaults, {
   fingerprint: {
     exclude: ['images'],
     prepend: '/lightning/'
   }
  });
  return app.toTree();
};
{% endhighlight %}

## Configure Phoenix with some catchall routes

My choice for the application is to serve the `index.html` from an `/app` endpoint and the assets from somewhere else.  This may seem a bit stupid, but, it means that we can use Ember's pretty URL's without a `#` because anything after `app` is ignored by Phoenix in this approach.

All of the assets are service with under the `/lightning` scope to prevent any clashes with the namespace.

On the server itself, the lightning assets are located in an isolated folder with no connection to the rest of the application.  This reduces the risk of malicious directory traversal.  The whole application is served under a deployment user with very limited permissions and no root access.

I like the isolation because I can just keep deploying to that folder and the server never needs to restart.  Any user with a cached index.html will still be able to get the correct assets and they shouldn't experience any inconvenience.  If the backend API changes, that may not be the case.

{% highlight elixir linenos %}
scope "/lightning", MyApp do
  get "/*catch_all", LightningController, :assets
end

scope "/app", MyApp do
  get "/", AppController, :index
  get "/*catch_all", AppController, :index
end
{% endhighlight %}

The Controller is set up to serve only the index.html file and will ignore any parameters which are sent.  It's a very simple Plug.

{% highlight elixir linenos %}
defmodule Accreditus.AppController do
  use Plug.Builder
  import Plug.Conn
  plug :index

  def index(conn, _params) do
    index = Path.expand("~/deploys/lightning") <> "/index.html"
    conn
    |> put_resp_header("Content-Type", "text/html")
    |> send_file(200, index)
  end
end
{% endhighlight %}

In the LightningController we serve any file present in the lightning folder.  As part of my Ember build, I gzip all of the things, so instruct Plug to serve the zipped version if it exists.

As a polite server, we tell the user if the file was not found.

{% highlight elixir linenos %}
defmodule Accreditus.LightningController do
  use Plug.Builder
  plug Plug.Static, at: "/lightning/assets", from: Path.expand("~/deploys/lightning/assets"), gzip: true
  plug :not_found

  def not_found(conn, _) do
    send_resp(conn, 404, "Not found")
  end
end
{% endhighlight %}


# But does it scale?

I'm hosting most of my side projects on a $5 Digital Ocean instance and don't have a ton of users.  I think many folks that comment on more sophisticated systems of CDN's and the like are solving different problems to me!  I know those tools exist and at the point where I *need* them, I'll explore content distribution, but for now, not being able to handle load would be a nice problem to have!

As an aside, if you want to get $10 credit, sign up with Digital Ocean using my referral link <https://m.do.co/c/b7034c26858f>.

In an upcoming blog post I'll share the results of running my application on a Raspberry Pi 2 and hitting it hard!  Let's see how that scales :)

# Wrapping up

Although I wrote a bunch of code that I eventually threw away, I learned a ton along the way.  There's a wonderful joy in deleting code (for me at least!).  The less code it takes to implement a solution, the fewer entrypoints for bugs.

In an upcoming post I'll share my deployment strategy where I use exrm for hot code deploys and ansible for automation.

# Shameless plug

I'm a consultant application developer with [ThoughtWorks](http://www.thoughtworks.com) and if you would like us to work with you and your company, reach out at <mailto:mfeckie@thoughtworks.com>

In the meantime, Elixir and Ember all of the things :)

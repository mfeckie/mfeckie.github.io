---
layout: post
title: Shitty UX in the name of 'Security'
tags: [user experience, software design, security]
image:
  background: triangular.png
---
Today I saw someone in a Slack discussing a website that blocked pasting.  This led to me wanting to share some thoughts on the topic of user experience (UX) and security.

## Why does (product / service) stop me from pasting?

The only real argument that I hear for blocking users from pasting, particularly into password fields is "It's a security risk".  If that's true, lets explore what than means.

A competent, user focussed developer will always be looking for ways to keep their software as secure as possible.  This is a good thing™.  One potential vector for attack is through user entered information, this is particularly problematic for web applications / forms.

Un-sanitized user input leaves your system with the potential for SQL injection and other serious security issues.  SQL injection is a well understood problem these days and a lot of people have worked hard to provide very good defaults for most web frameworks.  If you're hand rolling a user -> database interaction and you don't understand this problem space in depth, stop doing it or get a lot of support from someone who does.  That is the responsible thing to do.

Another possible vector for attack is if a site does not scrub input for HTML / JavaScript tags.  If this is a text field, there is the risk of a malicious individual inserting their own markup to your site, which could, for example, capture user passwords or redirect them to a malicious site which results in fraud etc.  Again, if you're making software where this is a risk, you have an obligation to understand this attack vector and do something about, either using good libraries to help or with the assistance of someone experience in the area.

One particularly annoying way of 'securing' sites is blocking pasting into input fields.  Generally, on a web application, this is done using some JavaScript code.  This is a TERRIBLE idea.

Any individual with enough knowledge to be preparing a SQL injection attack or exploiting an un-sanitized user input approach is already sufficiently skilled to turn off your shitty JavaScript 'security'.  Seriously, if you're writing software that is so insecure[^1] that you believe blocking the paste function is good / necessary, you really need to invest in some genuine security training or a different career.

## 'Security' that disadvantages users without genuine benefit

Another pattern that is really grinding my gears lately is the way government services here in Australia implement 'security'.

Take, for example, the abomination that is MyGov.

To login, I'm presented with this:

![MyGov Login]({{ site.url }}/images/MyGovSignIn.png)

This require that I use an 8 character alphanumeric username which was issued when I signed up.  I can't ever remember what it is, so I either hope that I'm on a machine where I have the username saved, or I have to go digging through emails to find it.

Once I've gotten through that, I'm presented with a 2 factor authentication challenge.

![MyGov 2 Factor]({{ site.url }}/images/MyGov2Factor.png)

This is where my blood really start to boil.  I'm sent a 9 character alphanumeric one time code to my mobile phone.  Unless you've got a fantastic memory or can receive SMS on your computer, this is a really annoying process.

So I asked the question of the people of MyGov.

<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/myGovau">@myGovau</a> To access myGov I&#39;m sent a security code that is 9 characters long. Difficult to remember. How is 9 more secure than 5?</p>&mdash; Elixir of Mutton (@mfeckie) <a href="https://twitter.com/mfeckie/status/658931660646256640">October 27, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

And then this ...

<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/myGovau">@myGovau</a> how is a long security code sent to my device more secure than a short one? If my device is compromised, code length is irrelevant</p>&mdash; Elixir of Mutton (@mfeckie) <a href="https://twitter.com/mfeckie/status/659145166624702464">October 27, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

And this ...

<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/myGovau">@myGovau</a> of course they do, I&#39;m not suggesting two factor is a bad idea. What I&#39;m saying is 4 vs 9 digits.</p>&mdash; Elixir of Mutton (@mfeckie) <a href="https://twitter.com/mfeckie/status/659225247661469696">October 28, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

## Is a 9 character alphanumeric challenge more secure than 5?

I'm not against 2 factor authentication, in fact I think it's a great idea.  There are few things we protect more aggressively than our mobile <strike>Facebook viewer</strike> phones, and sending a code there is very reasonable.

Where this gets unreasonable is the WAY in which that resource is used.

2 Factor authentication works on the principle that if my password has been compromised, the likelihood that an attacker has access to my mobile phone as well is very low (e.g. this would probably need to be an in person type attack).

In this circumstance, I would argue that a 4 or 5 character code is just as secure as a 9 character one.  Here's why:

Let's take alphanumeric here to mean a-z + A-Z + 0-9, or 26 + 26 + 10.  This gives us 62 possibilities per position.

9 positions = 62 x 62 x 62 x 62 x 62 x 62 x 62 x 62 x 62 = 13,537,086,546,263,600.  In words, that 13 quadrillion possibilities.  I don't even have a mental picture of what a quadrillion even looks like, but let's just agree, it's a damn big number.

5 positions =  62 x 62 x 62 x 62 x 62 = 916,132,832.  In words, that's 916 million 132 thousand 832 possibilities.  I think you can agree that, whilst smaller than the last one, it's still very big.

4 positions = 62 x 62 x 62 x 62 = 14,776,336.  In words, that's 14 million 776 thousand 336.  Significantly smaller, but still rather large.

## So what does all this mean in terms of security?

Well, the likelihood of someone with my password, but not my mobile phone, guessing the 2 factor authentication code (assuming 3 attempts are allowed) is 3 / 13,537,086,546,263,552, which is 2.216134165758175644624123635451272010182621932036242861... × 10^-16.

This is an insanely small number.  To put in some kind of context, you've a higher chance of winning the top lotto prize twice in your life buying only one ticket a week.

Event with a 4 position code, the likelihood of guessing correctly are 2.0302732693679948804629239616641094246909382677816746993... × 10^-7.

Once again, you've still got a higher chance of winning the top prize in lotto twice in your life buying only one ticket a week!

## What's the risk of a shorter code?

From my perspective, we still have all of the benefits of 2 factor authentication even with a 4 position alphanumeric code, without causing unnecessary inconvenience to users.

I'm very much in favour of protecting the details of my users, but let's use a bit a science to understand the risk of using shorter codes.  In my view, the maths of a 4 character position code on a 2 factor SMS make it safe and convenient.

## Wrapping up

Stop and think through your UX, particularly when considering securing access to your platform.  2 factor is a great thing, but don't get carried away with the size of the challenge.  4 characters is easy to remember without making your service vulnerable to brute force attack (unless you do something stupid, like allowing infinite attempts at a verification challenge!).

[^1]: poorly written / poorly understood / poorly implemented / poorly designed or just plain shit

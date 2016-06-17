---
layout: wide-post
title: Recursion and Pattern Matching in Elixir
tag: [elixir erlang recursion pattern-matching]
image:
background: triangular.png
---

<p>In this post I will try and explain how pattern matching and recursion work in Elixir with some (hopefully) simple examples.</p>

<p>I’m not going go into the ‘why’, for that, there are many people better qualified than me to explain.</p>

<p>What I hope to acheive is a mental model for reasoning about what’s actually happening.</p>

<h1 id="lets-start-at-the-very-beginning">Let’s start at the very beginning</h1>

<p>Imagine we have a list of things and we want to know the length of it. The simplest case would be a list that is empty.</p>

<p>So given this list <code class="highlighter-rouge">[]</code>, we would expect the length to be <code class="highlighter-rouge">0</code></p>

<p>Let’s write a module that deals with this.</p>

<figure class="highlight"><pre><code class="language-elixir" data-lang="elixir"><table style="border-spacing: 0"><tbody><tr><td class="gutter gl" style="text-align: right"><pre class="lineno">1
    2
    3</pre></td><td class="code"><pre><span class="k">defmodule</span> <span class="no">MyEnumerator</span> <span class="k">do</span>
  <span class="k">def</span> <span class="n">length</span><span class="p">([]),</span> <span class="k">do</span><span class="p">:</span> <span class="m">0</span>
  <span class="k">end</span><span class="w">
  </span></pre></td></tr></tbody></table></code></pre></figure>

<p>What we have here is a function that will accept an empty list and return 0.</p>

<p>If we call it with anything other than that, we will see an error.</p>

<figure class="highlight"><pre><code class="language-elixir" data-lang="elixir"><span class="n">iex</span><span class="p">(</span><span class="m">2</span><span class="p">)</span><span class="o">&gt;</span> <span class="no">MyEnumerator</span><span class="o">.</span><span class="n">length</span><span class="p">([])</span>
  <span class="m">0</span>
  <span class="n">iex</span><span class="p">(</span><span class="m">3</span><span class="p">)</span><span class="o">&gt;</span> <span class="no">MyEnumerator</span><span class="o">.</span><span class="n">length</span><span class="p">([</span><span class="sd">"</span><span class="s2">A"</span><span class="p">,</span> <span class="sd">"</span><span class="s2">B"</span><span class="p">,</span> <span class="sd">"</span><span class="s2">C"</span><span class="p">])</span>
  <span class="o">**</span> <span class="p">(</span><span class="no">FunctionClauseError</span><span class="p">)</span> <span class="n">no</span> <span class="n">function</span> <span class="n">clause</span> <span class="n">matching</span> <span class="ow">in</span> <span class="no">MyEnumerator</span><span class="o">.</span><span class="n">length</span><span class="o">/</span><span class="m">1</span>
  <span class="ss">iex:</span><span class="m">2</span><span class="p">:</span> <span class="no">MyEnumerator</span><span class="o">.</span><span class="n">length</span><span class="p">([</span><span class="sd">"</span><span class="s2">A"</span><span class="p">,</span> <span class="sd">"</span><span class="s2">B"</span><span class="p">,</span> <span class="sd">"</span><span class="s2">C"</span><span class="p">])</span></code></pre></figure>

<p>OK, this is definitely a success, we can now determine that an empty list has a length of 0.  Well done!</p>

<h1 id="and-for-our-next-trick">And for our next trick</h1>

<p>What about a list that has one element?  Or a list that has a thousand?</p>

<p>Now we need to actually figure out what to do with the list to determine its length.  One way is to use an accumulator.  An accumulator is a place we can store the current length of the list, we can then go through and add 1 for every element.</p>

<p>Before we do that, let’s do the simplest thing that could work.</p>

<figure class="highlight"><pre><code class="language-elixir" data-lang="elixir"><table style="border-spacing: 0"><tbody><tr><td class="gutter gl" style="text-align: right"><pre class="lineno">1
  2
  3
  4
  5</pre></td><td class="code"><pre><span class="k">defmodule</span> <span class="no">MyEnumerator</span> <span class="k">do</span>
  <span class="k">def</span> <span class="n">length</span><span class="p">([]),</span> <span class="k">do</span><span class="p">:</span> <span class="m">0</span>
  <span class="k">def</span> <span class="n">length</span><span class="p">([</span><span class="n">_</span><span class="p">]),</span> <span class="k">do</span><span class="p">:</span> <span class="m">1</span>
  <span class="k">def</span> <span class="n">length</span><span class="p">([</span><span class="n">_</span><span class="p">,</span> <span class="n">_</span><span class="p">]),</span> <span class="k">do</span><span class="p">:</span> <span class="m">2</span>
  <span class="k">end</span><span class="w">
  </span></pre></td></tr></tbody></table></code></pre></figure>

<p>We can now handle a case where the list has one or two elements.</p>

<figure class="highlight"><pre><code class="language-elixir" data-lang="elixir"><span class="n">iex</span><span class="p">(</span><span class="m">4</span><span class="p">)</span><span class="o">&gt;</span> <span class="no">MyEnumerator</span><span class="o">.</span><span class="n">length</span><span class="p">([</span><span class="sd">"</span><span class="s2">A"</span><span class="p">])</span>
  <span class="m">1</span>
  <span class="n">iex</span><span class="p">(</span><span class="m">5</span><span class="p">)</span><span class="o">&gt;</span> <span class="no">MyEnumerator</span><span class="o">.</span><span class="n">length</span><span class="p">([</span><span class="sd">"</span><span class="s2">A"</span><span class="p">,</span> <span class="sd">"</span><span class="s2">B"</span><span class="p">])</span>
  <span class="m">2</span>
  <span class="n">iex</span><span class="p">(</span><span class="m">6</span><span class="p">)</span><span class="o">&gt;</span> <span class="no">MyEnumerator</span><span class="o">.</span><span class="n">length</span><span class="p">([</span><span class="sd">"</span><span class="s2">A"</span><span class="p">,</span> <span class="sd">"</span><span class="s2">B"</span><span class="p">,</span> <span class="sd">"</span><span class="s2">C"</span><span class="p">])</span>
  <span class="o">**</span> <span class="p">(</span><span class="no">FunctionClauseError</span><span class="p">)</span> <span class="n">no</span> <span class="n">function</span> <span class="n">clause</span> <span class="n">matching</span> <span class="ow">in</span> <span class="no">MyEnumerator</span><span class="o">.</span><span class="n">length</span><span class="o">/</span><span class="m">1</span>
  <span class="ss">iex:</span><span class="m">9</span><span class="p">:</span> <span class="no">MyEnumerator</span><span class="o">.</span><span class="n">length</span><span class="p">([</span><span class="sd">"</span><span class="s2">A"</span><span class="p">,</span> <span class="sd">"</span><span class="s2">B"</span><span class="p">,</span> <span class="sd">"</span><span class="s2">C"</span><span class="p">])</span></code></pre></figure>

<p>Now we can handle lists of length 1 and 2!  Huzzah.</p>

<p>And we could, go on like this forever, but that’s gonna get tedious really quickly!</p>

<h1 id="recursion-to-the-rescue">Recursion to the rescue</h1>

<p>Here we’re going to start breaking our brain a little if we’ve come from other programming paradigms.  It’s OK, it’ll be worth it, I promise (no guarantees though 😜).</p>

<p>So now we need to be able to match a list of any length.</p>

<figure class="highlight"><pre><code class="language-elixir" data-lang="elixir"><table style="border-spacing: 0"><tbody><tr><td class="gutter gl" style="text-align: right"><pre class="lineno">1
  2
  3
  4
  5
  6</pre></td><td class="code"><pre><span class="k">defmodule</span> <span class="no">MyEnumerator</span> <span class="k">do</span>
  <span class="k">def</span> <span class="n">length</span><span class="p">([]),</span> <span class="k">do</span><span class="p">:</span> <span class="m">0</span>
  <span class="k">def</span> <span class="n">length</span><span class="p">(</span><span class="n">list</span><span class="p">)</span> <span class="ow">when</span> <span class="n">is_list</span><span class="p">(</span><span class="n">list</span><span class="p">)</span> <span class="k">do</span>
  <span class="n">length</span><span class="p">(</span><span class="n">list</span><span class="p">,</span> <span class="m">0</span><span class="p">)</span>
  <span class="k">end</span>
  <span class="k">end</span><span class="w">
  </span></pre></td></tr></tbody></table></code></pre></figure>

<p>Here we will have a function that will accept one parameter, which is a list.  I’ve put the guard clause here because we only care about lists.</p>

<p>This function immediately calls another function, which take two parameters.</p>

<p>We haven’t defined this yet.</p>

<p>This may look a bit funky, but I’ll try and explain what’s happening here soon.</p>

<figure class="highlight"><pre><code class="language-elixir" data-lang="elixir"><table style="border-spacing: 0"><tbody><tr><td class="gutter gl" style="text-align: right"><pre class="lineno">1
2
3
4
5
6
7
8
9
10</pre></td><td class="code"><pre><span class="k">defmodule</span> <span class="no">MyEnumerator</span> <span class="k">do</span>
  <span class="k">def</span> <span class="n">length</span><span class="p">([]),</span> <span class="k">do</span><span class="p">:</span> <span class="m">0</span>
  <span class="k">def</span> <span class="n">length</span><span class="p">(</span><span class="n">list</span><span class="p">)</span> <span class="ow">when</span> <span class="n">is_list</span><span class="p">(</span><span class="n">list</span><span class="p">)</span> <span class="k">do</span>
  <span class="n">length</span><span class="p">(</span><span class="n">list</span><span class="p">,</span> <span class="m">0</span><span class="p">)</span>
  <span class="k">end</span>
  <span class="k">def</span> <span class="n">length</span><span class="p">([],</span> <span class="n">accumulator</span><span class="p">),</span> <span class="k">do</span><span class="p">:</span> <span class="n">accumulator</span>
  <span class="k">def</span> <span class="n">length</span><span class="p">([</span><span class="n">_head</span> <span class="o">|</span> <span class="n">tail</span><span class="p">],</span> <span class="n">accumulator</span><span class="p">)</span> <span class="k">do</span>
  <span class="n">length</span><span class="p">(</span><span class="n">tail</span><span class="p">,</span> <span class="n">accumulator</span> <span class="o">+</span> <span class="m">1</span><span class="p">)</span>
  <span class="k">end</span>
  <span class="k">end</span><span class="w">
  </span></pre></td></tr></tbody></table></code></pre></figure>

<p>We now have two functions, in the first we’re saying, if you give me an empty list AND and an accumulator, just return the accumulator.  This is also known as a base clause, this is where the recursion stops.  Makes sense, right?  Once a list is empty, we can stop counting the elements it contains.</p>

<p>If the function receives a list with 1 or more elements, we’ll start/continue counting the elements by adding one to the accumulator and then calling length with the tail of the list.  This is recursion.</p>

<p>Let’s try and work through a short list to visualise how the functions are called and the data structures passed around.</p>

<table class="function-calls">
  <thead>
    <tr>
      <th>Step</th>
      <th>Data Structure</th>
      <th>Function Call</th>
      <th>Matches</th>
    </tr>
  </thead>
  <tbody>

    <tr>
      <td>1</td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="p">[</span><span class="sd">"</span><span class="s2">A"</span><span class="p">,</span> <span class="sd">"</span><span class="s2">B"</span><span class="p">]</span></code></td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="n">length</span><span class="p">([</span><span class="sd">"</span><span class="s2">A"</span><span class="p">,</span> <span class="sd">"</span><span class="s2">B"</span><span class="p">])</span></code></td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="n">length</span><span class="p">(</span><span class="n">list</span><span class="p">)</span> <span class="ow">when</span> <span class="n">is_list</span> <span class="n">list</span></code></td>
    </tr>

    <tr>
      <td>2</td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="p">[</span><span class="sd">"</span><span class="s2">A"</span><span class="p">,</span> <span class="sd">"</span><span class="s2">B"</span><span class="p">]</span></code></td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="n">length</span><span class="p">([</span><span class="sd">"</span><span class="s2">A"</span><span class="p">,</span> <span class="sd">"</span><span class="s2">B"</span><span class="p">],</span> <span class="m">0</span><span class="p">)</span></code></td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="n">length</span><span class="p">([</span><span class="n">head</span><span class="o">|</span><span class="n">tail</span><span class="p">],</span> <span class="n">accumulator</span><span class="p">)</span></code></td>
    </tr>

    <tr>
      <td>3</td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="p">[</span><span class="sd">"</span><span class="s2">A"</span><span class="p">,</span> <span class="sd">"</span><span class="s2">B"</span><span class="p">],</span> <span class="m">0</span></code></td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="n">length</span><span class="p">([</span><span class="sd">"</span><span class="s2">A"</span><span class="p">,</span> <span class="p">[</span><span class="sd">"</span><span class="s2">B"</span><span class="p">]],</span> <span class="m">0</span><span class="p">)</span></code></td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="n">length</span><span class="p">([</span><span class="n">head</span><span class="o">|</span><span class="n">tail</span><span class="p">],</span> <span class="n">accumulator</span><span class="p">)</span></code></td>
    </tr>

    <tr>
      <td>4</td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="p">[</span><span class="sd">"</span><span class="s2">B"</span><span class="p">],</span> <span class="m">1</span></code></td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="n">length</span><span class="p">([</span><span class="sd">"</span><span class="s2">B"</span><span class="p">,</span> <span class="p">[]],</span> <span class="m">1</span><span class="p">)</span></code></td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="n">length</span><span class="p">([</span><span class="n">head</span><span class="o">|</span><span class="n">tail</span><span class="p">],</span> <span class="n">accumulator</span><span class="p">)</span></code></td>
    </tr>

    <tr>
      <td>5</td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="p">[],</span> <span class="m">2</span></code></td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="n">length</span><span class="p">([],</span> <span class="m">2</span><span class="p">)</span></code></td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="n">length</span><span class="p">([],</span> <span class="n">accumulator</span><span class="p">)</span></code></td>
    </tr>

    <tr>
      <td>6</td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="m">2</span></code></td>
      <td><code class="highlight pad language-elixir" data-lang="elixir"><span class="m">2</span></code></td>
      <td>--</td>
    </tr>
  </tbody>
</table>

<p>Hopfully you can see that this has a very similar effect to ‘popping’ from a stack in another paradigm, but the nice thing here is our orginal list remains unchanged.</p>

<p>It’s kinda neat though, that we just keep calling the same named function until such time as we’re done.</p>

<p>This ends my simple intro to recursion, for something a bit more in depth, I recommend <a href="http://learnyousomeerlang.com/recursion#hello-recursion">http://learnyousomeerlang.com/recursion#hello-recursion</a></p>

<h1 id="pattern-matching">Pattern matching</h1>

<p>We’ve had a basic introduction to recursion, which relied heavily on pattern matching too.</p>

<p>Pattern matching can be a great way to reduce conditional logic, allowing very clear isolation and simpler testing.</p>

<p>A common coding challenge is to write code that determines whether a given year is a leap year.</p>

<p>The rules are fairly simple.  A leap year is evenly divisible by 4, except if years what are evenly divisible by 100, unless it is also evenly divisible by 400!.</p>

<p>So we could do the super terse thing and make just a bunch of if <code class="highlighter-rouge">rem(year, 4) == 0 || ...</code></p>

<p>Or we could make use of pattern matching.</p>

<figure class="highlight"><pre><code class="language-elixir" data-lang="elixir"><table style="border-spacing: 0"><tbody><tr><td class="gutter gl" style="text-align: right"><pre class="lineno">1
2
3
4
5
6
7
8
9
10</pre></td><td class="code"><pre><span class="k">defmodule</span> <span class="no">Year</span> <span class="k">do</span>
  <span class="k">def</span> <span class="n">is_leap?</span><span class="p">(</span><span class="n">year</span><span class="p">)</span> <span class="k">do</span>
  <span class="k">case</span> <span class="p">{</span> <span class="n">rem</span><span class="p">(</span><span class="n">year</span><span class="p">,</span> <span class="m">4</span><span class="p">),</span> <span class="n">rem</span><span class="p">(</span><span class="n">year</span><span class="p">,</span> <span class="m">100</span><span class="p">),</span> <span class="n">rem</span><span class="p">(</span><span class="n">year</span><span class="p">,</span> <span class="m">400</span><span class="p">)</span> <span class="p">}</span> <span class="k">do</span>
  <span class="p">{</span><span class="m">0</span><span class="p">,</span> <span class="m">0</span><span class="p">,</span> <span class="m">0</span><span class="p">}</span> <span class="o">-&gt;</span> <span class="no">true</span>
  <span class="p">{</span><span class="m">0</span><span class="p">,</span> <span class="m">0</span><span class="p">,</span> <span class="n">_</span><span class="p">}</span> <span class="o">-&gt;</span> <span class="no">false</span>
  <span class="p">{</span><span class="m">0</span><span class="p">,</span> <span class="n">_</span><span class="p">,</span> <span class="m">0</span><span class="p">}</span> <span class="o">-&gt;</span> <span class="no">true</span>
  <span class="n">_</span> <span class="o">-&gt;</span> <span class="no">false</span>
  <span class="k">end</span>
  <span class="k">end</span>
  <span class="k">end</span><span class="w">
  </span></pre></td></tr></tbody></table></code></pre></figure>

<p>With this style, we can make decisions based on the <em>shape</em> or <em>patter</em> of our data.</p>

<p>Now this example isn’t necessarily the best use case for pattern matching, but I think it highlights some of the potentional.</p>

<p>When you combine pattern matching with the pipeline operator and multiple function definitions we have a really nice opportunity to get rid of nested and hard to reason conditional logic.</p>

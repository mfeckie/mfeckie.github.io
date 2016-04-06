---
layout: post
title: Custom Mix Tasks for Database Seeding
tags: [elixir, phoenix, devops]
image:
  background: triangular.png
---

# Why a custom Mix task?

I love the built in support for seeds with Phoenix, but sometimes I don't want to programatically generate a bunch of stuff, sometimes I just want to initalize the database in a known state.  This is particularly useful for things that rarely change, like category names or lists of countries.

Leaning on tools provided by Postgres can also be significantly faster than running a load of transactions.

## How to define my own Mix task?

The docs for Mix show that in order to declare a Task, we simply `use Mix.Task` and provide a `run/1` function. <http://elixir-lang.org/docs/v1.2/mix/Mix.Task.html>

That's pretty simple!

Lets do that.

Start off with a basic Mix project.  `mix new db_seeder` 

Move into the newly created project. `cd db_seeder`

Next we'll create a folder to store things neatly `mkdir -p lib/mix/tasks`.

Create the following file `lib/mix/tasks/hello_world.ex`

{% highlight elixir linenos %}
defmodule Mix.Tasks.HelloWorld do
  use Mix.Task
  
  def run(_args) do
    Mix.shell.info "Hello, World!"
  end
end
{% endhighlight %}

Compile the application `mix compile`.

If we run `mix help`, we don't see our task, but if we excute the command, it works.  **Note** the module name is converted to snake_case when invoking commands.

{% highlight bash linenos %}
mix hello_world
Hello, World!
{% endhighlight %}

So why doesn't it appear in our task list when running `mix help`?  Well it's because documentation is a first class citizen in Elixir, so if we don't document the task, it doesnt exist!

Let's fix that.

{% highlight elixir linenos %}
defmodule Mix.Tasks.HelloWorld do
  use Mix.Task

  @shortdoc "Say's Hello to the glorious World!"
 
  def run(_args) do
    Mix.shell.info "Hello, World!"
  end
end
{% endhighlight %}

`mix compile` then `mix help`

{% highlight bash linenos %}
mix escript.build     # Builds an escript for the project
mix hello_world       # Say's Hello to the glorious World!
{% endhighlight %}

Whoop, there it is!

## A less trivial task

So given a collection of raw SQL files I want to run them again the database in my current environment.

The first thing I think I need is a way of talking directly to the shell.

Luckily, Mix provides a Shell module for that.  Let's test it out.

{% highlight elixir linenos %}
defmodule Mix.Tasks.Seeder do
  use Mix.Task

  @shortdoc "Seeds the database"

  def run(_args) do
    Mix.shell.cmd("psql -d my_super_site_#{Mix.env} -c 'select * from users'")
  end


end

{% endhighlight %}

Assuming you have a table that matches the query, you'll see its output if you compile and run the task.

### Now let's get some actual files

The File module allows us to query our current working directory, so we can easily find a relative path.

Let's wrap that in a helper.

{% highlight elixir linenos %}
defmodule Mix.Tasks.Seeder do
  use Mix.Task

  @shortdoc "Seeds the database"

  def run(_args) do
    Mix.shell.cmd("psql -d my_super_site_#{Mix.env} -c 'select * from users'")
  end

  defp sql_dir do
    Path.join([File.cwd!, "priv", "repo"])
  end

end

{% endhighlight %}

Now, I'm going to hard code the files for seeding, thought we could pass them in as args.  For my use case, they are basically unchanging.

{% highlight elixir linenos %}
defmodule Mix.Tasks.Seeder do
  use Mix.Task

  @shortdoc "Seeds the database"

  def run(_args) do
    Mix.shell.cmd("psql -d my_super_site_#{Mix.env} -c 'select * from users'")
  end

  defp sql_dir do
    Path.join([File.cwd!, "priv", "repo"])
  end
  
    defp file_names do
    [
      "things_1.sql",
      "things_2.sql",
      "things_3.sql"
    ]
  end

end
{% endhighlight %}

And of course, we now need to run those files, rather thant the static query.

With a spot of string interpolation, we can easily iterate over those files.

{% highlight elixir linenos %}
defmodule Mix.Tasks.Seeder do
  use Mix.Task

  @shortdoc "Seeds the database"

  def run(_args) do
    exec_sql
    Mix.shell.info "Seeding completed successfully"
  end


  defp exec_sql do
    Enum.each(file_names, fn file_name ->
      Mix.shell.cmd("psql -d my_super_site_#{Mix.env} -f #{Path.join(sql_dir, file_name)}")
    end)
  end

  defp sql_dir do
    Path.join([File.cwd!, "priv", "repo"])
  end
  
    defp file_names do
    [
      "things_1.sql",
      "things_2.sql",
      "things_3.sql"
    ]
  end

end
{% endhighlight %}

And with that, we're done.

## Wrap up

As you can see, it's fairly trivial to implement your own Mix tasks in Elixir.  Of course you could easily write this with a shell script, but where would the fun be in that!

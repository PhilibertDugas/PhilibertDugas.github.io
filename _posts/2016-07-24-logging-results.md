---
layout: post
---
## Logging results of requests

*July 24th, 2016*

After a week of vacation, I'm back to learning Elixir (and blogging about it). This week I tackled the problem of logging results to a file. Load testing without results to look at wouldn't be efficient uh?

First thing first, what do we want to log? Thinking about it, we want at least the following points:

- Status code of the response
- Body of the response
- Benchmarking results (eg. execution time)

Looking at the current code, this might be tricky. In my second [post](http://philibertd.com/2016/07/02/load-testing.html) we defined the `HTTPSender` module. The module was designed to send asynchronous HTTP requests. This is great, but the implementation isn't meeting our need. By using the built-in asynchronous feature of [HTTPotion](https://github.com/myfreeweb/httpotion#asynchronous-requests), we split the result into 3 Elixir structs: `AsyncHeaders`, `AsyncChunk`, and `AsyncEnd`. Since we trigger many requests at once, it's hard to associate headers and chunks. Furthermore, instrumenting the request time would probably require some hack around.

#### Refactoring the `HTTPSender` module

To fix the issues described above, I decided to shy away from HTTPotion's asynchronous model. Instead, I wrap the the request into it's own [Task](http://elixir-lang.org/docs/stable/elixir/Task.html). Turns out the code becomes much simpler:

{% gist d1f981d401cb4f0de337e00a55315506 %}

We don't have to worry anymore about launching a receiver process to process the many parts of the response. Easy enough, launch a thread which sends a request and stores the result in some variables. With this, we'll be able to log the variables.


#### Log the results

First thing to do, google about elixir logging. I land on the [Logger](http://elixir-lang.org/docs/stable/logger/Logger.html) module. Reading through the page, it's explained that you have to define [backends](http://elixir-lang.org/docs/stable/logger/Logger.html#module-backends) for the logger. Those backends define where the message will be logged. Looked easy to begin with, turns out Elixir doesn't provide a file backend by default (what I'm looking for). Luckily, we can define our custom backends. As a first iteration, I'll just log to the console and then I can worry about the file backend.


#### Console logging

Configuring the logger is the first thing to do. This configuration will live in the `config/config.exs` file. Just following the example from the doc, a good old copy/paste gives the following:

{% highlight elixir %}
config :logger, backends: [:console], compile_time_purge_level: :info
{% endhighlight %}

The backends keyword is explicit enough, we will log to the console. The second option is intriguing. It's easy to copy/paste code without understanding what it does. If I don't understand the code, I'll always take the time to learn about it. Turns out `compile_time_purge_level` will remove any `Logger` statement lower than `:info` at compile time. That's amazing, the runtime won't have to worry about other logging statements. The first iteration of logging looks like this:

{% gist 82c921e5242ea8c403e1e95113badc41 %}

Two lines got added. First, the `require Logger` is needed to access the module's macro. The second line is an invocation of the [info](http://elixir-lang.org/docs/stable/logger/Logger.html#info/2) macro. I'll spare the detail of adapting the other code to this new module and jump straight to results.

{% highlight elixir %}
iex(1)> Runner.main
Finished starting threads
finished sending request
finished sending request
finished sending request
finished sending request
finished sending request
finished sending request
finished sending request
finished sending request
finished sending request
finished sending request
:ok
iex(2)>
12:59:46.977 [info]  Request http://philibertd.com finished with status code: 200

12:59:46.977 [info]  Request http://philibertd.com finished with status code: 200

12:59:46.977 [info]  Request http://philibertd.com finished with status code: 200

12:59:46.977 [info]  Request http://philibertd.com finished with status code: 200

12:59:46.977 [info]  Request http://philibertd.com finished with status code: 200

12:59:46.977 [info]  Request http://philibertd.com finished with status code: 200

12:59:46.977 [info]  Request http://philibertd.com finished with status code: 200

...
{% endhighlight %}

#### File logging

We saw the result already of logging to the console, next step is to log this into a file. Should be a simple configuration to a file backend for the logger. Luckily enough, someone already tackled the [file backend](https://github.com/onkel-dirtus/logger_file_backend) problem. Since there's no instructions about how to include it, here's a snapshot of my `mix.exs` to use the backend

{% gist 4940a553bb3cadd8b511ce35d8f4f102 %}

Having this done, I configure the logger by giving a file to log into.

{% gist 191b63d3dc361f055a8d31ef651aec5a %}

We changed here the backends to be a `LoggerFileBackend` and naming this backend `:info_log`. We then configure this `:info_log` backend with a file to log into and a `:info` level. Just like magic, it works out of the box!

{% highlight elixir %}
iex(1)> Runner.main
Finished starting threads
finished sending request
finished sending request
finished sending request
finished sending request
finished sending request
finished sending request
finished sending request
finished sending request
finished sending request
finished sending request
:ok

 cat info.log
 13:57:14.115 [info] Request http://philibertd.com finished with status code: 200
 13:57:14.115 [info] Request http://philibertd.com finished with status code: 200
 13:57:14.115 [info] Request http://philibertd.com finished with status code: 200
 13:57:14.115 [info] Request http://philibertd.com finished with status code: 200
 13:57:14.115 [info] Request http://philibertd.com finished with status code: 200
 13:57:14.115 [info] Request http://philibertd.com finished with status code: 200
 13:57:14.115 [info] Request http://philibertd.com finished with status code: 200
 13:57:14.115 [info] Request http://philibertd.com finished with status code: 200
 13:57:14.115 [info] Request http://philibertd.com finished with status code: 200
...
{% endhighlight %}

#### Conclusion

Logging in elixir turns out to be quite flexible. Custom backends is a great way to change how the data is logged without changing any code. The next step is to instrument the requests with the time of execution. I'll want also to start tackling more complex scenarios than the current one.

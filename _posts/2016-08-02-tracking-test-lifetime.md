---
layout: post
---
## Rethinking the load test

*August 2nd, 2016*

Over the past few weeks, I've developed a routine in the weekends. Every Saturday, I spend around 2-3 hours coding some elixir. If it's not too late, I'll write my blog post about the coding session. Worst case scenario, I write the blog post on Sunday.

This week, I wanted to migrate my previous code into an Phoenix project and build a web page out of it. The first iteration is simple. Go to the web page, click a button to start a load test, then see results when the test is over. **When the test is over**. That's not something the current implementation was tackling.

I had to halt on the Phoenix migration for this weekend and revisit my current code. The first step was to refactor the `HTTPSender` which had the wrong implementation. The second step was to detect when the test is done.

#### Refactoring HTTPSender

In the previous weeks, the `HTTPSender` was always sending asynchronous requests. Looking back, it makes no sense. If we have 10 threads which are expected to each do 10 http requests, we want those requests to be sequential. By putting those requests asynchronous, we are actually launching 100 threads. Each of these 100 threads are then executing 1 HTTP request. Furthermore, the criteria of completion for a thread is that it's requests are done. Making those requests sequential is going to simplify the code.

Another thing I wanted to tackle was the request execution time. Erlang has a great function built to calculate the execution time of a function. This is exactly what I use in the snippet below.

{% gist 3d1195ee77c28c3826e5de168ade1d1e %}

Few changes from last week. First we deal with any kind of methods now. HTTPotion expects a downcased atom version of the method to send. Since Poison returns string keys (and not atom keys) I have a little bit of casting to do.

Second, [:timer.tc](http://erlang.org/doc/man/timer.html#tc-1) calculates to execution time of the HTTP request. I love how it returns a `{Time, Value}` format. It's much cleaner than wrapping the function with timestamps and subtracting them afterward. Since the time value is in nanoseconds, I divide by a factor of 1000 to log a millisecond value. Milliseconds are more appropriate for HTTP requests.

#### Waiting for tasks

Now that we dealt with the easy stuff, time for some asynchronous fun. First of, I was using mostly `Task.start` in my previous code. Reading the [Elixir documentation](http://elixir-lang.org/docs/stable/elixir/Task.html#summary), I noticed that `Task.async` was the alternative when a task must be awaited on.

Second, I have many tasks running (1 task by thread) so I need to wait for many tasks at once. Good thing `Task.yield_many` is there for me.

Finally, I have to keep all those tasks in a list so that I can await on this list when needed. There are different ways to keep a state in Elixir and I opted for the easiest one in my opinion. Sending the list of tasks as an argument in the function allows me to await on these tasks when needed.

{% gist 10c3eff44cbf287515a76ad2fbb7adfc %}

The first argument is the thread group. This group defines how many threads to launch, the request information and it's repetitions.

The second argument is the list of tasks to be awaited on. These tasks represent launched threads in the context of the load test.

Every time we launch a thread, we use the `Task.async` function. On every iteration, we insert this task into the `tasks` argument by using the `List.insert_at(tasks, 0, task)` function. When the thread count is down to 0, we finished launching all the thread. We are then ready to wait for their completion. `Task.yield_many(tasks, 30000)` will wait until every tasks are done up to a maximum of 30 seconds. This hardcoded value of 30 seconds is definitely something to change but it allowed me to test the code.

There we have it, this current code launches many threads. Each threads will execute a many sequential HTTP requests. The main code will wait until every thread is done which would indicate the end of the load test.

#### What's next

Now that I can detect when a test is finished, we can store the results in a database and show it with Phoenix. Next week, I'll be going through my first Phoenix project to launch basic load tests.

There's still a lot of ground to cover, I haven't been exploring unit testing at all so far. I'd love also to use [Phoenix channels](http://www.phoenixframework.org/docs/channels) to display live stats of a current running load tests. Stay tuned in the upcoming weeks as I tackle all these areas.

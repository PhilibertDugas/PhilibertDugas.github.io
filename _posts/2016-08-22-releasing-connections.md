---
layout: post
---
## Releasing server connections

*August 22nd, 2016*

In the last blog post, I showed how I migrated my load testing tool to a web application.
Using the Phoenix framework, it was straight forward to get started and have early results.
The web application could trigger load test and display results.
While this was exciting to begin with, it presents an issue.
Waiting for a load test to finish can be a long process.
During that waiting time, we are holding a connection open to the server.
It wouldn't be hard to open many tabs, and launch long running tests before the server starts blowing up.

This week, I focused on refactoring the code so that the connections to the server are released.
By refreshing a "progress" page, it's possible to see if the test is still running.
In the world of Rails, it would be the equivalent of launching a job.

#### Launching the task

I thought at first that this was going to be an easy thing to do, **not so fast**.
Phoenix doesn't come with a job framework like [Rails do](http://guides.rubyonrails.org/active_job_basics.html)
With a little bit of research, I found some open source frameworks like [exq](https://github.com/akira/exq).
While some of these libraries are most likely the right solution long term, I wanted to try something simpler.

The simplest iteration I could think about was the following:

1. Trigger a load test with the `create` action
2. Start an asynchronous job and redirect the user to a "in progress" page
3. Verify the load test status when the user refreshes the "in progress" page

Triggering a load test isn't too hard, we've been doing this for the past few week.
Retrieving this load test in the context of another action was more challenging.
Essentially, we need a global state on the server side.

Having a global state is not a good idea when your application needs to scale, but it's easy to implement.
Since I like to make small steps, I started this way.

{% gist b686d36e74ac17721a820d247af7cf30 %}

This code snippet shows the `create` action of the controller. It starts by using `Task.start` to launch the load test.
We don't need to use `Task.async` since we are not going to await for the end of this process in this function.
What we see afterward is storing the state in a global `Map`. Essentially, storing a hard-coded string as the key, with the task as the value.
This allows us to retrieve this task in another action. For this to work, we need some configuration in `lib/elt.ex`

{% gist b8cc19984c0c368ad49b180d6872c234 %}

The important part of this section is at line 13. When the Phoenix server starts, it calls `Map.Bucket.start_link`.
It also makes this process supervised, meaning that it will be restarted if it crashes.
With this, we can store all our tasks in a global `Map`

#### Retrieving the task

Retrieving the task at this point is trivial. Simply query the map with the appropriate id to get the task.

{% gist 2f477e75c543430729a57909ec9d5c91 %}

Right now, the key is hardcoded with **"1"**. This needs to be changed and passed as a parameter to the function.
Once I retrieved the load test in another action, using `Process.alive?(pid)` tells me if the test is over or not.
This is a very simple check, and it could be elaborated further in the future.

If the process is still running, we render a "progress" page explaining how the load test is still running.
In the other case, we redirect the user to the list of results.

#### Result

It's always rewarding to see some results :)

[![Async load test](http://img.youtube.com/vi/S5q5gGLXqbc/0.jpg)](https://www.youtube.com/watch?v=S5q5gGLXqbc&feature=youtu.be)

#### Improvements

The current implementation works fine for quick development, but it's worth thinking about potential improvements.

1. Using an external queue would allow better scalability
2. Supervising the load tests to handle crashes and stuff
3. Updating the "progress" page with websocket

There's many things that I look forward to, stay tuned for the progress of this project !

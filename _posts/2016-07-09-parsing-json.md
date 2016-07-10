---
layout: post
---
## Parsing JSON and sending requests

*July 9th, 2016*

This week I focused on getting a minimal test plan defined in JSON and executing it. Defining the plan wasn't too hard, I took a trimmed down version of [Golt's plan](https://github.com/dudang/golt/blob/master/test/golt-test.json). The light version looks like [this](https://github.com/PhilibertDugas/elixir-learning/blob/master/multithread/elt-test.json). This test will start 10 threads. Each of them will execute 3 GET requests to my blog. Simple enough, let the hacking begin.

#### Parsing JSON
First, I need to read the file. Once I have it's content, I want to parse the JSON into a struct. Not going to implement my own JSON parser, [Poison](https://github.com/devinus/poison) is the library I'm using to do that.

{% gist 1df360053e0a3cba5643ecf5516cdc86 %}

So we have a module `Runner` which will be acting as the entry point for our program. The new function `get_struct` reads the file "elt-test.json". It then transform the content into the `Elt` struct defined [here](https://github.com/PhilibertDugas/elixir-learning/blob/master/multithread/lib/elt.ex). Nothing too tricky. Time to start launching the threads. Each of these threads will launch their own series of requests.

#### Launching Threads

{% gist badb971bd3d87978f7e7a4d390566d35 %}

We introduced quite a lot of code here, let's go through it. Right now, I only want to worry about the first element of the `Elt` list. The line `[ thread_group | _ ]` takes the head of the list and assigns it to `thread_group`. The rest of the list is discarded since I'm using the `_` operator.

I have my thread group structure ready to get kicked off. The function `launch_threads` is responsible for this. An amazing feature of elixir is how you can use pattern matching in function arguments. I'm expecting a map with the following keys: `{ "repetitions" =>, "requests" =>, "threads" => }`. The declaration above assigns each value to the variable declared in the function signature. Moreover, the `param` variable is assigned the whole map value.

Now, I have 10 threads to start. In an imperative language, I could loop 10 times and start background threads. In this context, I will use recursion 10 times. Using the `case` statement, I can inspect the value of `threads`. If it's at 0, my loop is finished and I can output a message to the console. For other values, I'm starting a `Task` using [HTTPSender](http://philibertd.com/2016/07/02/load-testing.html) defined in last week's post. We see here, `launch_request` is not currently implemented. It will be in the next iteration so no need to worry about for now.

Back to launching the following threads. I'm calling the same method, decrementing `threads` value by 1 to avoid looping infinitely. Note the syntax, `%{ param | "threads" => threads - 1}`. This creates a new map, having all of `param` values except for `"threads"`.

#### Launching Requests

{% gist 8cac66be77b1ac5a3dcaec587e1ba7e8 %}

The last piece of the puzzle is to send HTTP Requests. The new function `launch_request` receives a url, a number of repetitions and a receiver pid. To know more about this `pid`, make sure to read last week's post. Using recursion again, I'm using the `HTTPRequest` module which sends asynchronous requests. Once that's done, I'm repeating the function until there are no more repetition. Simple enough.

#### Result

What's the point of coding all of this if we don't see it in action? Time for some demo!

{% highlight elixir %}
iex> Runner.main
Finished starting threads
:ok
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
200
200
200
200
200
200
200
200
200
200
200
Request Chunk Received
Request Chunk Received
Request Chunk Received
200
Request Chunk Received
Request Chunk Received
Request Chunk Received
Request Chunk Received
Request Chunk Received
Request Chunk Received
end
end
end
Request Chunk Received
end
end
end
end
end
end
end
.........
{% endhighlight %}

The output is quite clear. First it finishes booting all the threads. Then it finishes sending all the requests. Since they are asynchronous, we don't get the result right away. Once the requests are done, we start receiving the `AsyncHeader`, `AsyncChunk`, and `AsyncEnd` structs from HTTPotion. Some requests seem a bit slower then others. Note the `AsyncHeader` that arrived after 3 `AsyncChunk`. With a longer running test (more requests), we would probably see much more of them. I trimmed down the output to the first 10 requests. This output keeps going for all 30 requests.

#### Conclusion

This first step was for the simplest scenario. Still, I'm super impressed by the power of Elixir. In around 33 lines of code, we can get a command line load testing tool. Elt is very limited, in the upcoming weeks I plan to keep exploring and learning Elixir through this project. One thing I want to approach soon is Elixir testing, logging and eventually the [Phoenix framework](http://www.phoenixframework.org/).

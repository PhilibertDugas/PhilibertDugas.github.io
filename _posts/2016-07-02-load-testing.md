---
layout: post
---
## Elt, Elixir load testing

*July 2nd, 2016*

Back in late 2015, I came across this code review assigned to me. Over 500 lines of XML code written by a colleague QA at the time, a JMeter load testing scenario.
No way I could understand everything going on in there, so I decided I'd try this scenario on my local machine.

Checkout the branch, kick off the JMeter repulsive GUI, start the test. Computer crashed in around 30 seconds.
There were two main problems with this. First, the scenario itself was hard to review because of the XML syntax.
Second, it was difficult to run the scenario locally as the JMeter would take too much resources on the computer.

That's when I started creating [Golt](https://github.com/dudang/golt). It's purpose was to fix those two problems specifically.
The test scenarios are in YAML or JSON format, which is much more reviewable.
The program itself uses Go to leverage it's lightweight threading system.
I found this to be a great project to learn Go. Not a trivial application, but nothing too complex neither. While I never got to use Golt back at work, I enjoyed the learning process.

I decided I want to do the same thing with Elixir. Writing a load tester which uses YAML or JSON as an input. When I read the following lines in the Elixir getting started's guide, it convinced me even more:

> *Elixir’s processes should not be confused with operating system processes. Processes in Elixir are extremely lightweight in terms of memory and CPU (unlike threads in many other programming languages). Because of this, it is not uncommon to have tens or even hundreds of thousands of processes running simultaneously.*

So here I go. The first step this week was to send HTTP requests without worrying about parsing YAML or JSON. While searching about how to send HTTP requests in Elixir, I found [HTTPotion](https://github.com/myfreeweb/httpotion). This client interesting as it supported asynchronous requests out of the box. I decided to build a module around it to see how it goes.

{% gist c22881b7a07038b2ccacadff357f06c1 %}

This module serves provides two functions. The first one sends a request and streams it's result to a receiver process. The second one boots a receiver process and returns it's PID. This first iteration turned out to be wrong:

```
iex> {:ok, pid} = HTTPSender.receive_request
{:ok, #PID<0.134.0>}

iex> HTTPSender.send_request("http://philibertd.com", pid)
%HTTPotion.AsyncResponse{id: -576460752303423304}
** (EXIT from #PID<0.159.0>) an exception was raised:
    ** (KeyError) key :to_string not found in: %HTTPotion.AsyncHeaders
```

Right, so I can't cast the `AsyncHeaders` struct to string. Let's adjust this and match the documented structs:

{% gist b453845c928864495185d04325ec7d5b %}

Here, I'm using using pattern matching to extract the information I want. In this case, the status code from the headers and the chunk content. The second try went this way:

```
iex> {:ok, pid} = HTTPSender.receive_request
{:ok, #PID<0.134.0>}

iex> HTTPSender.send_request("http://philibertd.com", pid)
%HTTPotion.AsyncResponse{id: -576460752303423304}
200
```

So we see some improvements here. We received the `AsyncHeaders` and it printed out the status code value. Problem is, we didn't handle the `AsyncChunk` and `AsyncEnd`. Turns out that a `Task` will execute the function and stop afterwards. Since the response is sent in 3 different messages, we want to loop until we receive the `AsyncEnd` message.

The final iteration looks like this:

{% gist 06a381853f4fa4e552296865a065dc45 %}

In this example, we introduced a new private function `loop`. This function starts by receiving the messages from the http response. It then loops until it receives the `AsyncEnd` message at which points it stops. Since we’re doing functional programming, looping is actually recursion! Note that our public method now doesn’t define the logic in it’s `Task.start_link`. It delegates to `loop` which is cleaner in my opinion:

```
iex> {:ok, pid} = HTTPSender.receive_request
{:ok, #PID<0.134.0>}

iex> HTTPSender.send_request("http://philibertd.com", pid)
%HTTPotion.AsyncResponse{id: -576460752303423487}
200
<!DOCTYPE html>
<html>
  <head>
    <!-- Begin Jekyll SEO tag v2.0.0 -->
<title>Philibert’s website</title>
........
end
```

We send the request out and are able to match all 3 messages! To make the output more readable, I trimmed down the chunk of the response. Now that we are able to send asynchronous http requests, the next step is to build a test plan and parse it. I'll be doing this in the next post!

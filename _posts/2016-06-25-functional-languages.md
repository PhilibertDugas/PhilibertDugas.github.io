---
layout: post
---
## Elixir: My introduction into functional programming

My first exposure to web development was during a summer internship back when I was in college. At the time, I was using PHP, JQuery, HTML, CSS, nothing too extravagant. The fall semester that followed, I took a web class. I thought it would be a ride through, that is until we started to use [Yesod](http://www.yesodweb.com) for the class. A web framework built in [Haskell](https://www.haskell.org). It was my first time programming in a functional language: brutal. Back then, I thought I’d never be able to learn functional programming. I felt it was so different than anything I had ever done. `for` loops? Nope. Side effect? Can't do that either.

Still, functional programming always intrigued me, I tried around a year ago to pick up [Scala](http://www.scala-lang.org/). I felt it was much more approachable than Haskell, but still not for me. Recently [Elixir](http://elixir-lang.org) came under my radar. Elixir is a functional language built on top of the [Erlang](https://www.erlang.org/) VM, which has many similarities to Ruby. I feel like Elixir is the perfect opportunity for me to dive back into functional programming. It's similarity with Ruby, the language I use the most these days, makes it a great candidate for me to learn it.

So far, I think the language is amazing. It combines Ruby expressiveness and Erlang stability. The functional aspect that felt so intimidating back then feels now like hours of fun.

#### Pipe operator

I got to say the pipe operator is a cool aspect of the language. If you're familiar with the Unix pipe operator, you'll be right at home. Using the example taken from the [Getting Started Guide](http://elixir-lang.org/getting-started/enumerables-and-streams.html#the-pipe-operator), a chain of function call would look like this:

```
iex> Enum.sum(Enum.filter(Enum.map(1..100_000, &(&1 * 3)), odd?))
7500000000
```

Not too friendly to read uh? First, we're creating a range of numbers with `1..100_000`. Next, we're passing it through `Enum.map` giving it a method `&(&1 * 3)`. In this case `&1` is the first argument passed to the function. This would multiply every number in the range by 3. We pass in this result to the `Enum.filter` method along with the `odd?` function, which filters out every even numbers. Finally we sum the result of the resulted range, giving a final result of `7500000000`. I find all this function nesting quite hard to read, let's see how the pipe operator helps:

{% highlight elixir %}
iex> 1..100_000 |> Enum.map(&(&1 * 3)) |> Enum.filter(odd?) |> Enum.sum
7500000000
{% endhighlight %}

The pipe operator takes the output from the left side and passes it along as an argument to the right side. Comparing this line of code with the other one above, I feel it’s much more expressive. It represents the flow of data in a more concise way. Functions play such a big role in functional programming. I think having an operator like this one is brilliant and will contribute to much cleaner code.

#### What's next
I’m planning to write a few projects using Elixir and Phoenix since I want to dive deep into this. I’ll try to write a blog post per week about my progress. To conclude this first blog post, I want to give a shout out to [Simple Programmer](http://simpleprogrammer.com) which helped me kicking off this blog. If anyone is into blogging, I’d recommend checking John Sonmez content, it's really good! Specifically, I took the blogging course. It's a free ~3 week class which helps you kick off your blog and give you insights about getting traffic and marketing yourself. Proudly displaying the badge here since I just completed the class.

<a alt="blogging badge" href="http://simpleprogrammer.com/2015/03/02/my-free-blogging-course-is-getting-unbelievable-results/"><img src="http://simpleprogrammer.com/wp-content/uploads/2015/04/badge.png"></a>

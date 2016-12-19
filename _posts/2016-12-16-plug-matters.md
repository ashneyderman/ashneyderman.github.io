---
layout: default
title: Plug Explained
---

Ever wondered how does plug or phoenix router find your controllers logic? There
is a fair amount of code generation that happens behind the scenes - by way of
Elixir macros - to provide this routing. How exactly does that code look? What
can we infer from the that is generated? I tackle these and related questions
in this blog post.


## Snippets
{% highlight elixir %}
def foo do
  puts 'foo'
end
{% endhighlight %}

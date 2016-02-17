---
layout: post
title: "Evaluating Binding in ERB"
date: 2016-02-16 21:12:42 -0500
comments: true
categories: "Flatiron School"
---

For this blog post, I first set out to understand the use of **binding** in ERB (specifically in the `result` method). Central to that turned out to be a method called `eval`, and this post will mostly discuss the interaction between `eval` and binding, especially in the context of ERB.  

ERB, which stands for Embedded RuBy, is a way to embed ruby code and logic in HTML templates. At one key step in creating our template, we call a method called `result(binding)`. It does some *shenanigans* and gives us back exactly what we want.

![shenanigans](http://i.giphy.com/UNeHvSzLsXwE8.gif)

The main topic of this blog is those shenanigans, but first let's go over ERB.

# ERB

Let's say we have a file called `add_movie.html.erb` that contains the following:

``` HTML
<p>"Added #{movie_name} to your collection."</p>
```

We want to take that template, and insert something useful instead of `#{movie_name}`. First, we need to create a string. Why do we need to create a string? Because that's what ERB.new takes as a parameter. Let's go ahead and create our ERB object.

``` Ruby
def render(file)
  file_name = "views/templates/#{file}.html.erb"
  file_contents = File.read(file_name) # first we create a string
  template = ERB.new(file_contents) # then we create a new ERB object using that string
  binding.pry
  # ... continued below
```

Since we've hit a `binding.pry`, let's take a look at what we've created so far in this method. 

![Looking at template with binding.pry](http://i.imgur.com/E8iuTRr.png)

Most of this will look the same every time you create a new ERB object, except for `@src`, which we'll be looking at in a minute. It's part of the magic. Let's keep going.

``` Ruby
  formatted_file_contents = template.result(binding) # shenanigans
  output_file = "views/output/#{file}.html"
  new_file = File.write(output_file, formatted_file_contents)
  `open -a 'Google Chrome' #{output_file}`
end
```

We call the `result` method on our ERB object, and pass in **binding**. We know that we need to pass in binding so that our template will know what the variable #{movie_name} should be, but not exactly how binding helps. Then ERB does some magic somehow and puts out a string that has had its ruby code magically evaluated. 

![The value of formatted_file_contents](http://i.imgur.com/XxdoZQG.png)

![Shia tells it like it is](http://i.imgur.com/YsbKHg1.gif)

# The Source
To figure out what `result(binding)` is doing, let's go to the source (code).

``` Ruby
               # File erb.rb, line 831
def result(b=TOPLEVEL_BINDING)
  if @safe_level
    proc {
      $SAFE = @safe_level
      eval(@src, b, (@filename || '(erb)'), 0) # <- Right here
    }.call
  else
    eval(@src, b, (@filename || '(erb)'), 0) # <- here too
  end
end
```

There's a lot here, but if we strip it down, it's really only running one command: `eval()`. What is `eval()`?

# Eval and Binding
`eval()` is a method that takes in a string and evaluates that string as ruby code.

``` Ruby
eval("7 * 5") # this will return 35
```
*Note of warning: Do not use `eval()` without adult supervision. It is dangerous.*

The cool thing about `eval()` is that you can pass in an optional parameter to it: **binding**. 

The Ruby docs define **binding** as follows: "Objects of class Binding encapsulate the execution context at some particular place in the code and retain this context for future use." Or as [this blog](https://codequizzes.wordpress.com
/2014/05/18/rubys-binding-class-binding-objects/) put it, "A Binding is a whole scope packaged as an object. The idea is that you can create a Binding to capture the local scope and carry it around." 

So, binding is like a portable scope, and the main place it appears is in `eval()`. The binding parameter will define the scope in which the code you've passed in is executed. 

``` Ruby
def a_totally_different_scope
  str = "Goodbye"
  binding
end

str = "Hello" 
# here, we don't pass in a scope, so it defaults to the current scope
puts eval("str + ' Jeff'")                             #=> "Hello Jeff"

# here, we pass in a scope, so we'll go to that totally_different_scope to
# figure out what str equals
puts eval("str + ' Jeff'", a_totally_different_scope)  #=> "Goodbye Jeff"
```

Let's keep looking at the source code for `result()`.

``` Ruby
               # File erb.rb, line 831
def result(b=TOPLEVEL_BINDING)
  if @safe_level
    proc {
      $SAFE = @safe_level
      eval(@src, b, (@filename || '(erb)'), 0) # <- Right here
    }.call
  else
    eval(@src, b, (@filename || '(erb)'), 0) # <- here too
  end
end
```

We've figured out what b is, but what is that `@src` parameter? It's an instance variable, so we know it must belong to the object calling `result()`, which is template. Let's look at template again.

![Looking at template with binding.pry](http://i.imgur.com/E8iuTRr.png)

Taking a closer look at `@src`, it looks a lot like ruby. In Ruby, the semicolon `;` lets you put more than one command on a single line, so let's split `@src` up into separate lines.

```
"#coding:UTF-8\n
_erbout = ''
_erbout.concat \"<p> \"
_erbout.concat(( \"Added \#{@movie_name} to your collection.\" ).to_s)
_erbout.concat \" </p>\"
_erbout.force_encoding(__ENCODING__)"
```

That looks exactly like ruby, just inside of a string. Exactly what we need for `eval`.

When you call `template.result(binding)` it runs `eval()` on the `@str` instance variable of template. It evaluates that string as ruby code, in the scope of whatever binding was passed in. It creates an empty string _erbout, and then builds back up the string we passed in to ERB.new, without the `<% %>` tags and calling `.to_s` on the code grouped in those tags.

It then passes the result back out as a string which we can then use to create an HTML template.

When it comes down to it, ERB is still magic, but we can at least be in on the trick.



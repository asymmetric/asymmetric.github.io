---
layout: post
title: Try Elixir Online
---


<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

  - [IO in Elixir](#io-in-elixir)
  - [Evaluating code](#evaluating-code)
  - [Elixir syntax in CodeMirror](#elixir-syntax-in-codemirror)
  - [Security](#security)
  - [Domain, hosting and DNS](#domain-hosting-and-dns)
  - [TODOs](#todos)

<!-- markdown-toc end -->

**Update:** I've added an update to the security section.

Today I [released](http://tryelixir.online) something I have been working on for a little while. It's
basically an IEx instance you can access from your browser.

The problem I was trying to solve is one I've had while learning Elixir myself:
I would often be reading Dave Thomas's [Programming Elixir book](https://pragprog.com/book/elixir/programming-elixir)
on my iPad, and at somepoint I'd want to try out some Elixir code. Obviously on
an iPad I can't install the runtime, so I was left with no other option than to
open my laptop. Hopefully [this project](tryelixir.online) will provide an
alternative for people on mobile devices.

So, how was the project implemented? It's actually quite simple. It's a Phoenix
app, without database, using [CodeMirror](https://codemirror.net/) on the
frontend side of things for code editing.

But in order to bring the whole idea to completion, I had to solve the following problems:

* Evaluate the code sent by the user
* Capture and return all the output, including errors
* Find a pretty domain
* Code the frontend
* Choose an online code editor
* Implement a syntax highlighter

Let's look at the first point.

### IO in Elixir

With Elixir's focus on distributed computing, it should come as no surprise that
IO itself can be thought of as a distributed process.

So, how does IO work in Elixir?

Basically every IO function either implicitly or explicitly writes to an IO
device. In the case of
[`IO.inspect/2`](http://elixir-lang.org/docs/stable/elixir/IO.html#inspect/2),
for example, you do it implicitly; its counterpart
[`IO.inspect/3`](http://elixir-lang.org/docs/stable/elixir/IO.html#inspect/3)
receives an additional argument: the device.

Some examples of devices are `:stdio` and `:stderr`, but a device can be any
Elixir process, and can be passed around as a pid (which is a first-class data
type in Elixir) or an atom representing a process.

Another example is `IO.puts/2`. If you look at
[its definition](http://elixir-lang.org/docs/stable/elixir/IO.html#puts/2),
you'll see the first argument it accepts has a default value, `group_leader()`.

[In Erlang](http://erlang.org/doc/man/erlang.html#group_leader-2), a process is
always part of a group, and a group always has a **group leader**. All IO from
the group is sent to the group leader. The group leader is basically the default
IO device, and it can be configured by calling `:erlang.group_leader/2`.

In our case, we want to somehow save all output, and then be able to return it
to the browser. How do we go about doing that?

Turns out Elixir has a very nice module that exposes a string as an IO device.
The module is aptly called
[`StringIO`](http://elixir-lang.org/docs/stable/elixir/StringIO.html), and it
allows us to save and retrieve all data that has been sent to the device.

Combining these two concepts together, the group leader and `StringIO`,
allows us to achieve our goal: redirect all output to a string, store it, and
return it:

``` elixir
{:ok, device} = StringIO.open ""
:erlang.group_leader(device, self)
```

By the way, this stuff is explained quite well on
[this page](http://elixir-lang.org/getting-started/io-and-the-file-system.html)
of the documentation.

### Evaluating code

The next interesting thing happening in this project is it has to evaluate the
code that the user sends at runtime.

Again, Elixir offers a simple solution in the form of
[`Code.eval_string/3`](http://elixir-lang.org/docs/stable/elixir/Code#eval_string/3).
It has only one mandatory argument, the code as a string, and returns a tuple of
the form `{result, bindings}`. We are only interested in the result.

``` elixir
result =
  content
  |> Code.eval_string
  |> elem(0)

IO.inspect result
```

Note that we use `IO.inspect` here, instead of `IO.puts`, as that allows us to
leverage the existing implementations of the
[Inspect protocol](http://elixir-lang.org/docs/stable/elixir/Inspect.html).

This gets us almost all the way there. But what happens if the user's code
causes an exception? Ideally, we should recover from the exception and output
the error message, same as it would happen on IEx.

Let's wrap our code in a `try/rescue` block:

``` elixir
    try do
      result =
        content
        |> Code.eval_string
        |> elem(0)

      IO.inspect result
    rescue
      exception -> IO.inspect Exception.message(exception)
    end

```

And that's all. With our backend code in place, let's now move to the frontend.

### Elixir syntax in CodeMirror

CodeMirror is an online editor for code. It is distributed via NPM in a modular
way, each module in its own file. This meant that I could use bower for the
package, and in my `bower.json` only require the files that I needed, thereby
avoiding bloat and saving a lot of space in the final `application.js`:

``` javascript
{
  "name": "TryElixir",
  "dependencies": {
    "codemirror-elixir": "asymmetric/codemirror-elixir",
    "fetch": "^0.11.0",
    "es6-promise": "^3.2.1",
    "bootstrap": "^3.3.6",
    "hint.css": "^2.2.0"
  },
  "overrides": {
    "codemirror": {
      "main": [
        "lib/codemirror.js",
        "lib/codemirror.css",
        "theme/material.css",
        "addon/mode/simple.js"
      ]
    },
    "bootstrap": {
      "main": [
        "dist/css/bootstrap.css"
      ],
      "dependencies": {}
    }
  }
}
```

There was no syntax highlighter file for Elixir, so I had to
[build one](https://github.com/asymmetric/codemirror-elixir). It's still a work
in progress, and PRs are very welcome. Actually, CodeMirror offes 2 ways of
defining syntaxes (what it calls "modes"): one is RegEx based, and it's very
simple. The other one is to define a proper lexer
([here](https://github.com/codemirror/CodeMirror/blob/master/mode/ruby/ruby.js)
is the one for Ruby for example) and although it's the recommended way, it was
definitely too much work for my use-case. But if you're in that kind of thing,
there's an opportunity for you!

### Security

(**Note:** This section was updated on May 11 2016)

Given that this project allows you to execute arbitrary code, just how much risk is there?

Initially, I thought that Heroku's
[ephemeral filesystem](https://devcenter.heroku.com/articles/dynos#ephemeral-filesystem)
would be enough: users wouldn't be able to permanently affect the application,
even if they somehow managed to run "malicious" code.

And that's true, but I was greatly underestimating the amount of damage that can
be done when you can run arbitrary code.

For example, you can shut down the VM with `:erlang.halt`, or restart it with
`:init.reboot`. Additionally, you can run code that will never stop executing,
while stealing every CPU cycle available.

To address these problems, I've come up with two solutions: a blacklist of
forbidden commands, and performing code evaluation inside an [async Task](http://elixir-lang.org/docs/stable/elixir/Task.html#async/1)
.

Async tasks are particularly interesting. You can invoke one with `Task.async(fn
-> your_function() end)`, and afterwards call `Task.yield(your_task,
your_timeout)` to send it off to do its work.

The key thing here is the timeout: after it expires, the Task is expected to
have a value. If it has one, it's returned; if it doesn't, we can handle that
however we want. In my case, I kill it brutally and just return a message to the
user, informing them that the computation took too long.

### Domain, hosting and DNS

I was very happy to have found a catchy TLD for the site. The registrar is
[Namecheap](https://www.namecheap.com/?aff=98531) (affiliate link) and it only
cost $0.89 for the first year!

The app is hosted on Heroku, and normally what you would do is point
`www.yourdomain.com` to `your-app.herokuapp.com`. But I wanted to get rid of
the `www.` subdomain, and it turns out you can't do that with a lot of DNS
providers. The reason is that
[CNAME records](https://en.wikipedia.org/wiki/CNAME_record) can only be created
for subdomains. Heroku does not provide each app with a fixed IP obviously, so
you can't just create an `A` record and call it a day.

[Some providers](https://devcenter.heroku.com/articles/custom-domains#add-a-custom-root-domain)
though *do* offer so-called `ANAME`/`ALIAS` records, bust most of them cost an
order of magnitude more than the domain itself. That's why I was very surprised
to find out that CloudFlare offers their DNS services *for free*! So I proceeded
to hand over DNS resolution for my domain to them (which by the way took
something like 5 minutes!), configure the
[root CNAME](https://blog.cloudflare.com/introducing-cname-flattening-rfc-compliant-cnames-at-a-domains-root/)
and that was it! Now [the domain](http://tryelixir.online) is availale without
that ugly `www.`!

### TODOs

While I'm pretty happy with the result, there are definitely some ideas that are
worth investigting.

It might be interesting to rewrite the app without Phoenix, as a barebones Plug
app. The backend is extremely simple, the challenging part would be requiring
frontend modules without using Brunch (or having to setup Brunch myself).

Another important thing to do would be to default to HTTPS, maybe using Let's
Encrypt.

And, last but not least, is the tutorial. [Most](http://tour.golang.org/)
[other](http://elm-lang.org/try)
[online](http://tryruby.org/levels/1/challenges/0)
[interpreters](https://tryhaskell.org/) double as intros to the language, with
chapters, lessons and excercises. This is not the main focus of the project (as
I said, I just wanted an easy way to try out code as I was learning), but if
people find it useful, I might implement it.

And I think that's all.

Have fun!

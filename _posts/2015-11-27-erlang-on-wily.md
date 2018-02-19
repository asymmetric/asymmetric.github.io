---
title: Installing Erlang on Ubuntu Wily
---

I'm on Ubuntu Wily, and I was trying to install the `erlang-dev` package to use
[HTTPoison][]. The problem though was that APT wanted to downgrade Erlang to an
older version (16.0), in order to resolve some conflicts. This was obviously
undesirable, so I set out to investigate.

It turns out that, at least for Wily, the entry in
`/etc/apt/sources.list.d/erlang-solutions.list` added by the `erlang-solutions`
package is incorrect.

It reads: `deb http://binaries.erlang-solutions.com/debian squeeze contrib`,
whereas the [official instructions][] say you should have `deb http://packages.erlang-solutions.com/ubuntu wily contrib`.

As soon as I change that, the problem was solved.

### TLDR;

* Uninstall the `erlang-solutions` package
* Add the repo manually in `/etc/apt/sources.list.d/erlang-solutions.list`:

{% highlight bash %}
deb http://packages.erlang-solutions.com/ubuntu wily contrib
{% endhighlight %}

* Install packages

[httpoison]: https://github.com/edgurgel/httpoison
[official instructions]: https://www.erlang-solutions.com/resources/download.html

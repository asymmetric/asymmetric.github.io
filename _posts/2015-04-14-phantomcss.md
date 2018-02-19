---
title: Visual Regression Testing with PhantomCSS
---

PhantomCSS is a fantastic project for doing Visual Regression Testing of your
web app.

It's based on [PhantomJS][], [CasperJS][] and a library written by the same
authors as PhantomCSS, called [Resemble.js][].

### Visual Regression Testing

First of all, what is Visual Regression Testing? Well, according to
[Wikipedia][regression-testing], regression testing is the practice of
detecting unwanted changes in a system (regressions), introduced by other,
possibly unrelated changes.

Applied to CSS (hence the "visual" part) this means that you take a screenshot
of a CSS selector at a moment in time when it looks as expected (your
**baseline**). You then have tests which, on each run, take a screenshot of the
same selector against your app, and compare it to the baseline.

### The idea

The idea is that if you run these type of tests after introducing changes,
you'll be able to catch regressions **automatically**. What's even more
awesome, is that you can very easily test across screen sizes (and if you wish,
also across rendering engines - cfr. [SlimerJS][])

### But aren't integration tests enough?

You might wonder: why add another level of testing? Can't we already test the
structure of the page using tools like `Capybara` or `CasperJS` itself,
asserting on the HTML structure of the page, and possibly even on the CSS
classes attached to the elements?

You could, but you'd be wrong. Asserting on the HTML structure is a gross
misunderstanging of the scope and goal of integration tests; and even if you
assert that the a div has a `.red` class, there's no way to be sure that the
element is actually red.

### The stack

So, with that cleared out of the way, let's proceed to analyze the stack:

* **PhantomJS** is a headless WebKit browser, exposing a JavaScript API you can
  use to drive the browser.
* **CasperJS** is a library built on top of PhantomJS (and SlimerJS), providing
  a much easier to use API, especially useful for writing navigation steps and
  tests. Think [Capybara][] for JavaScript.
* **Resemble.js** is a library for comparing and diffing images. It allows you
  to define a threshold, and differences below that threshold are treated as
  a match. This is useful in many cases, like for when you're running tests on
  different OSes that render fonts slightly differently.

What PhantomCSS does is it glues these tools together, and adds its own API for
taking screenshots and comparing them to the baselines.

### How it works

### Example

Here is a fairly typical PhantomCSS test:

{% highlight javascript linenos=table %}
casper.test.begin("Test the example page", function(test) {
  casper.start();

  casper.then(function() {
    casper.viewportSize(1200, 800);
    casper.open('http://test.example.com');
  }).
  then(function() {
    phantomcss.screenshot('.come-on-my#selector', 'filename');
  });

  casper.then(function() {
    casper.viewportSize(600, 400);
    casper.open('http://test.example.com');
  }).
  then(function() {
    phantomcss.screenshot('.come-on-my#selector', 'filename-small');
  });

  casper.then(function() {
    phantomcss.compareAll();
  });

  casper.run(function() {
    test.done();
  });
});
{% endhighlight %}

In this example, we're loading the page twice, at two different resolutions,
and taking screenshots of the same selector.

At the end of the test, we're comparing the saved screenshots with the
baselines.

But when have the baselines been saved? Time for some explanation.


### The flow

The first time you run a test with PhantomCSS, the library looks into the
baselines directory (**where?**), looking for a baseline that matches the
filename passed to the `screenshot` function. If it doesn't find one, it will
deduce that we're running the test for the first time, and save the screenshot
as a baseline. This also means that it will not run any of the `compare` steps,
as there's nothing to compare yet obviously.

On every subsequent run, the baselines directory will be populated with files
and PhantomCSS will proceed to make the comparisons.

If you want to create the baselines from scratch, you can either delete the
files manually, or configure the command line parameter for the `rebase`
function (more on that [here](#configuring-phantomcss)).

### Structure of a CasperJS test

A full description of the structure of a CasperJS test is out of the scope of
this blog post, but a point should be mentioned:

CasperJS tests are written in an asynchronous way, but avoiding the use of
callbacks (and in fact, that's one of the major advantages of using CasperJS
instead of raw PhantomJS). This means that tests are **declared** using `then()`,
but they're only executed, in the order in which they were declared, with
`run()`.

### Configuring PhantomCSS

PhantomCSS configuration is done through the `init()` function. This is how I
currenlty configure it:

{% highlight javascript linenos=table %}
phantomcss.init({
  libraryRoot:            'vendor/components/phantomcss',
  screenshotRoot:         'test/visual/screenshots/baselines',
  comparisonResultRoot:   'test/visual/screenshots/results',
  failedComparisonsRoot:  'test/visual/screenshots/failures',
  rebase:                  casper.cli.get('rebase'),
  mismatchTolerance:       0.1,
});
{% endhighlight %}

Most options are self-explanatory. The most interesing one is `rebase`, which
is basically saying: when I run the tests using the `--rebase` flag,
**overwrite the baselines**.

### How to run tests

To run a test, you invoke `casperjs test filename.js`.

It's also possible to run tests in more than one file, by using globs:
`casperjs test *.js`.

### Keeping things DRY

If you're splitting your tests across multiple files (which you should probably
do), you'll soon find yourself repeating pieces of code over and over - like
the above mentioned `.init()` call for example.

I solved this by centralizing my configuration to its own file, which i decided
to call `common.js` (great name right?)

{% highlight javascript linenos=table %}
var require = patchRequire(require);
var phantomcss = require('../../vendor/components/phantomcss/phantomcss');

phantomcss.init({
  libraryRoot:            'vendor/components/phantomcss',
  screenshotRoot:         'test/visual/screenshots/baselines',
  comparisonResultRoot:   'test/visual/screenshots/results',
  failedComparisonsRoot:  'test/visual/screenshots/failures',
  rebase:                  casper.cli.get('rebase'),
  mismatchTolerance:       1,
});

exports.phantomcss = phantomcss;

var viewports = { 
    'smartphone-portrait':  { width: 320,  height: 480  },  
    'smartphone-landscape': { width: 480,  height: 320  },  
    'tablet-portrait':      { width: 768,  height: 1024 },
    'tablet-landscape':     { width: 1024, height: 768  },  
    'desktop-standard':     { width: 1280, height: 1024 }
};

exports.set_viewport = function(name) {
  var viewport = viewports[name];

  return casper.viewport(viewport.width, viewport.height);
};

casper.options.viewportSize = { 
  width: viewports['desktop-standard'].width,
  height: viewports['desktop-standard'].height
};
{% endhighlight %}

As you can see, I'm setting some configuration values, defining some helper
functions, and exporting the whole thing using CommonJS's `exports` object.

In your tests you can then do something like

{% highlight javascript linenos=table %}
var common = require('support/common');
common.set_viewport('desktop-standard');
common.phantomcss.screenshot();
{% endhighlight %}

For more info on how to use modules in CasperJS, check [this
page](https://casperjs.readthedocs.org/en/latest/writing_modules.html). There
you'll also find an explanation for the weird two first lines.

[CasperJS]: https://github.com/n1k0/casperjs
[PhantomJS]:http://phantomjs.org/
[Resemble.js]: http://huddle.github.com/Resemble.js/
[Capybara]: https://github.com/jnicklas/capybara
[SlimerJS]: http://slimerjs.org/
[regression-testing]: https://en.wikipedia.org/wiki/Regression_testing

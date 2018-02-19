---
title: Cucumber Notes
---


CUCUMBER
========

Terminology
-----------

* **Feature Step**: the textual descriptions which form the body of a scenario
* **Step Definition**: *matcher* methods, implementation of a step.
  * composed of a *regexp* and a *block*, which receives all of the matched elements as arguments
* **Feature**: indivisible unit of functionality, e.g. an authentication challenge and response user interface. Each Feature is composed of Scenarios
* **Scenario**: a block of statements inside a feature file that describe some behaviour desired or deprecated in the feature


Concepts
--------

* Use **descriptive**, instead of **procedural** feature steps
  * Feature steps are about **what** needs to happen, not **how**
  * *Will this wording need to change if the implementation does?*
    * answer should be no
* The verb used in the step definition doesn't matter:
  * A *Given* feature clause can match a *When* step definition matcher.
* The physical directory structure is flattened by Cucumber
  * any file ending in `.rb` is loaded
  * steps defined in one file can be used in any feature
  * name clashes result in errors
  * the `-r` parameter instead allows to selectively load files/directories
* Immediately implement the new step requirement in the application using the absolute minimum code that will satisfy it.
  * The result is that you will never have untested code in your app
    * Because all code is a product of a test
* Use `module`s to group together calls to the Capybara API
* *Cucumber* uses transactions, and transactions are rolled-back after each `Scenario`

Method
------

* For each feature step
  * Write step definition
  * Run, and watch it fail
  * Write application code that makes it pass
  * Without changing the step's logic, change the "test criteria", to make sure it's passing for the right reason
  * Reset the "test criteria"

A Good Step Definition
----------------------

* The matcher is short.
* The matcher handles both positive and negative (true and false) conditions.
* The matcher has at most two value parameters
* The parameter variables are clearly named
* The body is less than ten lines of code
* The body does not call other steps

Examples
-------

### Feature

{% highlight gherkin %}

Feature: Some terse yet descriptive text of what is desired
In order that some business value is realized
An actor with some explicit system role
Should obtain some beneficial outcome which furthers the goal
To Increase Revenue | Reduce Costs | Protect Revenue  (pick one)

  Scenario: Some determinable business situation
      Given some condition to meet
         And some other condition to meet
       When some action by the actor
         And some other action
         And yet another action
       Then some testable outcome is achieved
         And something else we can check happens too

  Scenario:  A different situation
      ...
{% endhighlight %}

### Step

{% highlight ruby %}

When /statement identifier( not)? expectation "([^\"]+)"/i do |boolean, value|
  actual = expectation( value )
  expected = !boolean
  message = "expectation failed for #{value}"
  assert( actual == expected, message )
end

{% endhighlight %}

### Modules

{% highlight ruby %}
module LoginSteps
  def login(name, password)
    visit('/login')
    fill_in('User name', :with => name)
    fill_in('Password', :with => password)
    click_button('Log in')
  end
end

World(LoginSteps)

When /^he logs in$/ do
  login(@user.name, @user.password)
end

Given /^a logged in user$/ do
  @user = User.create!(:name => 'Aslak', :password => 'xyz')
  login(@user.name, @user.password)
end
{% endhighlight %}

### Classes

{% highlight ruby %}
# features/support/local_env.rb
class LocalHelpers
  def execute( command )
    stderr_file = Tempfile.new( 'script_stdout_stderr' )
    stderr_file.close
    @last_stdout = `#{command} 2> #{stderr_file.path}`
    @last_exit_status = $?.exitstatus
    @last_stderr = IO.read( stderr_file.path )
  end
end

World do
  LocalHelpers.new
end
{% endhighlight %}

Hey guys,

Recently upgraded a Rails 3 project at work to [Capybara](https://github.com/jnicklas/capybara) 2 from 1.1.2.
Ran into a few minor bumps along the way and which I've shared below and hopefully it helps others choosing to upgrade.

**Useful links:**

- [Change log](https://github.com/jnicklas/capybara/blob/master/History.md)
- [Upgrade guide](http://techblog.fundinggates.com/blog/2012/08/capybara-2-0-upgrade-guide/)
- [Why wait_until was removed](http://www.elabs.se/blog/53-why-wait_until-was-removed-from-capybara)
- [Upgrading to Capybara 2 with Rspec 2](https://github.com/rspec/rspec-rails/blob/master/Capybara.md)

Most of this work was backwards compatible. I was able to push multiple fixes, keeping close to master and leaving the gem upgrade as the last commit.

### The 4 biggest changes for us

- [wait_until](https://groups.google.com/forum/?fromgroups#!topic/ruby-capybara/qQYWpQb9FzY)
- All methods which find or manipulate fields or buttons now [ignore them when they are disabled](https://github.com/jnicklas/capybara/commit/dd805d639b62a9bf12773f8e3b9df3c5e5dd8cc2)
- [find](https://github.com/jnicklas/capybara/commit/cc05b1d63b1201027da7b568a7bd0467df9f7e0a) now raises an error if more than one element was found. Meaning every selector should be unique or use first instead (which won't wait like find does)
- [undefined method 'visit'](https://github.com/rspec/rspec-rails/blob/master/Capybara.md#upgrading-to-capybara-20). We had a few Rspec-Capybara tests which had to be moved into spec/features.


### wait_until

We have a large collection of features and a lot of those were using wait_until. Updating all our features to not use it was a large task so to keep moving I decided write a monkey patch which gives us access to a wait_until block.
The idea being that we can than weed our features off the old behaviour one at a time instead of a big bang approach.

```ruby
class Capybara::Session

  def wait_until(timeout = Capybara.default_wait_time)
    Timeout.timeout(timeout) do
      sleep(0.1) until value = yield
      value
    end
  end

end
```

### find vs first

There were a lot of places in the code where we would use find instead of first. Most of these were easy to change but sometimes we needed to wait for the page and first wouldn't cut it. To get around this I created a find_first method which would try to find it and if there were more than one just return the first (exactly what the old find did). This was another patch which was fixed up later with better use of selectors.

```ruby
def find_first(locator)
  element = nil
  begin
    element = find(locator)
  rescue Capybara::Ambiguous
    element = first(locator)
  end
  return element
end
```

**EDIT:** It turns out that you can just use `all(selector).first` because `find` uses `all` under the hood anyway.

### checking for disabled elements

We had a step to ensure the button was disabled which had to be changed. Below is the before and after. If there is a better way of doing this let me know!

```
-Then /^the "([^\"]*)" button is disabled$/ do |title|
-  find_button(title)["disabled"].should_not == nil
-end
+Then /^the delete button is disabled$/ do
+  first(".place_action input[type=\"submit\"][value=\"Delete →\"]")[:disabled].should == "true"
+end
```

### rack_server witin driver

We have a step which was getting rack server mappings and port number from Capybara rack_server accessor. They were changed to the following respectively.

````ruby
#from
Capybara::current_session.driver.rack_server.app.instance_variable_get(:@mapping)
#to
Capybara::current_session.driver.app.instance_variable_get(:@mapping)

#from
page.driver.rack_server.port
#to (Not backwards compatible)
page.server.port
```

The biggest chunk of work was fixing up the hundereds (exaggeration) of ambiguous errors that we were getting. Thankfully each fix was easily backported so I didn't end up with a massive change set locally or sitting on an ever aging branch.

Hope this helps people with their upgrade

-Matt
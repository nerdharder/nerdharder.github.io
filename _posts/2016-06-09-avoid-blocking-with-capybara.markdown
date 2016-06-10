---
layout: post
title:  "Avoid blocking with Capybara"
date:   2016-06-09 10:04:30 -0400
categories: ruby rails
---

[Capybara](https://github.com/jnicklas/capybara) is an awesome tool for testing and screen-scraping. Combined with [poltergeist](https://github.com/teampoltergeist/poltergeist), it provides a very intuitive interface for driving the headless Webkit driver PhantomJS:

```ruby
def sign_in username, password
  fill_in_login_form username, password
  login_succeeded?
end

def fill_in_login_form username, password
  page.visit 'http://www.example.com/'
  page.fill_in 'Username', with: username
  page.fill_in 'Password', with: password
  page.click_button 'Login'
end

def login_succeeded?
  if page.has_text?('Success!')
    true
  elsif page.has_text?('Login failed!')
    false
  else
    raise "Unexpected content on page: #{page.text}"
  end
end
```

The method above will try to sign in to our site and return `true` if login succeeded, or `false` if it failed.

But wait: why does this method seem to take **forever** to run when login fails?

## Synchronization

A big part of Capybara's magic lies in the [synchronize](http://www.rubydoc.info/github/jnicklas/capybara/Capybara/Node/Base:synchronize) method. It uses this method internally to wait until some condition is met. For instance, when you call:

```ruby
  page.fill_in 'Username', with: username
```

Capybara will check whether a field named `Username` appears on the page. If it doesn't, it doesn't fail immediately: instead, it keeps retrying for up to 20 seconds[^1] until it shows up. That way, if the site uses AJAX to load the login form, Capybara will block until the form is in the DOM. No need for `sleep` or explicit retries in your code.

As you may expect, this same method is used when we try to `fill_in` the password field, when we try to `click_button 'Login'`... and when we test for the presence of `Success!` on the page.

So when we encounter a failed login attempt, we end up waiting on `page.has_text?('Success!')` for 20 seconds before proceeding to test the condition `page.has_text?('Login failed!')`. (And if something went wrong and neither message shows up, we end up waiting **another** 20 seconds before giving up and raising an error.)

## Eliminating unnecessary blocking

One way to avoid unnecessarily blocking in this case is to explicitly use the `synchronize` method:

```ruby
def login_succeeded?
  page.document.synchronize do
    if page.has_text?('Success!')
      true
    elsif page.has_text?('Login failed!')
      false
    else
      raise Capybara::ExpectationNotMet
    end
  end
rescue Capybara::ExpectationNotMet
  raise "Unexpected content on page: #{page.text}"
end
```

If the block passed to `synchronize` raises `Capyabara::ExpectationNotMet`, it will be retried until the timeout is hit. Thus, this new version of the code will wait up to 20 seconds for **either** `Success!` or `Login failed!` to appear before proceeding.

(Note: only the outermost `synchronize` block is repeatedly retried. So now that the whole thing is in a `synchronize` block, the individual calls to `has_text?` will immediately return true or false, rather than blocking until the condition is met.)

## Other approaches

Sometimes there's an even simpler way than explicitly calling `synchronize`. For instance, if we were looking for a CSS selector rather than text content on the page:

```ruby
def login_succeeded?
  if page.has_css?('.success_message')
    true
  elsif page.has_css?('.failure_message')
    false
  else
    raise "Unexpected content on page: #{page.text}"
  end
end
```

then we could rewrite this as:

```ruby
def login_succeeded?
  message = page.find('.success_message,.failure_message')
  message[:class].include?('success_message')
rescue Capybara::ElementNotFound
  raise "Unexpected content on page: #{page.text}"
end
```

By passing multiple selectors to `page.find`, we can block until exactly one of those selectors appears on the page.

-----

[^1]: The timeout is configurable by setting `Capybara.default_max_wait_time`

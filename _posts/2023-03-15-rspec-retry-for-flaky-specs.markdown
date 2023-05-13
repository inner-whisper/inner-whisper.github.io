---
layout: single
title:  "On using rspec-retry for flaky specs"
# TODO: set posting datetime
date:   2023-05-14 00:10:00 +0300
categories: testing
tags: ["testing", "rspec", "flaky specs"]
toc: true
---

On any more or less large project we encounter such a problem as flaky-specs. On CI, the test crashes with an error, but when run locally, the test turns out to be green.

There are many aspects of working with such tests and many articles have been written on them. But I would like to touch upon one topic - rerun flaky tests. Is it worth using this approach for any flaky test?

## TLDR

If you find it too difficult to reproduce the conditions of specific `flaky` tests, use the `rspec-retry` gem. If possible, limit this use with gem settings such as `exceptions_to_retry`, and apply it only to certain types of tests. radually reduce the number of retry tests: learn how to reproduce test failure conditions.

## Problem

There are many reasons why running a single test as part of an entire test suite can result in an unstable crash (flaky spec):
1. The order in which the tests are run
2. a certain moment of time (e.g. `on Sunday after 22:00`)
3. Network problems
4. Sorting result of data selection from SQL queries (if no explicit sorting)
5. etc.

Some of these conditions should be reproduced when debugging a flaky test and make the test robust against them. We will talk about it in further articles. In this article I want to discuss the issue of working with tests for which conditions of failure are relatively hard to reproduce (if possible).

## Solution

For the `rspec` testing library - there is a tool [rspec-retry](https://github.com/NoRedInk/rspec-retry), which allows us to restart the test when it fails.

I am not a supporter of restarting any flaky-test, because thereby we lose early signals about those tests which should be processed immediately (it may be time to make more serious decisions about such tests).

I think it's worth solving problems as they arise, rather than sweeping them under the rug. Over time, problems will start to creep out from under that rug, and fixing them will be more difficult. It also increases overall CI build time.

The `rspec-retry` has the ability to limit the list of exceptions according to which it will be raised.

The approach is as follows: if we understand that debugging a flaky-spec (for now) is impossible, we add this particular situation to the "allowlist" for `rspec-retry`.

For example, we see the `js` `system` test crash with an exception `Selenium::WebDriver::Error::StaleElementReferenceError`.

We can handle it with the following setting:

```ruby
# spec/support/rspec_retry.rb

require 'rspec/retry'

RSpec.configure do |config|
  # ...

  config.around :each, :js do |ex|
    # If retry count is not set in an example, this value is used by default.
    # Note that currently this is a 'try' count. If increased from the default of 1, all examples will be retried.
    # We plan to fix this as a breaking change in version 1.0.
    retry_count = 3

    exceptions_to_retry =
      [
        # example of `Selenium::WebDriver::Error::StaleElementReferenceError` - .../-/jobs/560354
        Selenium::WebDriver::Error::StaleElementReferenceError
      ]

    ex.run_with_retry(retry: retry_count, exceptions_to_retry: exceptions_to_retry)
  end
end
```

As new exceptions are identified, our config has grown a bit. The current version of the `rspec-retry` configuration is as follows:

```ruby
# spec/support/rspec_retry.rb

require 'rspec/retry'

RSpec.configure do |config|
  # show retry status in spec process
  config.verbose_retry = true
  # show exception that triggers a retry if verbose_retry is set to true
  config.display_try_failure_messages = true

  config.around :each, :js do |ex|
    # If retry count is not set in an example, this value is used by default.
    # Note that currently this is a 'try' count. If increased from the default of 1, all examples will be retried.
    # We plan to fix this as a breaking change in version 1.0.
    retry_count = 3

    exceptions_to_retry =
      [
        # example of `Timeout::Error` - .../-/jobs/556851
        Timeout::Error,
        # example of `Selenium::WebDriver::Error::UnknownError` - .../-/jobs/556473
        Selenium::WebDriver::Error::UnknownError,
        # example of `Selenium::WebDriver::Error::StaleElementReferenceError` - .../-/jobs/560354
        Selenium::WebDriver::Error::StaleElementReferenceError
      ]

    ex.run_with_retry(retry: retry_count, exceptions_to_retry: exceptions_to_retry)
  end
end
```

Over time, almost every team/specialist expands their competencies. Accordingly, the understanding of "reproducibility of test conditions" may shift, respectively, some of the exceptions will already be handled and can be excluded from the configuration.

## Summary

A possible way to deal with `flaky` specs is to try to restart them in case of a crash. It is desirable to use this approach only when reproducing test crash conditions is difficult for one reason or another. Gem `rspec-retry` can help to implement this approach. The settings of this gem allow you to limit the scope in which it is applied.

Use the `rspec-retry' approach carefully, as it can shoot you in the foot.

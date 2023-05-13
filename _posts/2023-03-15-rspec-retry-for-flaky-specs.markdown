---
layout: single
title:  "On using rspec-retry for flaky specs"
# TODO: set posting datetime
date:   2023-03-12 11:00:00 +0300
categories: testing
tags: ["testing", "rspec", "flaky specs"]
toc: true
---

On any more or less large project we encounter such a problem as flaky-specs. On CI, the test crashes with an error, but when run locally, the test turns out to be green.

There are many aspects of working with such tests and many articles have been written on them. But I would like to touch upon one topic - rerun flaky tests. Is it worth using this approach for any flaky test?

<!-- На любом более-менее крупном проекте мы сталкиваемся с такой проблемой, как flaky-спеки. На CI тест падает с ошибкой, а при воспроизведении локально данный тест уже оказывается зеленым.

Существует множество аспектов работы с такими тестами и по ним написано множество статей. Но я хотел бы затронуть одну тему - перезапуск flaky-тестов. Стоит ли использовать этот подход для любого flaky-теста?
 -->

## TLDR

If you find it too difficult to reproduce the conditions of specific `flaky` tests, use the `rspec-retry` gem. If possible, limit this use with gem settings such as `exceptions_to_retry`, and apply it only to certain types of tests. radually reduce the number of retry tests: learn how to reproduce test failure conditions.

<!-- Если вы считаете, что воспроизведение условий конкретных `flaky`-тестов слишком сложно, то используйте gem `rspec-retry`. По возможности, ограничьте это использование с помощью настроек гема таких, как `exceptions_to_retry`, а также применяйте его только для определенных типов тестов. Постепенно снижайте количество перезапускаемых тестов: учитесь воспроизводить условия падения тестов.
 -->

## Problem

There are many reasons why running a single test as part of an entire test suite can result in an unstable crash (flaky spec):
1. The order in which the tests are run
2. a certain moment of time (e.g. `on Sunday after 22:00`)
3. Network problems
4. Sorting result of data selection from SQL queries (if no explicit sorting)
5. etc.

Some of these conditions should be reproduced when debugging a flaky test and make the test robust against them. We will talk about it in further articles. In this article I want to discuss the issue of working with tests for which conditions of failure are relatively hard to reproduce (if possible).

<!-- 
Существует множество причин, почему запуск одиночного теста в рамках прогона всего набора тестов может закончиться нестабильным падением (flaky):
1. Порядок следования тестов
2. Определенное время
3. Сетевые проблемы
4. Сортировка результата выборки данных из SQL-запросов (при отсутствии явных сортировок)
5. etc...

Часть из этих условий при дебаге flaky-теста стоит воспроизводить и делать тест устойчивым относительно них. Об этом поговорим в дальнейших статьях. В данном материале я хотел обсудить вопрос работы с тестами, для которых условия падения воспроизвести относительно сложно (если вообще возможно). -->

## Solution

For the `rspec` testing library - there is a tool [rspec-retry](https://github.com/NoRedInk/rspec-retry), which allows us to restart the test when it fails.

I am not a supporter of restarting any flaky-test, because thereby we lose early signals about those tests which should be processed immediately (it may be time to make more serious decisions about such tests).

I think it's worth solving problems as they arise, rather than sweeping them under the rug. Over time, problems will start to creep out from under that rug, and fixing them will be more difficult. It also increases overall CI build time.

The `rspec-retry` has the ability to limit the list of exceptions according to which it will be raised.

The approach is as follows: if we understand that debugging a flaky-spec (for now) is impossible, we add this particular situation to the "allowlist" for `rspec-retry`.

For example, we see the `js` `system` test crash with an exception `Selenium::WebDriver::Error::StaleElementReferenceError`.

We can handle it with the following setting:

<!-- Для библиотеки тестирования `rspec` - существует инструмент [rspec-retry](https://github.com/NoRedInk/rspec-retry), который позволяет перезапускать тест при падении.

Я не являюсь сторонником перезапуска любого flakу-теста, т.к. тем самым мы теряем ранние сигналы о тех тестах, которые стоит обрабатывать сразу (возможно, пора принимать более серьезные решения относительно таких тестов).

Я считаю, что стоит решать проблемы по мере их возникновения, а не заметать их под ковер. Со временем проблемы из-под этого ковра начнут вылезать наружу, а их правка будет осложнена. Также это повышает общее время сборки CI.

У `rspec-retry` есть возможность ограничивать список exception-ов, в соответствии с которыми он будет райзиться.

Подход следующий: если понимаем, что дебаг flaky-спека (пока) невозможен, то конкретно эту ситуацию добавляем в `allowlist` для `rspec-retry`.

Например, мы видим падение `js` `system`-теста с exception-ом `Selenium::WebDriver::Error::StaleElementReferenceError`.

Можем обработать его с помощью следующей настройки:
 -->
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

<!-- По мере выявления новых exception-ов, наш конфиг немного разросся. Текущий вариант конфигурации `rspec-retry` выглядит следующим образом:
 -->

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

<!-- Со временем почти каждая команда/специалист расширяет свои компетенции. Соответственно, может сместиться понимание "воспроизводимости условий тестов", соответственно, часть exception-ов уже будет обрабатываться и их можно будет исключить из конфигурации.
 -->
## Summary

A possible way to deal with `flaky` specs is to try to restart them in case of a crash. It is desirable to use this approach only when reproducing test crash conditions is difficult for one reason or another. Gem `rspec-retry` can help to implement this approach. The settings of this gem allow you to limit the scope in which it is applied.

Use the `rspec-retry' approach carefully, as it can shoot you in the foot.

<!-- Возможное направление работы с `flaky`-спеками - попытка их перезапуска в случае падения. Желательно использовать этот подход только в случае, когда воспроизведение условий падения теста осложнено по той или иной причине. Gem `rspec-retry` может помочь в реализации этого подхода. При этом настройки этого гема позволяют ограничить scope, в котором он применяется.

Используйте подход "перезапуск теста в случае падения" аккуратно, т.к. он может выстрелить вам в ногу.
 -->

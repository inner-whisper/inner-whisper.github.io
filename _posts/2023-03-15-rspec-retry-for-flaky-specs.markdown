---
layout: single
title:  "On using rspec-retry for flaky specs"
# TODO: set posting datetime
date:   2023-03-12 11:00:00 +0300
categories: testing
tags: ["testing", "rspec", "flaky specs"]
# toc: true
---

Данная статья посвящена одному аспекту работы с flaky-спеками - перезапуску тест

Существует множество текущих условий тестового окружения, в которых тест может стать нестабильным (flaky):
1. Порядок следования тестов
2. Определенное время
3. Сетевые проблемы
4. Сортировка результата выборки данных из SQL-запросов (при отсутствии явных сортировок)
5. etc...

Часть из этих условий при дебаге flaky-теста стоит воспроизводить и делать тест устойчивым относительно них. Об этом поговорим в дальнейших статьях. В данном материале я хотел обсудить вопрос работы с тестами, для которых условия падения воспроизвести относительно сложно (если вообще возможно).

Для библиотеки тестирования `rspec` - существует инструмент `rspec-retry` (ссылка), который позволяет перезапускать тест при падении.

Я не являюсь сторонником перезапуска любого flakу-теста, т.к. тем самым мы теряем ранние сигналы о тех тестах, которые стоит обрабатывать сразу (возможно, пора принимать более серьезные решения относительно таких тестов).

Я считаю, что стоит решать проблемы по мере их возникновения, а не заметать их под ковер.

У `rspec-retry` есть возможность ограничивать список exception-ов, в соответствии с которыми он будет райзиться.

Подход следующий: если понимаем, что дебаг flaky-спека невозможен, то конкретно эту ситуацию добавляем в `allowlist` для `rspec-retry`.


Пример конфигурации для падения `Selenium::WebDriver::Error::StaleElementReferenceError` на `js`-тестах

Текущий вариант нашего конфига выглядит следующим образом:

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
        # example of `Timeout::Error` - https://git.kubriks.com/itpard/theater_tickets/-/jobs/556851
        Timeout::Error,
        # example of `Selenium::WebDriver::Error::UnknownError` - https://git.kubriks.com/itpard/theater_tickets/-/jobs/556473
        Selenium::WebDriver::Error::UnknownError,
        # example of `Selenium::WebDriver::Error::StaleElementReferenceError` - https://git.kubriks.com/itpard/theater_tickets/-/jobs/560354
        Selenium::WebDriver::Error::StaleElementReferenceError
      ]

    ex.run_with_retry(retry: retry_count, exceptions_to_retry: exceptions_to_retry)
  end
end

```

Как вы можете видеть, постепенно выросло количество Exception-ов, в отношении которых мы запускаем `retry`.

Со временем почти каждая команда/специалист расширяет свои компетенции. Соответственно, может сместиться понимание "воспроизводимости условий тестов", соответственно, часть из этих случаев будет обрабатываться.


Hi there!

This is my first post on this blog. I decided to start to share some of my technical experience here. Well, let's get started!

## Problem

Working on a long-lived Rails project, we can face the situation of frequently using code snippets on the Rails console. 

It can be some sort of context switching (for example, **tenant switching** or **user switching**) or some sort of tooling. It completely depends on your domain model and its technical implementation.

If you use gems like [apartment](https://github.com/influitive/apartment) for tenancy handling, your sessions on `development` or `production` console consist of constant switchings:

```ruby
Aparment::Tenant.switch!('tenant_name_1')

# some tenant_name_1 job

Aparment::Tenant.switch!('tenant_name_2')

# some tenant_name_2 job
```

After hundreds and thousands of times of writing `Apartment::Tenant...bla-bla` phrases, you feel anger. And you want to reduce your keyboard activity.

Such a routine can be simplified with **custom console helpers**.

## Solution

First, create a ruby class with the necessary methods:

```ruby
module Ops
  module CustomRailsConsoleMethods
    def t(tenant_name)
      Aparment::Tenant.switch!(tenant_name)
    end
  end
end
```
 
Second, inform Rails about using this module on console:

```ruby
# config/application.rb
console do
  Rails::ConsoleMethods.prepend(Ops::CustomRailsConsoleMethods)
end
```


Finally, we can simply use a shortened version of the command on the console:

```ruby
t 'tenant_name_1'

# some tenant_name_1 job

t 'tenant_name_2'

# some tenant_name_2 job
```


Yay! Speed up!

## Next steps

### Name helpers in the logic of your domain

Just some ideas about the logic of tenant switching.

Names of methods:
- `t` - switch tenant
- `tn` - print current tenant name
- `tl` - list existing tenant names
- `on_each_tenant` - iterator over all tenants

### Modify helpers according to your needs

The tenant switching method can apply some search patterns to arguments.

If you have the following list of tenant names:
- `big_theatre`
- `maly_theatre`
- `sydney_opera`

You can modify this switching method to use reduced arguments:
- `t 'big'`
- `t 'mal'`
- `t 'syd'`

Looks nice.

### Think about naming conflicts

Sometimes your **custom console helpers** can conflict with other code. So you can save yourself from these conflicts with:


```ruby
# ../ops/custom_rails_console_methods
if defined? t
  raise 'method `t` previously defined in code!'
else
  def t(tenant_name)
    Aparment::Tenant.switch!(tenant_name)
  end
end
```


## Summary

Your development speed is a big value, so such workflow improvements like **custom console helpers** matter. Take care of your time.

---
layout: post
title:  "DSL helpers for your Rails console"
date:   2023-01-02 15:00:00 +0300
categories: rails
tags: ["rails", "console", "development performance"]
---

Hi there!

This is my first post on this blog. I decided to start to share some of my technical experience here. Well, let's go!

Working on a long-lived Rails project, we can face the situation of frequently using code snippets on the Rails console. 

It can be some sort of context switching (for example, **tenant switching** or **user switching**) or some sort of tooling. It completely depends on your domain model and its technical implementation. Such a routine can be simplified with **custom console helpers**.

If you use gems like [apartment](https://github.com/influitive/apartment) for tenancy handling, your work on `development` or `production` console consists of constant switchings:

```ruby
Aparment::Tenant.switch!('tenant_name_1')

# some tenant_name_1 job

Aparment::Tenant.switch!('tenant_name_2')

# some tenant_name_2 job
```

After hundreds and thousands of times of writing `Apartment::Tenant...bla-bla` phrases, you feel anger. And you want to reduce your keyboard activity.

So, let's simplify this work with **custom console helpers**. 

First, create a ruby class with the necessary helpers:

```ruby
module Ops
  module CustomRailsConsoleMethods
    def t(tenant_name)
      Aparment::Tenant.switch!(tenant_name)
    end
  end
end
```
 
Also, we need to inform Rails about using this module on console:

```ruby
# config/application.rb
console do
  Rails::ConsoleMethods.prepend(Ops::CustomRailsConsoleMethods)
end
```


Now we can simply use:

```ruby
t 'tenant_name_1'

# some tenant_name_1 job

t 'tenant_name_2'

# some tenant_name_2 job
```


Yay! We got a boost!

## Next steps

### Name helpers in logic of your domain

Just some ideas from logic of tenant switching.

Names of methods:
- `t` - switch tenant
- `tn` - print current tenant name
- `tl` - list existed tenant names
- `on_each_tenant` - iterator over all tenants

### Modify helpers according with your needs

Tenant switching method can apply some search patterns on argument.

If your have following list of tenant names:
- `big_theatre`
- `maly_theatre`
- `sydney_opera`

You can modify this switching method to use reduced arguments:
- `t 'big'`
- `t 'mal'`
- `t 'syd'`

Looks nice.

### Think about naming conflicts

Sometimes your **custom console helpers** can conflict with other code. So you can safe your from these conflicts with:


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

Your development speed is a big value, so such worflow improvements like a **custom console helpers** matter. Spend your time wisely.

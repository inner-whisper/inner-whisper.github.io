---
layout: post
title:  "DSL helpers for your Rails console"
date:   2022-12-20 16:03:44 +0300
categories: rails
tags: ["rails", "console", "development performance"]
---

Hi there!

This is my first technical post on this blog.

Working on a long-lived Rails project, we can face the situation of frequently using code snippets in Rails console. 

It can be some sort of context switching (for example, **tenant switching** or **user switching**). It completely depends on your domain model and its technical implementation. Such routine can be simplified with **custom console helpers**.

If you use gem like [apartment](https://github.com/influitive/apartment) for tenancy, your work in `development` or `production` console consists of such switching actions:

```ruby
Aparment::Tenant.switch!('tenant_name_1')

# some tenant_name_1 job

Aparment::Tenant.switch!('tenant_name_2')

# some tenant_name_2 job
```

After hundreds and thousands times of writing `Apartment::Tenant...bla-bla` phrases, you feels anger.

So, lets simplify this work with **custom console helpers**:

```ruby
module Ops
  module CustomRailsConsoleMethods
    def t(tenant_name)
      Aparment::Tenant.switch!(tenant_name)
    end
  end
end
```
 
Also we need to inform Rails about using this module in console:

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


## Summary

Ключевые пункты:
- ваши хелперы должны соответствовать доменной логике вашего приложения. Идеи для хелперов переключения тенантов:
  - `t`, `tn`, `tl` и др.
- модифицируйте и усложняйте их под потребности
- подумайте о защите от переписывания уже существующих методов (`if defined?`)


You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

Jekyll requires blog post files to be named according to the following format:

`YEAR-MONTH-DAY-title.MARKUP`

Where `YEAR` is a four-digit number, `MONTH` and `DAY` are both two-digit numbers, and `MARKUP` is the file extension representing the format used in the file. After that, include the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

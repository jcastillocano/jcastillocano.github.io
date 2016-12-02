---
layout: post
title:  "Coobbook design for bakery + deploy in AWS"
date:   2016-11-30 21:18:39 +0000
categories: chef cookbook AWS AMI
---

## Problem

We've been working with [Chef][chef] in AWS for a while, creating CI/CD
pipelines for both my cookbooks and chef repositories (We use chef-solo).
Using Jenkins we could orchestrate lint, unit and integration tests using
EC2 instances. We are not gonna talk about tools rather than processes in
this post, so sorry to disappoint you if you were expecting something else.
In order to speed up our restack process, we pre-bake AWS AMIs per role, in
that way only configuration and secrets are applied during deploys.

Since this approach works so far, we have been dealing with the lack of
cookbooks with bakery and deploy recipes splitted out-of-the-box. How
many of you have dealt with a cookbook only with default.rb, where
package installation and configuration are both at the same level? It
happens a lot, mostly in small, simple cookbooks (don't get me wrong, it
is not a bad pattern), but this kind of cookbooks doesn't fit in our
solution: we try to split as much as possible installation and
configuration in our cookbooks, in order to apply them separately. As an
example, during AMI bakery, you want to install all packages (_nginx_,
_rsyslog_, _logrotate_), directories, default config, etc, but only when
you deploy you configure your _vhost_, creates _logrotate_ config, etc.
To achieve this, we need recipes for installation and recipes for
configuration, or at least flags to enable/disable this features.

## Workaround: flags

You cannot go around and update all the cookbooks in Chef Supermarket,
creating recipes for installation and configuration (if they don't
already have them), we tried with some of them we found on the way and,
even if some of them were updated, most of them were just forgotten.
Instead, what we did was to take advantage of *flags* (or *feature
flags*), where you can enable or disable some features based on
parameters. In that way, you can disable some packages installation, or
skip configuration until next chef execution. That works so far if the
cookbook supports feature flags (most of them do), so we mimic this
approach since the beginning.

However, problems came up testing. We use [kitchen][kitchen] to run our
integration tests in AWS, but in chef you cannot run on recipe twice
during the same execution. Searching for solutions, we found this
implementation of [test-kitchen][test-kitchen] that supports multi step
converge, really helpful if you want to run twice your recipe (thanks
@wjordan!). Using this version we tweaked all our chef repos to have two
steps per _test suite_ (see *.kitchen.yml* below)

{% highlight ruby %}
suites:
  - name: web
    steps:
      - run_list:
        - role[web]
        attributes:
          nginx:
            run_installation: true
      - run_list:
        - role[web]
        attributes:
          nginx:
            run_config: true
{% endhighlight %}

This piece of code will call _web role_ twice but in different chef
executions, so arguments won't mess each other. In the end, we had what
we needed: a way to test our bakery and deploy code without modifying
too much any cookbook. Nevertheless, how does it work under the hook?
Simple: _default.rb_.

## Make it simple is a must

This is our _recipes/default.rb_

{% highlight ruby %}
# Cookbook Name:: web
# Recipe:: default
#
# Copyright 2016, Juan Carlos Castillo
#
# All rights reserved - Do Not Redistribute
#
# Installs nginx package and configures nginx.conf

include_recipe 'web::package' if node['nginx']['run_installation']
include_recipe 'web::config' if node['nginx']['run_config']
{% endhighlight %}

In that way you don't need to worry which recipe installs or which
recipe configures your vhost, just include default always and
enable/disable features. It sounds redundant, but it makes things a lot
easier for both testing and deploys. Following this standard we manage
to have all our main roles using a few cookbook recipes, but the
possibilities are infinite.

# Summary

We manage to simplify our bakery and deploy process reusing our
cookbooks so that can be tested and reused over our pipelines (testing
and deploys). Making this little change in our cookbooks we manage to
reduce the size and complexity of our roles, and we made our test
process more robust and stable (patterns, patterns everywhere!).

We found this issue interesting, but we know there is a long way to go.
If you want to help us walking this road, please comment or send us an
email, we'd like to know about other options!

[chef]: https://www.chef.io
[kitchen]: https://kitchen.ci
[test-kitchen]: https://github.com/wjordan/test-kitchen/tree/multi-step-converge

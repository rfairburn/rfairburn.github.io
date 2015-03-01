---
layout: post
title: "Beyond Grains: Roles with an External Pillar"
description: "Using an ext_pillar to overcome the limitations of roles in grains"
category: Configuration Management
tags: ["SaltStack"]
---
{% include JB/setup %}
In SaltStack, managing system roles in grains has a number of limitations.  What other solutions are there?  This is a follow up to [The Grains Conundrum](/configuration%20management/2014/08/25/the-grains-conundrum/).

### Background

If you have already read [The Grains Conundrum](/configuration%20management/2014/08/25/the-grains-conundrum/), then you know that I have already talked about a number of limitations in usings grains for role management.  There are also additional considerations that I have not mentioned previously.  Including new and old issues we get a list that looks like this:

- Grains live on the minion
- A change to a role grain requires multiple state executions
- The previous model allows all minions to see all other minion roles
- A change in a role from a minion side could potentially expose unwanted private pillar data

There may be additional considerations, but these alone are reason enough to look for a different and better way.  I would not want to lose any flexibility in a change.  The ability to inherit roles from categories and the ability for a category to inherit other catories are both too integral for me to pass up.  For this reason, I chose not to rely on nodegroups or fancy globs in my top file.  [External Pillars](http://docs.saltstack.com/en/latest/topics/development/external_pillars.html) seemed to be the next logical choice.

### The External Pillar

In Salt, the [Pillar](http://docs.saltstack.com/en/latest/topics/tutorials/pillar.html) is really just a large Python dictionary.  However it can often store sensitive data that you may not want exposed to machines other than the intended targets.  Roles and categories should not be excluded from this.  While pillars do support templating to customize what a particular minion can see, [External Pillars](http://docs.saltstack.com/en/latest/topics/development/external_pillars.html) allow for much more customization.  While the example I use below will use a yaml file that is very similar to what you saw in [The Grains Conundrum](/configuration%20management/2014/08/25/the-grains-conundrum/), external Pillars are actually written in pure python and would therefore allow many choices for backend data sources.  This would include but not be limited to:

- a yaml or json file
- a nosql data source (such as MongoDB)
- a sql database such as MySQL or PostgreSQL
- etcd
- many many more

There are also a number of [built-in ext_pillars](http://docs.saltstack.com/en/latest/ref/pillar/all/index.html#all-salt-pillars) that can be utilized as well, but they do not include the manipulation of the pillar data that I desired for our role mechanics.  The proof-of-concept for my ext_pillar can be found on [GitHub](https://github.com/rfairburn/salt-roles-pillar).

### The Implementation

Custom ext_pillars need to live in the pillar subfolder under your [extension_modules](http://docs.saltstack.com/en/latest/ref/configuration/master.html#extension-modules) folder on your salt master.  In my configuration, that means I place it at /srv/modules/pillar/roles.py

As with the grains formula, we start with a yaml file that looks like this:

{% highlight yaml %}

categories:
# Used if a system is not indicated below
  default:
    categories:
      - nrpe
      - sudoers
# Service category designations
  dnsmasq:
    roles:
      - dnsmasq
      - hosts
  jenkins_deploy_target:
    sudoers.included:
      - jenkins
    categories:
      - sudoers
  nrpe:
    roles:
      - nagios.nrpe
      - files.nagios_plugins
  sudoers:
    roles:
      - sudoers
      - sudoers.included
      - files.sudoers_cleanup
    sudoers.included:
      - cloud-init
# Environment category designations
# Used to add designations and properly manage sudoers
# Potentially used to manage unique 'roles' per environment
# (dev,stage,test,uat,prod,support, etc)
#
# Nonprod designations
  non_prod_server:
    sudoers.included:
      - corp-non-prod
    categories:
      - sudoers
  demo_server:
    categories:
      - non_prod_server
  stage_server:
    categories:
      - non_prod_server
  test_server:
    categories:
      - non_prod_server
  uat_server:
    categories:
      - non_prod_server
# Prod designations
  prod_server:
    sudoers.included:
      - corp-prod
    categories:
      - sudoers
  support_server:
    categories:
      - prod_server
systems:
# Minion system. The line below should match grains['id']
  salt-minion:
    roles:
      - nagios.server
    categories:
      - default
      - prod_server
      - jenkins_deploy_target
# Another minion system example
  salt-master:
    roles:
      - files.salt_master_files
    categories:
      - default
      - support_server
      - dnsmasq
# Prune a role inherited from a category.  
# Good for "I want everything but X from this category"
    prune_roles:
      - hosts

{% endhighlight %}

The sections systems and categories are required and a default category is also expected (to match if nothing else does).  One thing that you will see added here that was not in the Grains Formula is an option to prune.  Any section inside a host or category can be pruned other than other categories (I was lazy with the implementation and did not want to walk the whole inheritence to ensure that everything inherited from a category was also pruned).  This allows for a situation such as follows:

- 59 of 60 prod servers get the same sudoers setup but the 60th one gets something else.

In that type of situation, I can just prune the sudoers.included item from the list that does not apply on the 60th server but can otherwise designate it with the prod_server category that makes sense for everything else.  Here is what that might look like:

{% highlight yaml %}

systems:
  odball-prod-server:
    roles:
      - nagios.server
    categories:
      - prod_server
    sudoers.included:
      - extra-secret-users
    # Remove default prod sudoers
    prune_sudoers.included:
      - corp-prod

{% endhighlight %}

### Other Config Files

In the master config, you will need to configure the external pillar in your master config.  I have the following in my /etc/salt/master.d/pillar.conf

{% highlight yaml %}

pillar_roots:
  base:
    - /srv/pillar

ext_pillar:
  - roles: /etc/salt/roles.yaml

{% endhighlight %}

The other place we need to have configured differently is our top file.  Previously we were matching grains there for our loop, but we can do the same thing with the pillar data from our ext_pillar.  It would look like this:

{% highlight jinja %}
{% raw %}

base:
  '*':
    - salt.minion
{% if 'roles' in pillar %}
  {% for role in salt['pillar.get']('roles', []) %}
    - {{ role }}
  {% endfor %}
{% endif %}

{% endraw %}
{% endhighlight %}

This time we do not have to worry aboout special reactors to run another highstate, or orchestrations that apply grains and then run a highstate.  All of these things 'just work' under this model.  One limitation currently exists however.  If you wish to use ext_pillar data in your pillar sls files, that does not currently work.  Salt 2015.2 is just around the corner though and adds a configuration option called [ext_pillar_first](http://docs.saltstack.com/en/latest/ref/configuration/master.html#ext-pillar-first) that appears like it will solve for this limitation.

### Questions/Comments

If you missed it above, please check [GitHub](https://github.com/rfairburn/salt-roles-pillar) for the latest copy of the roles ext_pillar.  As always if you have any questions or comments, please do not hesitate to ask, and I will do my best to provide what assistance I can.  I hope you have found this useful, and I actually successfully switched my test environment to it this weekend with a plan for full implementation in the near future.  I will post back with any changes that may come from that.

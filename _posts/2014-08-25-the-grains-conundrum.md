---
layout: post
title: "The Grains Conundrum"
description: "Thoughts on managing custom Grains in SaltStack"
category: Configuration Management
tags: ["SaltStack"]
---
{% include JB/setup %}
[SaltStack](http://saltstack.com) provides an excellent mechanism for obtaining pieces of information from managed minions called Grains.  Salt even supports generating custom Grains that you define, allowing you to specify anything you want about the minion.  It could be roles, categories, environments or anything else you could imagine.  The problem is that these Grains are actually defined on the minion, and I would prefer to manage all aspects of my environment directly from the master.  Read on to see how I tackled the problem.

### Background

The [SaltStack documentation](http://docs.saltstack.com/en/latest/) provides excellent reference about how you can use [Grains](http://docs.saltstack.com/en/latest/topics/targeting/grains.html).  Here is a small example of potential usage.

If I wanted to lookup the cpu flags that were available on a particular minion named salt-minion, for example, I could run something like this on my Salt master:

{% highlight console %}

# salt 'salt-minion' grains.item cpu_flags
salt-minion:
  cpu_flags: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx rdtscp lm constant_tsc rep_good nopl xtopology eagerfpu pni pclmulqdq ssse3 cx16 pcid sse4_1 sse4_2 x2apic popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm xsaveopt fsgsbase smep erms

{% endhighlight %}

I could even define custom Grains in /etc/salt/grains on a particular minion and and they would also be available for consumption as well.  The problem with this is that one typically implements SaltStack so that logging into a particular minion to make configuration changes is a thing of the past.

So maybe we could manage that grains file directly from our salt master...


### Enter the salt-grains-formula

[SaltStack Formulas](http://docs.saltstack.com/en/latest/topics/development/conventions/formulas.html) allow you to have your state information pre-defined in a way that makes them easy to apply to a minion.  The [salt-grains-formula](https://github.com/rfairburn/salt-grains-formula) is what I used to manage minion grains and help solve the problem above.

So what does the formula actually do, you ask?  If you looked at the link to the Grains documentation above, you might have noticed that the /etc/salt/grains file is just a YAML file that Salt loads as a Python dictionary.  This formula takes information in a Salt [Pillar](http://docs.saltstack.com/en/latest/topics/tutorials/pillar.html) (another YAML file) and applies the pertinent pieces to the minion.

The idea started with the concept of roles mentioned [here](http://docs.saltstack.com/en/latest/topics/targeting/grains.html#grains-in-etc-salt-grains).  I felt it was good to start with roles mapping to particular states what were assigned to a machine, but I also wanted a way to group these into some type of logical categories, and also to have default settings that were always applied if nothing was explicitly defined.

What I ended up with is a Pillar like this [example](https://github.com/rfairburn/salt-grains-formula/blob/master/pillar.example) where I defined categories that had groupings of roles.  Additionally I decided it made sense that categories could inherit roles from other categories as well.

So to give an example of how this all fits together, let us start with the server definition for the minion with the id of 'salt-minion' from the linked pillar example.  I know, very creative...  It looks like this


{% highlight yaml %}

salt-minion:
  roles:
    - nagios.server
  categories:
    - default
    - prod_server
    - jenkins_deploy_target

{% endhighlight %}

We see an individual role of 'nagios.server', so that will be applied directly to the /etc/salt/grains file of the salt-minion.  We also see 3 categories.  Let us take a quick look at what things are defined in those.


{% highlight yaml %}

default:
  categories:
    - nrpe
    - sudoers
prod_server:
  sudoers.included:
    - corp-prod
jenkins_deploy_target:
  sudoers.included:
    - jenkins
  categories:
    - sudoers

{% endhighlight %}

Right away we get to see how categories can apply other categories.  We do not see any new roles that get applied, but several sudoers.included roles do.  I will not go into the sudoers.included in this article other than to show how it might look in the final file.  I will try and include the sudoers magic in another article.  In any case, based upon the example Pillar and what we have seen so far, the final /etc/salt/grains file would look much like this:

{% highlight yaml %}

categories:
  - default
  - nrpe
  - sudoers
  - non_prod_server
  - jenkins_deploy_target
roles:
  - nagios.server
  - nagios.nrpe
  - files.nagios_plugins
  - sudoers
  - sudoers.included
  - files.sudoers_cleanup
sudoers.included:
  - cloud-init
  - corp-non-prod
  - jenkins

{% endhighlight %}

### The big picture

Now we have in place our Grains, but what do we do with them?  We need to at the very least have the roles applied as states to the minions.  Here a little [Jinja](http://docs.saltstack.com/en/latest/ref/renderers/all/salt.renderers.jinja.html) in our [top.sls](http://docs.saltstack.com/en/latest/ref/states/top.html) will do the magic:


{% highlight jinja %}
{% raw %}
base:
  '*':
    - salt.minion
    - grains
{% if 'roles' in grains %}
  {% for role in salt['grains.get']('roles', []) %}
    - {{ role }}
  {% endfor %}
{% endif %}
{% endraw %}
{% endhighlight %}

This almost gets us to where we want to be.  The problem with managing the Grains via a state and then applying the other states on the minion from the generate list is that we have to run a [highstate](http://docs.saltstack.com/en/latest/ref/states/highstate.html) twice in order to apply all configs properly.  This can also lead to some other weird dependency inconsistencies as well.  Luckily Salt already had the answer: [Orchestration](http://salt.readthedocs.org/en/latest/topics/tutorials/states_pt5.html).

I had the Orchestration run in such a way that the Grains state was triggered first and completed prior to applying a full highstate.  The pertinent parts of my Orchestration look like this:


{% highlight jinja %}
{% raw %}
refresh_grains:
  salt.state:
    - tgt: '{{ target }}'
    - tgt_type: '{{ tgt_type }}'
    - sls:
      - grains

run_highstate:
  salt.state:
    - tgt: '{{ target }}'
    - tgt_type: '{{ tgt_type }}'
    - highstate: True
{% endraw %}
{% endhighlight %}

The target and tgt_type are pieces of information that are parsed through a macro I wrote that allows for the Orchestration run to target different groups of servers.

Now we finally have a complete solution that works and is fairly flexible.

### More

I may talk more on this later or go into greater detail.  Please post a comment if you have any questions, and I will do my best to answer them and/or update the post with pertinent information.

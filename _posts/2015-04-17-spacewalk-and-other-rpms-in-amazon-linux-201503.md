---
layout: post
title: "Spacewalk and Other RPMS in Amazon Linux 2015.03"
description: ""
category: Configuration Management 
tags: ['Spacewalk', 'Amazon Linux', 'Python', 'RPMs', 'SaltStack']
---
{% include JB/setup %}
If you use Amazon Linux and have looked at the latest 2015.03 offering, you may have noticed that Amazon updated the system Python version to 2.7.  While they still provide fairly decent support for Python 2.6, this change caused several unforeseen headaches when deploying 2015.03 at work -- particularly surrounding [Spacewalk](http://spacewalk.redhat.com).  Read on to see how I addressed these issues...

### The Good

Those of you who have read my article [SaltStack: GMP and PyCrypto on Amazon Linux](/configuration%20management/2014/11/20/saltstack-gmp-and-pycrypto-on-amazon-linux/) should already be aware that I have utilized custom RPMs in the past to get around shortcomings of the supplied system packages.

What I did not explain in that article is that the system-supplied paramiko python package also used deprecated features in pycrypto causing messages similar to what you see in that article.  Amazon Linux 2015.03 supplies updated paramiko packages for both Python 2.6 and Python 2.7, so I no longer need to build my own.  That is excellent.  Additionally they have alternatives in place so that you can still default to Python 2.6 for most things if-needed.

I have also learned a bit about packaging since that article and have some better dependency integration and even found some things that were broken with the implementation as documented in that article.  I will share those updates below as well.

### The Bad

While many packages have been updated on Amazon Linux including Python, PHP, Apache, the kernel, and more, it seems that we are still stuck with an old version of libgmp.  This means that we still have to build PyCrypto ourselves.  We probably should also build it for both Python 2.6 and Python 2.7 given the state of Python on the system.

Additionally, I found I was no longer able to utilize the RHEL6/CentOS6 version of Spacewalk with Amazon Linux.  This really surprised me given that the OS still comes pre-configured with the EPEL6 repository.  The reason was simple enough with some further digging.

### The Ugly

With the default system Python now sitting at version 2.7 in Amazon Linux, system applications such as yum also utilize Python 2.7.  It seems obvious as stated, but for compatibility-sake, I had hoped it was not so.  In fact they are hard-coded that way even with the shebang showing python2.7 (#!/usr/bin/python27).  No wonder things such as the yum-rhn-plugin had no chance of working.  Spacewalk has no chance to run with the EL6 packages!

Additionally, since the Python alternative defaults to 2.7, the SaltStack bootstrap script utilized by salt-cloud also fails to bootstrap minions.  Luckily a one-line change setting the alternative immediately prior to starting the minion daemon solves this issue.

What about Spacewalk though?

### The Fix

As I saw it the only chance of seeing Spacewalk work for me again was to build my own RPMs.  We are running Spacewalk 2.0 on our clients, which is not the most recent version, but was meeting all of our needs.  What I did was downloaded and install the source RPMs for [Spacewalk 2.0](http://yum.spacewalkproject.org/2.0-client/RHEL/6/source/) and installed the sources into my build environment.

One of the first thing I realized was a number of Python 2.6 dependencies that were not supplied by Amazon Linux and actually came from EPEL.  I would need to build those first.  I will not go into too much detail about the specifics, but have supplied the .spec files on [GitHub](https://github.com/rfairburn/amazon-linux-rpms) to allow you to build your own.

NOTE: I had a gotcha when I was building these packages.  It was likely due to my alternative being set to Python 2.6 instead of Python 2.7 to make SaltStack work, but I will share in any case.  I had to build with the command as follows:

{% highlight bash %}

rpmbuild -ba --define '__python /usr/bin/python27' <specfile>

{% endhighlight %}

#### Python Dependencies

* [python-dmidecode](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/python-dmidecode.spec)
* [python27-ethtool](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/python27-ethtool.spec)
* [python27-gudev](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/python27-gudev.spec)
* [python27-hwdata](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/python27-hwdata.spec)

Any patches needed by these are also supplied.  You will need to download the reference sources yourself, but the locations should be included in the specfile or at worst obtained at the [EPEL6 SRPM Repo](https://dl.fedoraproject.org/pub/epel/6/SRPMS/).

#### Spacewalk

Here are the specs for Spacewalk tested against Amazon Linux 2015.03.  I have NOT tested these against any other operating system, and by the very nature of the changes I made, they will NO LONGER build on any other system.  I could have worked to keep cross-compatibility intact, but I was short on time and did not have additional build environments handy.


* [osad](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/osad.spec)
* [rhn-client-tools](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/rhn-client-tools.spec)
* [rhn-custom-info](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/rhn-custom-info.spec)
* [rhn-virtualization](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/rhn-virtualization.spec)
* [rhncfg](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/rhncfg.spec)
* [rhnlib](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/rhnlib.spec)
* [rhnmd](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/rhnmd.spec)
* [rhnsd](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/rhnsd.spec)
* [spacewalk-abrt](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/spacewalk-abrt.spec)
* [spacewalk-cert-tools](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/spacewalk-cert-tools.spec)
* [spacewalk-koan](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/spacewalk-koan.spec)
* [spacewalk-oscap](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/spacewalk-oscap.spec)
* [yum-rhn-plugin](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/yum-rhn-plugin.spec)

### Bonus

The [amazon-linux-rpms repo](https://github.com/rfairburn/amazon-linux-rpms) also contains my latest fpm build script for [gmp6](https://github.com/rfairburn/amazon-linux-rpms/blob/master/fpmbuild/gmp6.sh), which actually executes the ldconfig and adds the proper library paths to a file in /etc/ld.so.conf.d.  This ensures that the compiled PyCrypto package can actually use the gmp6 library when it needs to without intervention.

Speaking of [PyCrypto](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/python-crypto.spec), that should be the latest [spec](https://github.com/rfairburn/amazon-linux-rpms/blob/master/rpmbuild/SPECS/python-crypto.spec).  I strongly recommend using these over the previously mentioned ones in [SaltStack: GMP and PyCrypto on Amazon Linux](/configuration%20management/2014/11/20/saltstack-gmp-and-pycrypto-on-amazon-linux/).

Questions? Comments?  Feel free to leave them below as always!


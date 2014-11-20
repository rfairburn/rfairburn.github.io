---
layout: post
title: "SaltStack: GMP and PyCrypto on Amazon Linux"
description: "How to stop those annoying warnings about insecure GMP on your PyCrypto when using SaltStack (or other Python applications)"
category: Configuration Management 
tags: ["SaltStack", "Amazon Linux"]
---
{% include JB/setup %}
So you may have noticed recently that Salt (or other Python apps that utilize PyCrypto such as Ansible) has been giving some warnings that look like this when running on Amazon Linux:

{% highlight console %}

/usr/lib64/python2.6/site-packages/Crypto/Util/number.py:57: PowmInsecureWarning: Not using mpz_powm_sec.  You should rebuild using libgmp >= 5 to avoid timing attack vulnerability.
  _warn("Not using mpz_powm_sec.  You should rebuild using libgmp >= 5 to avoid timing attack vulnerability.", PowmInsecureWarning)
/usr/lib64/python2.6/site-packages/Crypto/Util/randpool.py:40: RandomPool_DeprecationWarning: This application uses RandomPool, which is BROKEN in older releases.  See http://www.pycrypto.org/randpool-broken
  RandomPool_DeprecationWarning)

{% endhighlight %}

These mainly revolve around libgmp being super old in Amazon linux and part of saltstack using PyCrypto which then depends upon libgmp.  This article details the process used to make RPM packages that solve for this issue.

The SaltStack specifics of the issue are documented here:

 * [https://github.com/saltstack/salt/issues/15548](https://github.com/saltstack/salt/issues/15548)

I used this information as a basis to get started but wanted to use package management for gmp and python-crypto to ease deployment to all of my servers.

Many other applications also depend upon the system gmp through coreutils, and I didn't want to mess with that whole chain, so these instructions will outline the steps to install a parallel version of gmp (6.0.0a in the case of this article) in another location (/opt/gmp6 in this case).

### Installing FPM

FPM is an amazing piece of software that allows you to turn basically anything into an RPM without having to go through the headaches of building your own specfile and fighting rpmbuild.  This is particularly useful in getting the gmp6 portion of this all working.  The FPM project and its source code can be found here:


 * [https://github.com/jordansissel/fpm](https://github.com/jordansissel/fpm)

I followed the following article to get things installed:

 * [https://github.com/jordansissel/fpm/blob/master/README.md](https://github.com/jordansissel/fpm/blob/master/README.md)

To summarize:

{% highlight console %}

yum install ruby-devel gcc
gem install fpm

{% endhighlight %}

Now we have fpm in /usr/local/bin to work with.  Great!

### Building GMP

Now that we have FPM installed to make packages easily, We'll need to build GMP and then packaged the built root with FPM.  I used a modified version of these steps:

 * [https://www.digitalocean.com/community/tutorials/how-to-use-fpm-to-easily-create-packages-in-multiple-formats](https://www.digitalocean.com/community/tutorials/how-to-use-fpm-to-easily-create-packages-in-multiple-formats)

Here are the actual steps step-by-step with comments:

{% highlight console %}

# Get SRPM for gmp
sudo get_reference_source -p gmp
# Build dependencies so we have them to compile (if needed)
sudo yum -y install rpm-build
sudo yum-builddep /usr/src/srpm/debug/gmp-4.3.2-1.11.amzn1.src.rpm
# Obtain source to build and build it:
wget https://ftp.gnu.org/gnu/gmp/gmp-6.0.0a.tar.bz2
tar -xjvf gmp-6.0.0a.tar.bz2
cd gmp-6.0.0
./configure --prefix=/opt/gmp6
mkdir /tmp/gmp
# Install into temporary directory for packaging
sudo make DESTDIR=/tmp/gmp install
# You do test your binaries, right?
make test
# Build the FPM package and say that it provides libgmp.so.10()(64bit) (Required for python-crypto below)
/usr/local/bin/fpm -s dir -t rpm -C /tmp/gmp --name gmp6 --version 6.0.0a --iteration 1 --description "libgpm 6" --provides 'libgmp.so.10()(64bit)' .

{% endhighlight %}

So following all of these steps you should have an RPM with the following name right in your current working directory:

{% highlight console %}

gmp6-6.0.0a-1.x86_64.rpm

{% endhighlight %}

### Python Crypto (PyCrypto)

We'll install the rpm from above as the first step of building our new python crypto library and do all of the following to get ready to build a new python-crypto rpm:

{% highlight console %}

# Additional dependencies to make latest PyCrypto
sudo rpm -Uvh gmp6-6.0.0a-1.x86_64.rpm
sudo yum -y install python-devel
# Official git location
git clone https://github.com/dlitz/pycrypto.git
cd pycrypto
git fetch --tags
# Tag of latest stable
git checkout v2.6.1

{% endhighlight %}

Ok here we have to change the setup.py just a little bit to make sure that it will utilize our new gmp while building.  For whatever reason, even exporting the CFLAGS and the LDFLAGS didn't quite get me there for this particular packages.  In any case here is the diff:

{% highlight diff %}

--- a/setup.py
+++ b/setup.py
@@ -370,7 +370,7 @@
 kw = {'name':"pycrypto",
       'ext_modules': plat_ext + [
             # _fastmath (uses GNU mp library)
             Extension("Crypto.PublicKey._fastmath",
-                      include_dirs=['src/','/usr/include/'],
+                      include_dirs=['src/','/usr/include/','/opt/gmp6/include'],
                       libraries=['gmp'],
                       sources=["src/_fastmath.c"]), 

{% endhighlight %}

So apply it how you will, I just modified the include_dirs line under the _fastmath section manually to add the /opt/gmp6/include, but you could make the above a diff file and apply if you chose.

With that diff in place, it is now time to build PyCrypto in the way that Python likes it.  This will get us close to where we need to be, but not quite there.  I'll explain more below

{% highlight console %}

CFLAGS=-I/opt/gmp6/include LDFLAGS=-L/opt/gmp6/lib ./setup.py bdist --format=rpm
Ok we're getting close here.  We have an RPM, but it's not quite right.  First the package is called pycrypto instead of the name of the OS package python-crypto, so we cannot cleanly upgrade with rpm or yum commands.  Let's install the source RPM build by this process and alter the SPEC just a bit.  
rpm -ivh ./dist/pycrypto-2.6.1-1.src.rpm
cd ~/rpmbuild/SPECS
cp pycrypto.spec python-crypto.spec

{% endhighlight %}

Here's a diff of what I did to change the spec:

{% highlight diff %}

--- pycrypto.spec       2014-11-18 20:05:05.000000000 +0000
+++ python-crypto.spec  2014-11-18 20:16:15.582803111 +0000
@@ -1,13 +1,13 @@
-%define name pycrypto
+%define name python-crypto
 %define version 2.6.1
 %define unmangled_version 2.6.1
-%define release 1
+%define release 1.99.gmp6
 
 Summary: Cryptographic modules for Python.
 Name: %{name}
 Version: %{version}
 Release: %{release}
-Source0: %{name}-%{unmangled_version}.tar.gz
+Source0: pycrypto-%{unmangled_version}.tar.gz
 License: UNKNOWN
 Group: Development/Libraries
 BuildRoot: %{_tmppath}/%{name}-%{version}-%{release}-buildroot
@@ -19,10 +19,10 @@
 UNKNOWN
 
 %prep
-%setup -n %{name}-%{unmangled_version}
+%setup -n pycrypto-%{unmangled_version}
 
 %build
-env CFLAGS="$RPM_OPT_FLAGS" python setup.py build
+env CFLAGS="$RPM_OPT_FLAGS -I/opt/gmp6/include" LDFLAGS="-L/opt/gmp6/lib" python setup.py build
 
 %install
 python setup.py install -O1 --root=$RPM_BUILD_ROOT --record=INSTALLED_FILES

{% endhighlight %}

A number of changes, but not too bad.  Now let's rebuild all with our new specfile:

{% highlight console %}

rpmbuild -ba python-crypto.spec
# make sure that package upgrades completely.
sudo rpm -Uvh ../RPMS/python-crypto-2.6.1-1.99.gmp6.x86_64.rpm

{% endhighlight %}

From there, we should get these RPMs signed and in your local yum repo (or at least managed by Salt for deployment).



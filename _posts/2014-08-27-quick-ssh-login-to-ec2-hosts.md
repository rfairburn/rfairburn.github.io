---
layout: post
title: "Quick SSH Login to EC2 Hosts"
description: "Generate a list of your EC2 hosts for quick selection"
category: scripting
tags: ["Python", "Amazon AWS", "Cloud"]
---
{% include JB/setup %}
Ever forget the DNS name and the IP of the Amazon EC2 host you were wanting to login to?  I know I have.  The script below will help with that.

### manage_ec2.py

This little script was based upon an idea I saw from a [coworker](https://github.com/seancdugan) at [C2FO](http://c2fo.com).  His solution was in bash, but relied upon Java to be able to handle the communicate with EC2.  I was hoping for something a little more elegant, unified, and fast.  Thus this script was born.  For your viewing pleasure (as always, latest on [Github](https://gist.github.com/rfairburn/15553a7eeb980302b84a)):

{% highlight python %}

#!/usr/bin/env python
# -*- coding: utf-8 -*-

"""
This script will list all hosts in an ec2 region and prompt you to connect
to them.

It expects the file .boto to exist in your home directory with contents
as follows:

[Credentials]
aws_access_key_id = <AWS_ACCESS_KEY_ID>
aws_secret_access_key = <AWS_SECRET_ACCESS_KEY>
"""

import boto.ec2
import os
import sys
from itertools import imap
from subprocess import call
import yaml
import datetime
import argparse


def parse_args():
    """
    Get arguments from the command like with argparse
    """
    parser = argparse.ArgumentParser(
        description='Generate a list of ec2 hosts and select one to connect to'
        )
    parser.add_argument(
        '--login_name', '-l', required=False, help='Login Name')
    parser.add_argument(
        '--port', '-p', required=False, help='SSH Port')
    parser.add_argument(
        '--identity_file', '-i', required=False, help='SSH Identity File')
    parser.add_argument(
        '--force_download', '-F', required=False,
        help='Force Download', default=False)
    parser.add_argument(
        '--file', '-f', required=False, help='Config File',
        default=os.path.expanduser('~') + '/.ec2_addr_cache')
    parser.add_argument(
        '--region', '-r', required=False, help='EC2 Region',
        default='us-west-2')
    parser.add_argument(
        '--perpage', '-P', required=False,
        help='Hosts to display per page (default 25, 0 to display all)',
        default=25, type=int)
    args = parser.parse_args()
    return args


def generate_hosts(filename, args):
    """
    Generate a list of hoosts in the ec2 region selected and save
    them to a yaml file
    """
    hosts = {}
    conn = boto.ec2.connect_to_region(args.region)
    reservations = conn.get_all_reservations()
    for reservation in reservations:
        instances = reservation.instances
        for instance in instances:
            if instance.state == 'running':
                if 'Name' in instance.tags:
                    hosts.update(
                        {instance.tags['Name']:
                            {'ip': instance.private_ip_address,
                                'key_name': instance.key_name}})
    with open(filename, 'w') as outfile:
        outfile.write(yaml.dump(hosts, default_flow_style=False))
    return hosts


def read_hosts(filename):
    """
    Load saved ec2 host file
    """
    hosts = yaml.load(file(filename, 'r'))
    return hosts


def hosts_dict(args):
    ec2_addr_cache_file = args.file
    if os.path.isfile(ec2_addr_cache_file) and not args.force_download:
        file_age = int(os.path.getmtime(ec2_addr_cache_file))
        now = int(datetime.datetime.now().strftime("%s"))
        if (file_age + 86400) < now:
            hosts = generate_hosts(ec2_addr_cache_file, args)
        else:
            hosts = read_hosts(ec2_addr_cache_file)
    else:
        hosts = generate_hosts(ec2_addr_cache_file, args)

    return hosts


def list_servers(args, startindex, perpage):
    """
    List the servers stored in the ec2 region file or regenerate the file if
    older than 24 hours or forced via the -F True flag
    """
    hosts = hosts_dict(args)
    length = max(imap(len, hosts))
    hostcount = len(hosts)
    if perpage <= 0:
        perpage = hostcount
    countwidth = len(str(hostcount))
    hostlist = sorted(hosts, key=lambda s: s.lower())
    if startindex < 0:
        startindex = 0
    elif startindex >= hostcount:
        startindex = hostcount - perpage
    for index in range(startindex, startindex + perpage):
        if index >= hostcount:
            break
        host = hostlist[index]
        # Start the list with 1 instead of 0 for humans
        printindex = index + 1
        print "%s) %s | %s | %s" % (
            str(printindex).rjust(countwidth),
            host.ljust(length), hosts[host]['ip'].ljust(15),
            hosts[host]['key_name'])
    print "Displaying hosts %i to %i of %i" % (
        startindex + 1, printindex, hostcount)
    selected_index = raw_input(
        "Please enter the host to connect to or (n)ext, (p)revious, (q)uit: ")
    if isinstance(selected_index, basestring) and selected_index == 'q':
        sys.exit(0)
    elif isinstance(selected_index, basestring) and selected_index == 'n':
        list_servers(args, startindex + perpage, perpage)
    elif isinstance(selected_index, basestring) and selected_index == 'p':
        list_servers(args, startindex - perpage, perpage)
    else:
        try:
            selected_index = int(selected_index) - 1
            # Indexes start at 0, so subtract one from here and length tests.
            if not (0 <= selected_index <= len(hostlist) - 1):
                raise TypeError('Range not valid!')
        except:
            print "Invalid entry, try again!"
            args.force_download = False
            list_servers(args)
        else:
            host_ip = hosts[hostlist[selected_index]]['ip']
            ssh = ['ssh']
            if 'port' in args and args.port is not None:
                ssh.extend(['-p', str(args.port)])
            if 'login_name' in args and args.login_name is not None:
                ssh.extend(['-l', args.login_name])
            if 'identity_file' in args and args.identity_file is not None:
                ssh.extend(['-i', args.identity_file])
            ssh.extend([host_ip])
            call(ssh)


if __name__ == '__main__':
    args = parse_args()
    list_servers(args, 0, args.perpage)

{% endhighlight %}

### About the script

This script should work on Python 2.6 and 2.7.  I have not tried it on 3.x, so YMMV there.  Requirements that are not part of a standard Python install are boto and PyYAML.  The script only polls AWS for the list of EC2 instances once per day (override with -F True), and stores the cached values in a YAML file in your home directory.  The list generated will be the list of private IP addresses for your instances, the 'Name' tag that was setup at creation of the server, and the 'Key' name that was used at creation to allow access as the ec2-user. 

Just enter the number next to the server in the list to try and ssh to it directly.

Please post below if you have any questions, otherwise I hope you found this useful!
 

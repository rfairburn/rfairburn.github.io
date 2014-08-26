---
layout: post
title: "VPN and DNS with Mac OS X"
description: "Beating that Mac OS resolver into shape"
category: scripting
tags: ["bash", "Mac OS X"]
---
{% include JB/setup %}
While I will say that I do not often use a Mac, I was recently given a MacBook Pro as my work system when I started at [C2FO](http://c2fo.com).  While my requirements for a system are usually pretty low (a terminal, ssh, vim, and a web browser), I quickly found that the Mac DNS resolver did not behave as I would expect, especially when connecting to my office VPN.

### VPN Woes

Now that is not to say that the VPN was not offering DNS.  Indeed it was.  In fact, I could even see that resolv.conf was getting updated when I connected.  Now I am aware that most of the operating system does not actually use the resolv.conf, so I also verified that the DNS entries were getting created with scutil as well. That being said, I still could not resolve internal hosts.

### The Script

What I found is that anything that modifies DNS in the 'scutil' manner was not reliable.  With that knowledge I made some shell functions that would poll scutil and update that DNS via networksetup.  With these sourced into my .bash_profile, I could connect and disconnect from the VPN, knowing that I would get the resolvers I wanted, and they would work for everything.  You can check out my Github [Gist](https://gist.github.com/rfairburn/b078be55f0be255add27) for the latest version, or use the version below, which was current as of this writing:

{% highlight bash %}

#!/bin/bash

# Source this file in your .bash_profile e.g.:
#
# source ~/gitcheckouts/vpn_heplers/.vpn_helpers.sh
#
# Note: This script works best with NOPASSWD: ALL configured in your sudoers file:
# /etc/sudoers:
# %admin  ALL=(ALL) NOPASSWD: ALL
#
# Usage:
#
# vpn_connect             - Connect to VPN and force DNS into place
# vpn_disconnect          - Disconnect and reset DNS to defaults
# update_resolver         - Force VPN DNS into place (good for already established connections)
# update_resolver delete  - Reset DNS to defaults (good if VPN is already disconnected)


function obtain_service_ids {

scutil <<<"list" | awk -F/ '/DNS$/ { if (length($(NF-1)) == 36 ) { print $(NF-1) } }' | sort -u

}

function obtain_service_id {
scutil <<EOF | awk -F' : ' '/PrimaryService/ { print $2 }'
open
get State:/Network/Global/IPv4
d.show
quit
EOF
}

function modify_scutil_dns {
declare -a DOMAINS=("${!1}")
shift
local RESOLVERS=$@
sudo scutil <<EOF
open
d.init
d.add ServerAddresses * ${RESOLVERS}
$(for DOMAIN in ${DOMAINS[@]}; do echo d.add DomainName ${DOMAIN}; done)
set State:/Network/Service/${SERVICE_ID}/DNS
quit
EOF
}

function modify_networksetup_dns {
declare -a DOMAINS=("${!1}")
shift
local RESOLVERS=$@
local INTERFACE=${NETWORK_INTERFACE}
sudo networksetup -setsearchdomains ${INTERFACE} ${DOMAINS[@]}
sudo networksetup -setdnsservers ${INTERFACE} ${RESOLVERS}
}

function obtain_scutil_dns {
scutil <<EOF | awk -F' : ' 'BEGIN { resolvers = ""}; { if ($1 ~ "[0-9]$") { resolvers = resolvers OFS $2 }}; END { print resolvers }'
open
get State:/Network/Service/${SERVICE_ID}/DNS
d.show
quit
EOF
}

function update_resolver {
if [ "$1" == "delete" ]; then
  local RESOLVERS="${DEFAULT_DNS}"
  local DOMAINS=("${DEFAULT_SEARCH_DOMAINS[@]}")
else
  local DOMAINS=("${OFFICE_SEARCH_DOMAINS[@]}")
  local ADD_RESOLVER=${OFFICE_DNS}
  local SCUTIL_RESOLVERS="$(obtain_scutil_dns)"
  local RESOLVERS="${ADD_RESOLVER} ${SCUTIL_RESOLVERS/${WORK_RESOLVER}/}"
fi
# modify_scutil_dns ${RESOLVERS}
modify_networksetup_dns DOMAINS[@] ${RESOLVERS}
}


function vpn_connect {
/usr/bin/env osascript <<EOF
tell application "System Events"
  tell current location of network preferences
    set VPN to service "${OFFICE_VPN_NAME}"
    if exists VPN then connect VPN
       repeat while (current configuration of VPN is not connected)
         delay 1
       end repeat
  end tell
end tell
return
EOF
update_resolver
}

function vpn_disconnect {
/usr/bin/env osascript <<EOF
tell application "System Events"
  tell current location of network preferences
    set VPN to service "${OFFICE_VPN_NAME}"
      if exists VPN then disconnect VPN
    end tell
end tell
return
EOF
update_resolver delete
}

# Set these to "empty" for DHCP
export DEFAULT_DNS="empty"
# Use a bash array to support multiple search domains
export DEFAULT_SEARCH_DOMAINS=( empty )
# Used in the scripts above
export SERVICE_ID=$(obtain_service_id)
# Configure these for your settings
export OFFICE_DNS="192.168.5.50"
export OFFICE_VPN_NAME="Office VPN"
# Use a bash style array for OFFICE_SEARCH_DOMAINS
export OFFICE_SEARCH_DOMAINS=( c2fo.com corp.c2fo.com )
export NETWORK_INTERFACE="Wi-Fi"

{% endhighlight %}

### Usage

The usage of the script is pretty straight forward.  Once you have sourced it in your .bash_profile, you can use 'vpn_connect' from the command line to bring up the connection, and 'vpn_disconnect' to tear it down.  The exports at the bottom of the script should be updated to match your particular needs.  Other than that, the only real issue is if the VPN disconnects without using the 'vpn_disconnect' function.  In this case, you can always run 'update_resolver delete' to reset the DNS to pre-VPN defaults.


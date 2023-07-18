# privacy-router
A guide to implement a private, secure, and free (as in freedom) OpenWRT-based homelab / secure home network

## Introduction

## Step 1: Prepare the device

## Step 2: Configure OpenWrt

### SSH

### DNS

We'll setup the router make aware to all network devices to send DNS queries to it. We'll tell the router not to send anything to our ISP, but to send them to our forwarder instead. We'll use `dnsmasq` to make forwarding decisions for queries. `dnsmasq` will forward those queries to the DNS resolver - we will set up `dnscrypt-proxy` for this purpose. `dnscrypt-proxy` will create implement anonymous DNS resolution between a set of anonymous relays and pre-selected resolvers with cryptographically safe, tamper-resistent, a privacy-enhancing features.

#### dnsmasq

First, update your packages. Then install some packages.

```
opkg update; opkg add dnsmasq bind-tools curl
```

Then, edit `/etc/dnsmasq.conf`

```
cp /etc/dnsmasq.conf{,.bak}

# READ this configuration file
# It is commented to understand the parameters used
curl -s https:// > /etc/dnsmasq.conf

# Enable and start dnsmasq
/etc/init.d/dnsmasq enable
/etc/init.d/dnsmasq start
```
Ensure you can query DNS through the service

```
dig +short a example.com @127.0.0.1
```

Next, set localhost to forward all resolution to this address. Set it's immutable bit to prevent overwrites from a network manager.

```
cat <<EOF > /etc/resolv.conf
nameserver ::1
nameserver 127.0.0.1
options trust-ad
EOF
```


#### dnscrypt-proxy

```
opkg install dnscrypt-proxy2
uci add_list dhcp.@dnsmasq[0].server='127.0.0.53'
uci commit dhcp
```

Edit `/etc/config/dhcp`

```
config dnsmasq
	# Ignore ISP's DNS by not reading upstream servers from /etc/resolv.conf
	option noresolv '1'
	# Ensure resolv.conf directs local processes to dnsmasq
	option localuse '1'
	# Disable dnsmasq cache as privoxy is more performant
	option cachesize '0'
```

Instruct dnsmasq to use dnscrypt-proxy as its server


```
sed 's/#server=127.0.0.53/server=127.0.0.53/'
/etc/init.d/dnsmasq restart
```

Check

You'll want to ensure NTP can work without DNS. In `/etc/config/system > config timeserver 'ntp'`, add these entries:

```
list server '216.239.35.0'    # Google
list server '216.239.35.4'    # Google
list server '216.239.35.8'    # Google
list server '216.239.35.12'   # Google
list server '162.159.200.123' # Cloudflare
list server '162.159.200.1'   # Cloudflare
```

Restart: `/etc/init.d/sysntpd restart

Completely disably ISP DNS servers. In `/etc/config/network`:

```
config interface 'wan'
	option peerdns '0'
```

Check

Force LAN clients to send DNS queries to dnscrypt-proxy. Add the following rules into `/etc/config/firewall`:

```
# Redirect unencrypted DNS queries to dnscrypt-proxy
# This will thwart manual DNS client settings and hardcoded DNS servers like in Google devices
config redirect
    option name 'Divert-DNS, port 53'
    option src 'lan'
    option dest 'lan'
    option src_dport '53'
    option dest_port '53'
    option target 'DNAT'

# Block DNS-over-TLS over port 853
# Assuming you're not actually running a DoT stub resolver
config rule
    option name 'Reject-DoT, port 853'
    option src 'lan'
    option dest 'wan'
    option dest_port '853'
    option proto 'tcp'
    option target 'REJECT'

# Optional: Redirect queries for DNS servers running on non-standard ports. Can repeat for 9953, 1512, 54. Check https://github.com/parrotgeek1/ProxyDNS for examples.
# Warning: can break stuff, don't use this one if you run an mDNS server
config redirect
    option name 'Divert-DNS, port 5353'
    option src 'lan'
    option dest 'lan'
    option src_dport '5353'
    option dest_port '53'
    option target 'DNAT'
```
And reload Firewall: /etc/init.d/firewall reload

#### Other services

##### tor

### HTTP/S

#### privoxy

### 


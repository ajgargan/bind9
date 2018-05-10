# bind9 named-chroot CentOS 7 SELinux and Local Firewall enabled

## What is this
UserData script for Centos 7 Chroot Bind DNS Server

tested on 
dcos-centos7-201710261514 - ami-0075d079
in AWS eu-west-1

```shell
#!/bin/sh -x


# Ensure SELinux is off so we can install and configure things
# We will enable it when we are done
setenforce 0

# Set my hostname
hostnamectl set-hostname ns1.dnsdemo.osite.co.za

# install the chrooted bind package
yum install -y bind-chroot firewalld 

# tun off the normal non chroot named
systemsctl stop named
systemctl disable named

# create zone files and config
cat <<EOF >/etc/named.conf
options {
    listen-on port 53 { any; };
    directory       "/var/named";
    dump-file       "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    allow-query     { any; };
    recursion no;

    dnssec-enable yes;
    dnssec-validation yes;

    /* Path to ISC DLV key */
    bindkeys-file "/etc/named.iscdlv.key";

    managed-keys-directory "/var/named/dynamic";

    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
};
logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "dnsdemo.osite.co.za" {
    type master;
    file "dnsdemo.osite.co.za";
    allow-update{none;};
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
EOF

cat <<EOF >/var/named/dnsdemo.osite.co.za
\$ORIGIN dnsdemo.osite.co.za.
\$TTL 60
@   IN  SOA ns1.dnsdemo.osite.co.za.    root.dnsdemo.osite.co.za. (
                2017051004 ; se = serial number
                60         ; ref = refresh
                60         ; ret = update retry
                60         ; ex = expiry
                60 )       ; min = minimum

@           IN      NS      ns1.dnsdemo.osite.co.za.
ns1         IN      A       192.168.0.33
wiki        IN      CNAME   wiki.osite.co.za.
EOF

chown root.named /var/named/dnsdemo.osite.co.za
chmod 640 /var/named/dnsdemo.osite.co.za

# Set SELinux Contexts
semanage fcontext -a -t named_zone_t /etc/named.conf
semanage fcontext -a -t named_zone_t /var/named/dnsdemo.osite.co.za

# setup the chroot jail. The above configs will be merged into the jail
/usr/lib/exec/setup-named-chroot.sh /var/named/chroot on

# turn on and persist the named-chroot service
systemctl enable named-chroot
systemctl start named-chroot

# Set firewalling
systemctl enable firewalld 
systemctl start firewalld
firewall-cmd --permanent --zone=public --add-service dns
firewall-cmd --reload

# Ensure that SELinux is enabled
sed -i 's/permissive/enforcing/g' /etc/selinux/config
setenforce 1
```

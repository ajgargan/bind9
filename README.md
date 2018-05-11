# BIND9 CentOS 7 named-chroot with SELinux and Local Firewalld enabled

## What is this:
UserData script for Centos 7 Chroot Bind DNS Server
* SELinux enabled
* BIND Running in chroot
* Firewalld running on instance
* Single forward only zone example 

## Using this:
* Launch Above AMI in eu-west Region
* Launch into a Public Subnet
* Enable UDP and TCP 53(DNS) in the Security group and TCP 22(SSH) 
* Configure an SSH Keypair you have access to.
* Paste the UserData below into the instance UserData under advanced

## Single Host UserData script
```bash
#!/bin/sh -x
# -x so we can see whats going on from the console output and logs

# Ensure SELinux is off so we can install and configure things
# We will enable it when we are done
setenforce 0

# Set my hostname
hostnamectl set-hostname ns1.dnsdemo.osite.co.za

# install the chrooted bind package
yum install -y bind-chroot firewalld 

# turn off the non chroot named
systemsctl stop named
systemctl disable named

# turn off and disable docker
systemsctl stop docker
systemctl disable docker

# turn off and disable postfix
systemctl stop postfix
systemctl disable postfix

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

# setup the named-chroot. The above configs will be merged into the chroot
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

# restart server
reboot
```

## Testing 
* From an Internet connected host
  * dig @<public_ip> wiki.dnsdemo.osite.co.za
  * dig @<public_ip> ns1.dnsdemo.osite.co.za

## Next Steps 
### Create CFN Template with
* EC2 Instance ns0 as master which does not accept resolves
* EC2 Instance ns1 as slave which resolves AZ1
* EC2 Instance ns2 as slave which resolves AZ2
* 1 EIP for each instance
* UserData which injects the EIP's into the BIND configs
* CFN-Hup for managing changes to the ZoneFile on ns0
* EC2 Instance Bastion host
* SG for Name Servers 
  * Allow zone transfers and lookups to ns0/1/2
  * Allow SSH From Bastion SG
* SG for Bastion
  * Allow SSH from CIDR provided

### Add DNSSEC zone signing
DNSSEC is not currently supported by all TLD and Registrars, notably for co.za.
Being ready for when it is is not a bad thing.
* setup keys
* setup DNSKEY Record
* sign zonefiles with DNSKEY to create RRSIG records
* Test with
  * dig +dnssec @8.8.8.8 some.domain.com
  eg:
  ```
    # dig +dnssec @8.8.8.8 www.cloudflare.com

    ; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> +dnssec @8.8.8.8 www.cloudflare.com
    ; (1 server found)
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51511
    ;; flags: qr rd ra ad; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags: do; udp: 512
    ;; QUESTION SECTION:
    ;www.cloudflare.com.		IN	A

    ;; ANSWER SECTION:
    www.cloudflare.com.	4	IN	A	198.41.215.162
    www.cloudflare.com.	4	IN	A	198.41.214.162
    www.cloudflare.com.	4	IN	RRSIG	A 13 3 5 20180512071419 20180510051419 35273 cloudflare.com. oJ4UB0kPYK0ckwfRndLdT/1up9pb9kFIFDYPe5fPPffcK9h4NyAw1PBy EWzS3TqVUaSAJbdvJezLC5X0gz/3Ig==

    ;; Query time: 25 msec
    ;; SERVER: 8.8.8.8#53(8.8.8.8)
    ;; WHEN: Fri May 11 06:14:29 UTC 2018
    ;; MSG SIZE  rcvd: 189
  ```
  * [https://dnssec-analyzer.verisignlabs.com/]

### User data
* Perform some Kernel Tuning /etc/sysctl.d/99_hardening.conf
  * Network Level
    * Disable the IP Forwarding
      * net.ipv4.ip_forward=0
    * Disable the Send Packet Redirects
      * net.ipv4.conf.all.send_redirects=0
      * net.ipv4.conf.default.send_redirects=0
    * Disable ICMP Redirect Acceptance
      * net.ipv4.conf.all.accept_redirects=0
      * net.ipv4.conf.default.accept_redirects=0
    * Enable Bad Error Message Protection
      * net.ipv4.icmp_ignore_bogus_error_responses=1
  * Other 
    * Restricting Core Dumps 
      * fs.suid_dumpable=0
    * Enable Exec Shield
      * kernel.exec-shield=1
    * Enable randomized Virtual Memory Region Placement
      * kernel.randomize_va_space=2

* Limits Tuning
  * Restrict Core Dumps
    * Add "*    hard   core    0" to /etc/limits.d/99_core_hardening.conf
* Lock down sshd
  * No Remote Root
  * PublicKey authentication Only
  * Alternative SSH Port (Remember to modify SG's)

* Lock down root shell /bin/nologin && /etc/securetty 

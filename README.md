# BIND9 CentOS 7 named-chroot with SELinux and Local Firewalld enabled

## What is this:
UserData script for Centos 7 Chroot Bind DNS Server
* SELinux enabled
* BIND Running in chroot
* Firewalld running on instance
* Single forward only zone example 

**NB!!! THIS IS WORK IN PROGRESS AND BY NO MEANS EXHAUSTIVE**

## Using this:
* Launch dcos-centos7-201710261514(ami-0075d079) in eu-west region . (This was chosen because of dcos(Data Center OS))
* Launch into a Public Subnet
* Enable UDP and TCP 53(DNS) in the Security group and TCP 22(SSH) 
* Configure host with an SSH Keypair you have access to.
* Paste the UserData below into the instance UserData under advanced

## Single Host UserData script
```bash
#!/bin/sh -x
# -x so we can see whats going on from the console output and logs

# Ensure SELinux is off so we can install and configure things
# We will enable it when we are done
setenforce 0

# Update all packages
yum update -y

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
    file "dnsdemo.osite.co.za.signed";
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

# Install package to make better entropy for cloud server entropy
yum install -y epel-release 
yum install haveged

# create DNSSEC signing keys
# Key Signing Key
dnssec-keygen -a NSEC3RSASHA1 -b 2048 -n ZONE dnsdemo.osite.co.za
# Zone Signing Key
dnssec-keygen -f KSK -a NSEC3RSASHA1 -b 4096 -n ZONE dnsdemo.osite.co.za

# Ensure private key perms
chown root:root /var/named/Kdnsdemo.osite.co.za.*private
chmod 600 /var/named/Kdnsdemo.osite.co.za.*private

# Find key file names to add to zone
KEY1=$(basename $(ls /var/named/Kdnsdemo.osite.co.za.*key | head -n1))
KEY1=$(basename $(ls /var/named/Kdnsdemo.osite.co.za.*key | tail -n1))

# Add keys to zone file
cat <<EOF >>/var/named/dnsdemo.osite.co.za
\$INCLUDE $KEY1
\$INCLUDE $KEY2
EOF

# sign zone (creates /var/named/dnsdemo.osite.co.za.signed)
dnssec-signzone -A -3 $(head -c 1000 /dev/random | sha1sum | cut -b 1-16) -N INCREMENT -o dnsdemo.osite.co.za -t dnsdemo.osite.co.za

# Set SELinux Contexts (not reallly required see autorelabel near the end)
# semanage fcontext -a -t named_zone_t /etc/named.conf
# semanage fcontext -a -t named_zone_t /var/named/dnsdemo.osite.co.za

# setup the named-chroot. The above configs will be merged into the chroot
/usr/lib/exec/setup-named-chroot.sh /var/named/chroot on

# turn on and persist the named-chroot service
systemctl enable named-chroot
systemctl start named-chroot

# Set firewalling to allow DNS access
systemctl enable firewalld 
systemctl start firewalld
firewall-cmd --permanent --zone=public --add-service dns
firewall-cmd --reload

# Harden Kernel Options
cat <<EOF >/etc/sysctl.d/hardening.conf
#Disable the IP Forwarding
net.ipv4.ip_forward=0
#Disable the Send Packet Redirects
net.ipv4.conf.all.send_redirects=0
net.ipv4.conf.default.send_redirects=0
#Disable ICMP Redirect Acceptance
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.default.accept_redirects=0
#Enable Bad Error Message Protection
net.ipv4.icmp_ignore_bogus_error_responses=1

#Restricting Core Dumps
fs.suid_dumpable=0
#Enable Exec Shield
kernel.exec-shield=1
#Enable randomized Virtual Memory Region Placement
kernel.randomize_va_space=2
EOF

# Harden limits around core dumps
cat <<EOF >/etc/limits.d/99_core_hardening.conf
*    hard   core    0
EOF

# backup sshd config
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.orginal

# Function to be able to set SSHD params
function set_sshd_param() 
{ 
   grep -v $1 /etc/ssh/sshd_config > /etc/ssh/sshd_config.tmp
   echo "$1 $2" >> /etc/ssh/sshd_config.tmp 
   
   # Only change the actual config if we don't break it.
   /usr/sbin/sshd -t
   if [ $? -eq 0 ]
   then
       mv /etc/ssh/sshd_config.tmp /etc/ssh/sshd_config
   fi
}

set_sshd_param Protocol 2
set_sshd_param PubkeyAuthentication yes
set_sshd_param PasswordAuthentication no
set_sshd_param PermitRootLogin no
set_sshd_param UsePAM yes
set_sshd_param Port 99

# restart sshd the function above already tests for a broken config
# we will reboot and then sshd will come up with new config anyway
# systemctl sshd restart

# Ensure that SELinux is enabled
sed -i 's/permissive/enforcing/g' /etc/selinux/config

# Tell SELinux to correct the file contexts
touch /.autorelabel

# restart server so that sysctl/limits changes can be effected and SELinux can be turned on and relabel file contexts as required.
reboot
```

## Testing 
* From an Internet connected host
  * dig @<your_public_ip> wiki.dnsdemo.osite.co.za
  * dig @<your_public_ip> ns1.dnsdemo.osite.co.za

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
~~DNSSEC is not currently supported by all TLD and Registrars, notably for co.za.~~ This was added late 2017.
~~Being ready for when it is is not a bad thing.~~
* setup Key Signing Keys and Zone Signing Key
* add both Keys to bottom of zonefile($INCLUDE <Keynamehere>)
* sign zonefiles with Zone Signing Key
* configure DS Records with registrar (osite.co.za) in this case. Which itself will need to be signed up to the TLD.
* Test with:
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
  * [versignlabs dnssec tool](https://dnssec-analyzer.verisignlabs.com/)

### User data

* Use an isolated block device for the chroot
  * Set the following fsoptions
    * noexec – Do not set execution of any binaries on this partition (prevents execution of binaries but allows scripts).
    * nodev – Do not allow character or special devices on this partition (prevents use of device files such as zero, sda etc).
    * nosuid – Do not set SUID/SGID access on this partition (prevent the setuid bit).
    
* Lock down root shell /bin/nologin && /etc/securetty 
  * echo > /etc/securetty
  * usermod -s /sbin/login root

* Possibly look at fail2ban etc. and also TCP Connection rate limiting via Firewalld/IPTABLES but realistically SSH **shouldn't** be accessible directly from outside the network. http://www.win.tue.nl/~vincenth/ssh_rate_limit_firewalld.htm

* Enable Auditing 
* Enable remote syslogging
* Enable host monitoring

## References:
### DNSSEC
* https://www.registry.net.za/content.php?gen=1&contentid=57&title=.ZA+Position+Statement+on+DNSSEC
* https://dnssec.co.za/
* https://nkosi.co.za/co-za-dnssec-how-to/
* https://www.digitalocean.com/community/tutorials/how-to-setup-dnssec-on-an-authoritative-bind-dns-server--2
### SELinux
* https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-security_threats
### Host Hardening
* https://www.cyberciti.biz/tips/linux-security.html

# Antidistraction Gateway

The internet is full of things that cause distractions, Facebook, Tictoc, Youtube. Major corporations have turned what used to be a tool for learning and productivity and entertainment into a dopamine machine that is hijacking our reward systems, disturbing our sleep, aging us, and worse stealing our futures.

While there are certainly browser plugins that can accomplish this, they don't add enough friction to really be effective. What good is installing a browser plugin you can click a button and disable or remove? 
For anyone even remotely technical, more is needed, and this project is geared for people with a moniker of Linux knowledge to build just that.



# Implementation

Because simple firewall rule tables are not granular enough to really function without braking extremely useful tools that exist, Antidistraction functions as a MITM proxy using Squid and DNSMasq. DNSMasq provides DHCP and DNS control, and Squid provides the restrctions - along with the performance benefit of having a proxy inline between yourself and the internet. 





# Platform

All that is needed is a system with 2 network controllers that is capable of running Linux. Thats a pretty low bar. I use a simple Dell laptop as my antidistraction gateway with a USB nic and the onboard nic. This has all been setup under Ubuntu 24.04LTS SERVER as of the time of this writing, so you'll want to have that installed on your gateway with nothing else installed.

The reference system has the following networks setup:


```
enx3c18a0d58d07: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.2.1  netmask 255.255.255.0  broadcast 192.168.2.255
--
enxa44cc876292a: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.134  netmask 255.255.255.0  broadcast 192.168.1.255

```

In the above, 192.168.2.1 is a static IP for the internal network you'll be creating using this gateway, and the 192.168.1.134 address is DHCP assigned from my home router.



# Installation

## Networking

I prefer using Network manager over netplan to manage my interfaces its just easier:

```
sudo apt update
sudo apt install -y network-manager
sudo systemctl enable NetworkManager
sudo systemctl start NetworkManager
#Get rid of the annoying 2 minute artifical wait on bootup:
systemctl disable systemd-networkd-wait-online.service
```

Make /etc/netplan/50-cloud-init.yaml look like:


```
network:
  version: 2
  renderer: NetworkManager

```

Run:

```
sudo netplan generate
sudo netplan apply
```

Reboot the system. On reboot all networking will not be working.

Use 'nmtui' to configure the interfaces using a curses interface.



## AntiDistraction Installation

Run the following commands:

```
apt-get update
apt install -y iptables-persistent net-tools ethtool ifupdown squid-openssl plocate dnsmasq selinux-utils git python3
systemctl enable dnsmasq
systemctl enable squid-ssl
```

Some of the above are simple QoL.

Setup the SSL root authority, this is so Squid can self sign certificates for all sites:

```
#Create the main key and cert in one command:
sudo -u proxy openssl req -new -newkey rsa:4096 -sha256 -days 7300 -nodes -x509 \
  -keyout /etc/squid/ssl/mitm.key \
  -out    /etc/squid/ssl/mitm.crt \
  -subj "/CN=Squid MITM CA"

#Setup diffie hellmen parameters:
sudo -u proxy openssl dhparam -out /etc/squid/ssl/dhparam.pem 2048

#Verify basic constraints:
sudo -u proxy openssl x509 -in /etc/squid/ssl/mitm.crt -text -noout | grep -A2 "Basic Constraints"
# Expect: CA:TRUE

#Copy and convert PEM to DER for other systems:
sudo -u proxy openssl x509 -in /etc/squid/ssl/mitm.crt -outform der -out /var/lib/squid/ca.der

#Initialize the Squid CA database:
sudo -u proxy mkdir -p /var/lib/squid/ssl_db
sudo -u proxy /usr/lib/squid/security_file_certgen -c -s /var/lib/squid/ssl_db -M 64MB


#Pull the repo:
cd /
git clone https://github.com/User-4574/AntiDistraction

#Backup files
mv /etc/dnsmasq.d/router.conf /etc/dnsmasq.d/router.conf.orig
mv /etc/squid/squid.conf /etc/squid/squid.conf.orig
mv /etc/iptables/rules.v4 /etc/iptables/rules.v4.orig
mv /var/spool/cron/crontabs/root /root/root.crontab.orig
mv /etc/rc.local /root/


#Update router.conf to pin machines or set static DNS overrides for your network.
ln -sf /AntiDistraction/dnsmasq/router.conf /etc/dnsmasq.d/router.conf

#Squid
ln -sf /AntiDistraction/squid/squid.conf /etc/squid/squid.conf

#Before symlinking these files, edit them, and insure the interfaces represent your environment.
#Your public interface=eth0
#Your private interface=eth1
InternalInterface=eth1
ExternalInterface=eth0
/AntiDistraction/iptables/rules.v4
for f in rules.v4 rules.v4_restricted; do
  tmp=$(mktemp)
  sed \
    -e "s/$InternalInterface/enx3c18a0d58d07/g" \
    -e "s/$ExternalInterface/enxa44cc876292a/g" \
    "/AntiDistraction/iptables/$f" >"$tmp"
  mv "$tmp" "/AntiDistraction/etc/iptables/$f"
done

ln -sf /AntiDistraction/iptables/rules.v4 /etc/iptables/rules.v4
ln -sf /AntiDistraction/iptables/rules.v4_restricted /etc/iptables/rules.v4_restricted

#These cron jobs are used to cut off internet access after 9PM. This helps sleep.
ln -sf /AntiDistraction/cron/root /var/spool/cron/crontabs/root
chmod +x /AntiDistraction/etc/rc.local 

#Increase FS limits for Squid stability
mv /etc/security/limits.conf /root
ln -sf /AntiDistraction/etc/security/limits.conf /etc/security/limits.conf

#Setup zram
ln -sf /AntiDistraction/etc/rc.local /etc/rc.local
ln -sf /AntiDistraction/etc/systemd/system/rc-local.service /etc/systemd/system/rc-local.service
systemctl daemon-reload
systemctl enable rc-local

#Start services
systemctl restart rc-local
systemctl restart dnsmasq
systemctl restart squid

```



At this point Squid should be running. 





## Setting up trust

Even if Squid is running fine you still have to setup trust within your systems, because Squid will be generating an internal CA signed certificate for every site that you visit on-the-fly. 



### On Linux

First you'll need to install 

You'll need to grab the CA certificate from your Proxy server. In ssh just:


```
cat /etc/squid/ssl/mitm.crt
```

Copy and paste the output to:

```
~/mitm.crt
#switch to root and also copy it to:
/usr/local/share/ca-certificates/squid-mitm.crt
update-ca-certificates
```

You will also have to install the certificate into your browser, in chrome go to:

```
chrome://certificate-manager/
```

And install it there.

### On iphone

Iphones have their own trust format and the certificate must be installed from a web server, uploading them to the phone doesn't work.

First, we need to get the base64 version of the DER certificate

```
base64 -w 0 /var/lib/squid/ca.der
```

Create a folder to serve the file:

```
mkdir ~/http
cd ~/http
CERT_PEM=/etc/squid/ssl/mitm.crt
OUT=Squid-CA.mobileconfig

UUID1=$(uuidgen)
UUID2=$(uuidgen)

cat >"$OUT" <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>PayloadContent</key>
  <array>
    <dict>
      <key>PayloadCertificateFileName</key>
      <string>Squid-CA.cer</string>
      <key>PayloadContent</key>
      <data>
$(sed '1d;$d' "$CERT_PEM" | tr -d '\n')
      </data>
      <key>PayloadDescription</key>
      <string>Root CA used by Squid proxy for SSL inspection</string>
      <key>PayloadDisplayName</key>
      <string>Squid Proxy Root CA</string>
      <key>PayloadIdentifier</key>
      <string>com.local.squidproxy.rootca.payload</string>
      <key>PayloadType</key>
      <string>com.apple.security.root</string>
      <key>PayloadUUID</key>
      <string>$UUID1</string>
      <key>PayloadVersion</key>
      <integer>1</integer>
    </dict>
  </array>
  <key>PayloadDisplayName</key>
  <string>Squid Proxy Root CA</string>
  <key>PayloadIdentifier</key>
  <string>com.local.squidproxy.rootca</string>
  <key>PayloadRemovalDisallowed</key>
  <false/>
  <key>PayloadType</key>
  <string>Configuration</string>
  <key>PayloadUUID</key>
  <string>$UUID2</string>
  <key>PayloadVersion</key>
  <integer>1</integer>
</dict>
</plist>
EOF
#Start a simple file server using Python.
python3 -m http.server 80

```

Now, in your phone go to:
Settings → General → About → Certificate Trust Settings

Point it at:

```
http://192.168.2.1/Squid-CA.mobileconfig

```

**The most important step in this process, prior to performing the above, requires a holy offering to the Apple gods** — something properly sanctified.

An offering from **Church’s Chicken** has proven sufficient in past rituals. Do **not** forget the coleslaw; the Apple gods frown upon offerings lacking holy cabbage.







# Other considerations

## Youtube

Youtube will not work after you implement this. However Youtube does have some useful videos, we simply want to eliminate distractions. I use a program called FreeTube. To make Freetube work adjust it to use an http proxy:

- Go to settings
- Set current invidious instance, mine is:https://inv.nadeko.net
- Tick enable tor/proxy
- Set proxy server to 192.168.2.1
- Proxy port number 3130

There are many other controls to prevent Youtube from turning into a distraction machine, such as filtering political content, there is an entire "Distraction Free" section to Freetube and I recommend checking it out.



## Other Applications/Bitwarden desktop

I use Bitwarden Desktop t manage my ssh keys and lots of other things, and its extremely picky about MITM proxies snooping traffic and even if you Splice everything associated with Bitwarden just doesn't work. The only way I've gotten it to work is:

- Dumping the snap version, and using of the appimage from their site.
- Install proxychains and set it up
- Use a script file to setup an SSH tunnel and proxychain a socks5 proxy to bypass Squid for the application entirely.

**On your workstation**

Install and configure proxychains:

```
apt-get update
apt install proxychains4

```

Make /etc/proxychains4.conf look like:

```
strict_chain
proxy_dns
remote_dns_subnet 224
tcp_read_time_out 15000
tcp_connect_time_out 8000
[ProxyList]
socks5  127.0.0.1 1080

```

Download the Bitwarden desktop appimage here:
https://bitwarden.com/download/#downloads-desktop

```
mkdir ~/bin
mv ~/Downloads/Bi<tab>  <- The appimage you downloaded above.
rename that appimage to Bitwarden.appimage
chmod 766 Bitwarden.appimage
```

Create a file called '**bitwardenproxy.sh**'

```
#!/bin/bash
set -euo pipefail

APP="$HOME/bin/Bitwarden.appimage"
USER=myuser
# Ensure SOCKS tunnel exists before launching (optional sanity check)
if ! ss -ltn | grep -qE '127\.0\.0\.1:1080\b'; then
  echo "SOCKS tunnel not listening on 127.0.0.1:1080 Starting Proxy" >&2
  ssh -i ~/.ssh/router -N -D 127.0.0.1:1080 $USER@192.168.2.1 &
fi

exec proxychains4 -q "$APP" --no-sandbox "$@"

```

Be sure you set USER in the above to the user you use to SSH to your router.



You will need need to have an authorized key on the router for your user. So on the router use ssh-keygen to create a private/pub keys (as your user):

```
root@jamierouter:~/http# ssh-keygen
Generating public/private ed25519 key pair.
Enter file in which to save the key (/root/.ssh/id_ed25519): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_ed25519
Your public key has been saved in /root/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:vjwkKOtodUCQPLWiCD8NNChqx/EeL+BAekS+UCaJh/s root@jamierouter
The key's randomart image is:
+--[ED25519 256]--+
|+*X.             |
|*@o+.            |
|B=*.o            |
|B=oO o           |
|+oB =.o S        |
|  E+.+.o.        |
|  .o. .o.        |
| o.    ...       |
|o..     o.       |
+----[SHA256]-----+
root@jamierouter:~/http# 

```

Now, in the above example:

1. The public key is /root/.ssh/id_ed25519.pub and needs to be copied to authorized keys on the router
2. The private key is /root/.ssh/id_ed25519 and will need to be placed on your workstation as ~/.ssh/router 

On the router as your user:

```
cd ~/.ssh
mv id_ed25519.pub authorized_keys
chmod 400 authorized_keys
cat id_ed25519
```

Now, on your workstation:

```
cd ~/.ssh
vi router
<paste the output of the cat on the router> :wq
chmod 400 router
```







# Further develompent and other information

## Facebook

Facebook redirects to facebook.com/marketplace. This is by design. This is not perfect. I am however blocking their graphql which makes the site seem wierd. You can refresh pages and they'll work, including your feed, but it absolutely prevents doomscrolling while maintaining some use of the site. Future updates will rate limit these so Facebook Marketplace, which is useful will work but not allow doom scrolling. Facebook will probably never work perfectly, I had to disable QUIC altogether to even get Squid rules to work, and mix Facebook and Facebook marketplace so much that allowing one and not the other is probably impossible.

## Amazon Prime Video

I keep Prime Video allowed because I'm a Stargate SG1 fan, and often times like to have it running in the background while I work on projects like this. In fact its playing right now. If you want for Amazon to work, but not Prime Video uncomment:

```
#http_access deny prime_hosts

```

And reload squid. Most of the other shows there I have already seen, so its not much of a distraction for me.



## Quickly and easily reloading Squid

You don't have to restart squid to load configuration changes. Just run:

```
squid -k reconfigure
```



## Youtube on the phone

As you can see in my dnsmasq router file, I'm pinning 192.168.2.200 to my phone, as well as other machines in my network, and in squid.conf I'm running specific rules on it and its allowing youtube. I have gone back and forth with this.


https://www.nighthour.sg/articles/2018/google-cloud-platform-test-lab-strongswan-vpn.html

gcloud init
gcloud config set compute/region europe-west1
gcloud config set compute/zone europe-west1-b
gcloud compute project-info describe
y
gcloud compute networks create testlabvpc --subnet-mode=custom
gcloud compute networks subnets create mgmt-subnet --network=testlabvpc --region=europe-west1 --range=172.16.0.0/24
gcloud compute networks subnets create testlab-subnet --network=testlabvpc --region=europe-west1 --range=172.16.1.0/24 

gcloud compute addresses create vpnserver-ip mgmtserver-ip \
--region europe-west1 --subnet mgmt-subnet \
--addresses 172.16.0.2,172.16.0.3

gcloud compute addresses create vpnserver-ext-ip \
--region europe-west1 

gcloud compute addresses list 
NAME              ADDRESS/RANGE  TYPE      PURPOSE       NETWORK  REGION        SUBNET       STATUS
mgmtserver-ip     172.16.0.3     INTERNAL  GCE_ENDPOINT           europe-west1  mgmt-subnet  RESERVED
vpnserver-ext-ip  35.241.182.25  EXTERNAL                         europe-west1               RESERVED
vpnserver-ip      172.16.0.2     INTERNAL  GCE_ENDPOINT           europe-west1  mgmt-subnet  RESERVED

gcloud compute firewall-rules create vpn-allow-access --direction=INGRESS --priority=1000 --network=testlabvpc --action=ALLOW --rules=udp:500,udp:4500,tcp:22 --source-ranges=0.0.0.0/0 --target-tags=vpn-allow-access

gcloud compute firewall-rules create mgmt-internal-allow-access --direction=INGRESS --priority=1000 --network=testlabvpc --action=ALLOW --rules=tcp:22,tcp:80,tcp:443,tcp:8080,icmp --source-ranges=172.16.0.0/24

gcloud compute firewall-rules create testlab-internal-deny-all --direction=EGRESS --priority=1000 --network=testlabvpc --action=DENY --rules=all --destination-ranges=0.0.0.0/0 --target-tags=testlab-denyall 

gcloud compute instances create vpnserver --zone=europe-west1-b --machine-type=f1-micro --subnet=mgmt-subnet --private-network-ip=172.16.0.2 --address=35.241.182.25 --maintenance-policy=MIGRATE --tags=vpn-allow-access --image=ubuntu-1804-bionic-v20180522 --image-project=ubuntu-os-cloud --boot-disk-size=10GB 
NAME       ZONE            MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
vpnserver  europe-west1-b  f1-micro                   172.16.0.2   35.241.182.25  RUNNING

gcloud compute instances create mgmtserver --zone=europe-west1-b --machine-type=f1-micro --subnet=mgmt-subnet --private-network-ip=172.16.0.3 --maintenance-policy=MIGRATE --image=ubuntu-1804-bionic-v20180522 --image-project=ubuntu-os-cloud --boot-disk-size=10GB

gcloud compute instances create labserver0 --zone=europe-west1-b --machine-type=f1-micro --subnet=testlab-subnet --no-address --maintenance-policy=MIGRATE --tags=testlab-denyall --image=ubuntu-1804-bionic-v20180522 --image-project=ubuntu-os-cloud --boot-disk-size=10GB

gcloud compute instances create labserver1 --zone=europe-west1-b --machine-type=f1-micro --subnet=testlab-subnet --no-address --maintenance-policy=MIGRATE --tags=testlab-denyall --image=ubuntu-1804-bionic-v20180522 --image-project=ubuntu-os-cloud --boot-disk-size=10GB 

gcloud compute instances list 
NAME        ZONE            MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
labserver0  europe-west1-b  f1-micro                   172.16.1.2                  RUNNING
labserver1  europe-west1-b  f1-micro                   172.16.1.3                  RUNNING
mgmtserver  europe-west1-b  f1-micro                   172.16.0.3   35.195.65.30   RUNNING
vpnserver   europe-west1-b  f1-micro                   172.16.0.2   35.241.182.25  RUNNING

#################################################################################################################################################################

on the remote vpn server

gcloud compute ssh vpnserver

sudo su -

timedatectl set-timezone Europe/Paris

vim /etc/login.defs 
# change to
# UMASK 077

apt-get update
apt-get dist-upgrade 

reboot

apt-get install strongswan iptables-persistent easy-rsa 
yes

vim /etc/security/limits.conf 
# 1024 to 65536

groupadd --gid 8000 strongswan
usermod -g strongswan strongswan 

make-cadir vpn-ca 

cd vpn-ca
vim vars 

export KEY_SIZE=4096
export KEY_COUNTRY="FR"
export KEY_PROVINCE="IleDeFrance"
export KEY_CITY="Paris"
export KEY_ORG="netyou"
export KEY_EMAIL="fabien.gatelier@netyou.fr"
export KEY_OU="netyou"

cp openssl-1.0.0.cnf openssl.cnf
source vars
./clean-all
./build-ca 

./build-key-server remote-vpn-gw 

./build-key-server local-vpn-gw 

cp keys/remote-vpn-gw.key /etc/ipsec.d/private/
cp keys/remote-vpn-gw.crt /etc/ipsec.d/certs/
cp keys/ca.crt /etc/ipsec.d/cacerts/ 

chown strongswan /etc/ipsec.d/private/remote-vpn-gw.key
chgrp strongswan /etc/ipsec.d/private
chmod 750 /etc/ipsec.d/private
chown strongswan /etc/ipsec.d/private/remote-vpn-gw.key
chmod 400 /etc/ipsec.d/private/remote-vpn-gw.key
chmod 644 /etc/ipsec.d/certs/remote-vpn-gw.crt
chmod 644 /etc/ipsec.d/cacerts/ca.crt 

vim /etc/ipsec.secrets 
# : RSA remote-vpn-gw.key 

chown strongswan /etc/ipsec.secrets
chmod 400 /etc/ipsec.secrets 

vim /etc/strongswan.d/charon.conf 
# user = strongswan 

vim /etc/ipsec.conf

config setup
	 strictcrlpolicy=no
	 uniqueids = yes
	 charondebug="ike 2, esp 2,  cfg 2, knl 2"

conn ikev2-vpn
       auto=add	
       compress=no
       keyexchange=ikev2
       aggressive = no
       type = tunnel
       fragmentation = yes
       forceencaps = yes
       ike = aes256gcm16-prfsha384-ecp384!
       esp = aes256gcm16-prfsha384-ecp384!
       lifetime = 1h
       dpdaction = clear
       dpddelay = 300s       

       leftid= "C=FR, ST=IleDeFrance, L=Paris, O=netyou, OU=netyou, CN=remote-vpn-gw, N=remote-vpn-gw, E=fabien.gatelier@netyou.fr"
       left=172.16.0.2
       leftsubnet=172.16.0.0/24,172.16.1.0/24
       leftcert = /etc/ipsec.d/certs/remote-vpn-gw.crt
       leftsendcert = always
      
       rightid="C=FR, ST=IleDeFrance, L=Paris, O=netyou, OU=netyou, CN=local-vpn-gw, N=local-vpn-gw, E=fabien.gatelier@netyou.fr"
       right=%any
       rightsourceip=51.158.101.103

cat /etc/ipsec.d/certs/remote-vpn-gw.crt 

 # Configure Network address translation to masquerade 51.158.101.103 to its own ip. 
 iptables -t nat -F POSTROUTING
 iptables -t nat -A POSTROUTING -s 51.158.101.103 -d 172.16.0.2 -j ACCEPT
 iptables -t nat -A POSTROUTING -s 51.158.101.103 -d 172.16.0.0/16 -o ens4 -j MASQUERADE
 
 # Set MSS size to prevent fragmentation issues
 iptables -t mangle -F FORWARD
 iptables -t mangle -A FORWARD -m policy --pol ipsec --dir in -p tcp -m tcp --tcp-flags SYN,RST SYN -m tcpmss --mss 1361:1536 -j TCPMSS --set-mss 1360
 iptables -t mangle -A FORWARD -m policy --pol ipsec --dir out -p tcp -m tcp --tcp-flags SYN,RST SYN -m tcpmss --mss 1361:1536 -j TCPMSS --set-mss 1360
 
 # remote vpnserver to only accept these services directed to it
 iptables -A INPUT -i lo -j ACCEPT
 iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
 iptables -A INPUT -p icmp --icmp-type 8 -m limit --limit 10/m  -j ACCEPT
 iptables -A INPUT -p icmp --icmp-type 11 -m limit --limit 10/m  -j ACCEPT
 iptables -A INPUT -p icmp --icmp-type 3 -m limit --limit 10/m  -j ACCEPT
 iptables -A INPUT -p tcp  --dport 22 -j ACCEPT
 iptables -A INPUT -p udp  --dport 500 -j ACCEPT
 iptables -A INPUT -p udp  --dport 4500 -j ACCEPT
 iptables -A INPUT -m limit --limit 10/m --limit-burst 15 -j LOG --log-prefix "firewall: "
 iptables -A INPUT -j DROP
 
 # remote vpnserver to only forward traffic for these services
 iptables -F FORWARD
 iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
 iptables -A FORWARD -p tcp -s 51.158.101.103 -d 172.16.0.0/24 --dport 22 -j ACCEPT
 iptables -A FORWARD -p tcp -s 51.158.101.103 -d 172.16.0.0/24 --dport 80 -j ACCEPT
 iptables -A FORWARD -p tcp -s 51.158.101.103 -d 172.16.0.0/24 --dport 443 -j ACCEPT
 iptables -A FORWARD -p tcp -s 51.158.101.103 -d 172.16.0.0/24 --dport 8080 -j ACCEPT
 
 iptables -A FORWARD -p tcp -s 51.158.101.103 -d 172.16.1.0/24 --dport 22 -j ACCEPT
 iptables -A FORWARD -p tcp -s 51.158.101.103 -d 172.16.1.0/24 --dport 80 -j ACCEPT
 iptables -A FORWARD -p tcp -s 51.158.101.103 -d 172.16.1.0/24 --dport 443 -j ACCEPT
 iptables -A FORWARD -p tcp -s 51.158.101.103 -d 172.16.1.0/24 --dport 8080 -j ACCEPT
 
 iptables -A FORWARD -p icmp --icmp-type 8 -m limit --limit 10/m  -j ACCEPT
 iptables -A FORWARD -p icmp --icmp-type 11 -m limit --limit 10/m  -j ACCEPT
 iptables -A FORWARD -p icmp --icmp-type 3 -m limit --limit 10/m  -j ACCEPT
 iptables -A FORWARD -m limit --limit 10/m --limit-burst 15 -j LOG --log-prefix "firewall: forward: "
 iptables -A FORWARD -j DROP
 
 # log outgoing connection from remote vpnserver 
 iptables -F OUTPUT
 iptables -A OUTPUT -m limit --limit 10/m --limit-burst 15 -j LOG --log-prefix "firewall: output: "

netfilter-persistent save 

vim /etc/sysctl.conf 

net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_forward = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.send_redirects = 0


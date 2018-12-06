# Leo1_Portfolio_2
**Group 3**\
Kiagias Konstantinos\
Loukas Athanasios

# Containers and scripting

In this assignment our goal is to create 2 unprivileged containers with enabled netwroking between them and the host.\
The first container named C1 will serve a simple .php web-server and display random numbers provided by the second container named C2.\
In the second container a script will produce those rundom numbers.

### Procedure
In order to create the unprivileged containers the first step is to create a default container configuration file.\
In this file we will specify id mappings,network setup and we will allow the unprivilledged user to use hosts network.\
The following commands executed following the guides: **LEO1 2018 Portfolio Two**\
and **https://help.ubuntu.com/lts/serverguide/lxc.html**

```bash
grep $USER /etc/subuid
grab $ USER / etc / subgid
mkdir -p ~/.config/lxc
echo "lxc.id_map = u 0 100000 65536" > ~/.config/lxc/default.conf
echo "lxc.id_map = g 0 100000 65536" >> ~/.config/lxc/default.conf
echo "lxc.network.type = veth" >> ~/.config/lxc/default.conf
echo "lxc.network.link = lxcbr0" >> ~/.config/lxc/default.conf
echo "$USER veth lxcbr0 2" | sudo tee -a /etc/lxc/lxc-usernet
```
After this step we created our containers with an image of alpine 3.4:
```bash
lxc-create -n C1 -t download -- -d alpine -r 3.4 -a armhf
lxc-create -n C2 -t download -- -d alpine -r 3.4 -a armhf
```
In the next step tmux proved to be really handy splitting the terminal session in to multiple windows.\
Starting containers
```bash 
lxc-start -n C1
lxc-start -n C2
```
#### Setting up C1
To modify our container first of all we need to run:
```bash
lxc-attach -n C1
```
Next step is to update and install all the required packages: 
```bash
apk update
apk add lighttpd php5 php5-cgi php5-curl php5-fpm
```
Php code for the web-server has to be added as index.php in the directory /var/www/localhost/htdocs/.\
We have to uncomment the include "mod_fastcgi.conf" located in /etc/lighttpd/lighttpd.conf and start the lighttpd service.
```bash
rc-update add lighttpd default
openrc
```

#### Setting up C2
As described earlier the second container will have to provide random numbers to C1.\
To generate the random numbers we wrote the script and chmod it in /bin/bash
The last step was to establish the byte stream between C1 and C2 using socat :
```bash
socat -v -v tcp-listen:8080,fork,reuseaddr exec:/bin/rng.sh
```
### Setup network bridge using lxc-net
Following the tutorial **https://angristan.xyz/setup-network-bridge-lxc-net/** we installed and configured dnsmasq-base in /etc/lxc/default.conf
```bash
lxc.network.type = veth
lxc.network.link = lxcbr0
lxc.network.flags = up
lxc.network.hwaddr = 00:16:3e:xx:xx:xx
```
and /etc/default/lxc-net
```bash 
USE_LXC_BRIDGE="true"
```

Prevent containers from getting dynamic ips was done in /etc/lxc/dhcp.conf and /etc/default/lxc-net:\
```bash
dhcp-host=C1,10.0.3.11
dhcp-host=C2,10.0.3.12
```
Load the configuration:
```bash
LXC_DHCP_CONFILE=/etc/lxc/dhcp.conf
systemctl restart lxc-net
lxc-stop -n C1 && lxc-start -n C1
lxc-stop -n C2 && lxc-start -n C2
```
### Route a host port to a container
Using the bridge containers can be accessed thorough the internet.
To map a hosts port to C1 we added the following rule to iptables 
```bash
iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 80 -j DNAT --to-destination 10.0.3.11:80
```
Now our container should be reachable through the 80 port using raspberry's pubic ip.

## Outcomes
Even though all of our settings and configurations seem to be correct we didn't manage to have access to the web-server.
Fetching random numbers from C2 was done correctly. Using curl C2:8080 we were able to receive the data stream.

### Troubleshooting
We checked that port forwarding was properly done using ``` iptables -t nat -L ```
We tried to re-set it using ``` iptables -t nat -F ```and 

```bash
iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 80 -j DNAT --to-destination 10.0.3.11:80
```
We used ```ifconfig -a```to check our host_nic.\
We used ```journalctl -xe --follow``` to fix container starting issues.\
In order to troubleshoot lxc-net issues we used:

```bash
sudo systemctl restart lxc-net
sudo systemctl status lxc-net
```







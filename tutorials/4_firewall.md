# Firewall Server install

After completing this tutural you should end up with a gateway server

## 1. Preparation

**Step 1:** Virtual Machine config - Firewall

You have to revert your machine to a virgin ubuntu server, if you did not create a snaptshot then take a clone of your initial ubuntu server.

Now change the network settings that it looks like this:

* 1st network card
    * Bridged with syntra network, this will be our WAN interface
* 2e network card
    * Attached to a private network which we will use for our merger.local

This is what it should look like:
![Syntra fw network][1]

Make sure you configire the linux network like this:

* set `eth0` to DHCP
* set `eth1` to static IP 192.168.25.40
    
**Step 2:** Virtual Machine config - DNS DHCP

Also start your DNS DHCP server and attach the first network card to the `merger.local` private lan
Discard the second network card.
make sure the gateway of your server points to `192.168.25.40`

**Step 2:** Let's test

Are you able to pin our gateway from the DNS DHCP server:

    root@dns-dhcp:~# ping -c1 192.168.25.40
    PING 192.168.25.40 (192.168.25.40) 56(84) bytes of data.
    64 bytes from 192.168.25.40: icmp_req=1 ttl=64 time=0.314 ms
    
    --- 192.168.25.40 ping statistics ---
    1 packets transmitted, 1 received, 0% packet loss, time 0ms
    rtt min/avg/max/mdev = 0.314/0.314/0.314/0.000 

So great you are able to ping our firewall.

Now let's try to ping google

    root@dns-dhcp:~# ping -c1 8.8.8.8
    PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
    
    --- 8.8.8.8 ping statistics ---
    1 packets transmitted, 0 received, 100% packet loss, time 0ms

As you can se we are unable to ping google, so our firewall / router is not configured yet.

## 2. Configuration

**Step 1:** Configure your linux to do ipv4 forwarding

By default any modern Linux distributions will have IP Forwarding disabled. This is normally a good idea, as most peoples will not need IP Forwarding, but if we are setting up a Linux router/gateway and maybe a VPN server (pptp or ipsec) then we will need to enable forwarding.

Open the file `/etc/sysctl.conf` and search for: `net.ipv4.ip_forward=1`
If you found it uncomment it.
If you can't find it just add it.

This will make sure next time you boot you will have ip forwarding enabled.

Now we have to apply the ip forwarding to our running system:

    $ sudo sysctl -p /etc/sysctl.conf
    net.ipv4.ip_forward = 1

Now you can verify if it is enabled:

    $ cat /proc/sys/net/ipv4/ip_forward
    1

**Step 2:** Test again

Now start a tcpdump and look what happens on your gateway when you now ping google

On the gateway:

    sudo tcpdump -i eth0 -n icmp

On your DNS DHCP server:

    root@dns-dhcp:~# ping -c1 8.8.8.8
    PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
    
    --- 8.8.8.8 ping statistics ---
    1 packets transmitted, 0 received, 100% packet loss, time 0ms


So it looks like you are still unable to ping, now look at your tcpdump, it will look like this:

    sysadmin@gateway:~$ sudo tcpdump -i eth0 -n icmp
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
    21:11:57.867811 IP 192.168.25.39 > 8.8.8.8: ICMP echo request, id 1467, seq 1, length 64

As you can see we are sending a packet with the source IP from our DHCP server to google.
This will not work since the syntra router does not know the network `192.168.25.0/24`

**Step 3:** Masquerade or SNAT behind the syntra IP

Now we have to 'hide' our private IP range (192.168.25.0/24) behind the address received from the syntra network.

We have 2 option:

* Use Source Natting (SNAT)
* Use Masquerading 

What you need is depends on how you configured your WAN interface.

* Use SNAT if you have a static IP configured on your WAN interface.
* Use Masquerading when your WAN interface has a DHCP address

In our case we have configure `eth0` to receive a DHCP addres from the syntra LAN.

    sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

This will tell to iptables that when traffic is send via eth0 you have to Masquerade


**Step 4:** Test again

Now start again tcpdump and look what happens on your gateway when you now ping google

On the gateway:

    sudo tcpdump -i eth0 -n icmp

On your DNS DHCP server:

    root@dns-dhcp:~# ping -c1 8.8.8.8
    PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
    64 bytes from 8.8.8.8: icmp_req=1 ttl=42 time=27.0 ms
    
    --- 8.8.8.8 ping statistics ---
    1 packets transmitted, 1 received, 0% packet loss, time 0ms
    rtt min/avg/max/mdev = 27.073/27.073/27.073/0.000 ms


Great! This worked !

Now look at your tcpdump again, what do you see now?

Great we have now routing enabled, but our firewall is still open in all directions.

**Step 5:** Close our firewall for INPUT

As you know the `INPUT` table contains all the rules which for traffic TOWARDS our firewall.
So for example your SSH session is considered as `INPUT`

We can allow established sessions to receive traffic:

    sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

When you receive a packet which has the state INVALID just drop the packet:

    sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP

The INVALID state means that the packet can't be identified or that it does not have any state.
Next we would like to allow ssh traffic if it comes from our LAN (eth1)

    sudo iptables -A INPUT -i eth1 -p tcp --dport 22  -j ACCEPT

Next we will allow [ICMP][2]

    sudo iptables -A INPUT -p icmp --icmp-type 0 -j ACCEPT
    sudo iptables -A INPUT -p icmp --icmp-type 8 -m limit --limit 2/s -j ACCEPT
    sudo iptables -A INPUT -p icmp --icmp-type 3/4 -j ACCEPT
    sudo iptables -A INPUT -p icmp --icmp-type 3/3 -j ACCEPT
    sudo iptables -A INPUT -p icmp --icmp-type 3/1 -j ACCEPT
    sudo iptables -A INPUT -p icmp --icmp-type 4 -j ACCEPT
    sudo iptables -A INPUT -p icmp --icmp-type 11 -j ACCEPT
    sudo iptables -A INPUT -p icmp -m limit -j LOG --log-prefix "ICMP/IN: "

Now that we have configured a basic setup let's change the default policy to DROP

    sudo iptables -P INPUT DROP

**Step 6:** Close our firewall for FORWARD

We can allow established sessions to receive traffic:

    sudo iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

When you receive a packet which has the state INVALID just drop the packet:

    sudo iptables -A FORWARD -m conntrack --ctstate INVALID -j DROP

Next we will allow [ICMP][2]

    sudo iptables -A FORWARD -p icmp --icmp-type 0 -j ACCEPT
    sudo iptables -A FORWARD -p icmp --icmp-type 8 -m limit --limit 2/s -j ACCEPT
    sudo iptables -A FORWARD -p icmp --icmp-type 3/4 -j ACCEPT
    sudo iptables -A FORWARD -p icmp --icmp-type 3/3 -j ACCEPT
    sudo iptables -A FORWARD -p icmp --icmp-type 3/1 -j ACCEPT
    sudo iptables -A FORWARD -p icmp --icmp-type 4 -j ACCEPT
    sudo iptables -A FORWARD -p icmp --icmp-type 11 -j ACCEPT
    sudo iptables -A FORWARD -p icmp -m limit -j LOG --log-prefix "ICMP/IN: "

Now that we have configured a basic setup let's change the default policy to DROP

    sudo iptables -P FORWARD DROP


  [1]: https://raw.github.com/netdata/syntra-linux/master/tutorials/img/syntra-fw.png
  [2]: http://eugene.oregontechsupport.com/articles/icmp.txt

# Firewall Server install

[[_TOC_]] 

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

**Step 3:** Network 

    


  [1]: https://raw.github.com/netdata/syntra-linux/master/tutorials/img/syntra-fw.png

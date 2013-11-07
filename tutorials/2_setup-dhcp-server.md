# DHCP Server

**Step 1:** Add an extra interface to your virtual machine

The syntra network already has a DHCP server, running a second one on the same network is a very bad idea.
Add an extra interface to your virtual machine.

**Step 2:** Installing dhcp server

    sudo apt-get install isc-dhcp-server
    
**Step 3:** Configure the DHCP server so it listens on the correct interface

Edit the file `/etc/default/isc-dhcp-server` and make sure `eth1` is added to the INTERFACES

    INTERFACES="eth1

**Step 4:** Adding a subnet to the config

The config file for the dhcp server is `/etc/dhcp/dhcpd.conf`
Now add a subnet config, here is an example:

    subnet 10.10.10.0 netmask 255.255.255.0 {
      range 10.10.10.100 10.10.10.200;
      option domain-name-servers 10.10.10.1;
      option domain-name "merger.local";
      option routers 192.168.25.40;
    }

Most options are self explaining

Now start the the DHCP server

    isc-dhcp-server restart

**Step 5:** Testing your setup

Boot up a linux client in the same network as `eth1` from our DHCP server.

While the client boots Watch the logfile via syslog: `tail -f /var/log/syslog`

    Nov  7 21:01:22 dns-dhcp dhcpd: DHCPDISCOVER from 08:00:27:91:18:57 via eth1
    Nov  7 21:01:23 dns-dhcp dhcpd: DHCPOFFER on 10.10.10.100 to 08:00:27:91:18:57 (linux-syntra-client-1) via eth1
    Nov  7 21:01:23 dns-dhcp dhcpd: DHCPREQUEST for 10.10.10.100 (10.10.10.1) from 08:00:27:91:18:57 (linux-syntra-client-1) via eth1
    Nov  7 21:01:23 dns-dhcp dhcpd: DHCPACK on 10.10.10.100 to 08:00:27:91:18:57 (linux-syntra-client-1) via eth1

As you can see the machine with hostname `linux-syntra-client-1` received the IP address `10.10.10.100`

**Step 6:** Enabling Dynamic DNS updates from the DHCP server

As you could see in the log file the DHCP server knows the name of the client requesting the an IP address.
What we could do is automaticly add a forward lookup and reverse lookup to the DHCP server.




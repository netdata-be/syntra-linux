# DHCP Server

**Step 1:** Add an extra interface to your virtual machine

The syntra network already has a DHCP server, running a second one on the same network is a very bad idea.
Add an extra interface to your virtual machine.

**Step 2:** Installing dhcp server

    sudo apt-get install isc-dhcp-server
    
**Step 3:** Configure the DHCP server so it listens on the correct interface

Edit the file `/etc/default/isc-dhcp-server` and make sure `eth1` is added to the INTERFACES

    INTERFACES="eth1"

**Step 4:** Adding a subnet to the config

The config file for the dhcp server is `/etc/dhcp/dhcpd.conf`
Now add a subnet config, here is an example:

    subnet 192.168.25.00 netmask 255.255.255.0 {
      range 192.168.25.100 192.168.25.200;
      option domain-name-servers 192.168.20.1;
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
    Nov  7 21:01:23 dns-dhcp dhcpd: DHCPOFFER on 192.168.20.100 to 08:00:27:91:18:57 (linux-syntra-client-1) via eth1
    Nov  7 21:01:23 dns-dhcp dhcpd: DHCPREQUEST for 192.168.20.100 (192.168.20.1) from 08:00:27:91:18:57 (linux-syntra-client-1) via eth1
    Nov  7 21:01:23 dns-dhcp dhcpd: DHCPACK on 192.168.20.100 to 08:00:27:91:18:57 (linux-syntra-client-1) via eth1

As you can see the machine with hostname `linux-syntra-client-1` received the IP address `192.168.20.100`

### Dynamic DNS updates

As you could see in the log file the DHCP server knows the name of the client requesting the an IP address.
What we could do is automatically add a forward lookup and reverse lookup to the DHCP server.

**Step 1:** Generating a key

The communication between DHCP server and DNS server is secured using a cryptographic key.

Generate a key file:

    sudo rndc-confgen -r /dev/urandom -a -c /etc/bind/ddns.key -k DDNS_KEY
    sudo chown root:bind /etc/bind/ddns.key
    sudo chmod 640 /etc/bind/ddns.key

In a production environment you would never use `/dev/uramdom` because the ouput contain less entropy then `/dev/random`.

**Step 2:** Adding the key to the DNS config

Make sure your `/etc/bind/named.conf.local` looks like this:

    include "/etc/bind/ddns.key";
    
    zone "merger.local" {
      type master;
      allow-update { key DDNS_KEY; };
      file "/etc/bind/zones.local/db.merger.local";
    };

    zone "25.168.192.in-addr.arpa" {
      type master;
      allow-update { key DDNS_KEY; };
      file "/etc/bind/zones.local/db.25.168.192.in-addr.arpa";
    };

The changes are:

 - include "/etc/bind/ddns.key";
 - allow-update { key DDNS_KEY; };
 
**Step 3:** Adding the key to the DHCP folder

The DHCP server needs the key as wel, so copy the directory:

    sudo cp /etc/bind/ddns.key /etc/dhcp/
    sudo chown root:root /etc/dhcp/ddns.key
    sudo chmod 640 /etc/dhcp/ddns.key
    
**Step 4:** Preparing the DHCP server for DDNS

Add the following to your `/etc/dhcp/dhcpd.conf`

    ddns-updates on;
    ddns-update-style interim;
    update-static-leases on;
    
***option domain-name***
This options specifies the domain name, which is also used for DDNS.

***ddns-update-style***
This option should always be interim. The only other option is adhoc, but that one is outdated .

***update-static-leases***
By default the DHCP-Server doesn't update the DNS entries of static leases. If you want it to update them, you need to set this option to on. It can be that this causes some problems, that's why the manpage of dhcpd.conf doesn't recommend the use of it. If you experience problems, turn it off, but then you have to configure these hosts statically not only for DHCP, but also for DNS.

**Step 5:** Adding zone config

Also add the following to your config: `/etc/dhcp/dhcpd.conf`

    include "/etc/dhcp/ddns.key";

    zone merger.local. {
      primary 127.0.0.1;
      key DDNS_KEY;
    }
    
    zone 25.168.192.in-addr.arpa. {
      primary 127.0.0.1;
      key DDNS_KEY;
    }

Now also add the following to your subnet

    ddns-domainname "merger.local.";
    do-forward-updates true;

**Step 6:** reload your DHCP server

    sudo service isc-dhcp-server restart
    
**Step 7:** Test your config



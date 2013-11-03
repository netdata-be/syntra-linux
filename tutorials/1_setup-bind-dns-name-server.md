# Bind DNS server install

**Step 1:** Installing bind

    sudo apt-get install bind9
    
**Step 2:** Configuring bind

There are 2 options for bind to resolve records:

1. Resolve the record self by staring to query the root DNS servers
2. Forward the query to an other DNS server and ask for a recursive lookup

You are going to use the syntra DNS server to do the actuall lookup.

Add the following to the file  `/etc/bind/named.conf.options`

    forwarders {
      172.28.0.20;
    };
    
Now reload your DNS server:

    sudo /etc/init.d/bind9 reload

**Step 3:** Test your DNS server

You are going to use tcpdump to verify the name resolving.

    sudo tcpdump -i eth0 -n host 172.28.0.20 and port 53

Now open a second console and query your server:

    dig +short www.google.com @localhost

You should have received an answer from your bind nameserver.

I you now take a a look at the first console where the tcpdump is running you will notice a packet started from your virtual server towards the syntra DNS server.

**Step 4:** Use your DNS server

Since your network interface card is booted with DHCP your `/etc/resolv.conf` is managed by a package called `resolvconf`.

This maens that `/etc/resolv.conf` is a symlink to `/run/resolvconf/resolv.conf`
In order to make modification to the resolv.conf you have to remove the symlink:

    sudo rm /etc/resolv.conf

Now re-create the file `/etc/resolv.conf` with the following content:

    nameserver 127.0.0.1
    search merger.local
    

**Step 5:** Adding merger.local to bind

We are going to configure an forward lookup zone for our `merger.local` domain

Add the following to the file: `/etc/bind/named.conf.local`

    zone "merger.local" {
      type master;
      file "/etc/bind/zones.local/db.merger.local";
    };
    
Now you will have to create the zonefile called `/etc/bind/zones.local/db.merger.local`

    sudo mkdir /etc/bind/zones.local
    
Now create the zone file, here is an example on how it could look like:

    $TTL  3600      ; Zone TTL default = 1 hour.
    @ IN  SOA  ns1.merger.local. noc.netdata.be.  (
      2013101103  ; Serial (YYYYMMDD##)
      86400       ; refresh (1 day)
      900         ; retry( 15 min)
      2419200     ; expire( 4 weeks)
      10800 )     ; Negative Cache(10hours40minutes)
    ;
    @       IN      NS      ns1.merger.local.
    
    $ORIGIN merger.local.
    
    .            IN MX 10 email
    ns1	         IN A    127.0.0.1
    session      IN A    192.168.25.37 


The top part is the [SOA][1] of the zone file.
Next follows actual records.

**Step 6:** Adding reverse lookup zone

Now we are going to add the reverse lookup zone for the IP's used in our network.
Append the following to the file `/etc/bind/zones.local/db.merger.local`

    zone "25.168.192.in-addr.arpa" {
      type master;
      file "/etc/bind/zones.local/db.25.168.192.in-addr.arpa";
    };

Note that the zone name is the reverse of our IP adresses.

And create a zone file containing the reverse lookup records:

`/etc/bind/zones.local/db.25.168.192.in-addr.arpa`

    $TTL  3600      ; Zone TTL default = 1 hour.
    @ IN  SOA  ns1.merger.local. noc.netdata.be.  (
      2013101103  ; Serial (YYYYMMDD##)
      86400       ; refresh (1 day)
      900         ; retry( 15 min)
      2419200     ; expire( 4 weeks)
      10800 )     ; Negative Cache(10hours40minutes)
    ;
    @       IN      NS      ns1.merger.local.
    
    $ORIGIN 25.168.192.in-addr.arpa.
    
    37       IN PTR session.merger.local.
    38	     IN PTR deployment.merger.local.
    
Note: Altough not required I would advise to add PTR records in a ascending order,
This makes it really easy to see what the next available IP would be.

**Step 6:** Adding forward zones to the Windows AD

Append the following to the file `/etc/bind/zones.local/db.merger.local`
    
    zone "contoso.local" {
      type forward;
      forwarders {
        192.168.20.21;
      };
    };
    
    zone "20.168.192.in.addr.arpa" {
      type forward;
      forwarders {
        192.168.20.21;
      };
    };

This means if you request a record mentioned in the zones contoso.local or 20.168.192.in.addr.arpa your query will be forwarded to the windows AD -in this case the IP 192.168.20.21 -

**Step 7:** Reload your bind service and test



  [1]: http://www.zytrax.com/books/dns/ch8/soa.html



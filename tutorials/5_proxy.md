# Proxy Server install

After completing this tutorial you should end up with:

 - a forward proxy server
 - a log analyzing tool to review your HTTP requests

The following will NOT be implemented (It should be easy extend the proxy server)

 - Antivirus Checking
 - URL whithlist / blacklists
 - Bandwith managment / Throtteling
 
## 1. Preparation

**Step 1:** In order to get started you should have completed the firewall tutorial

**Step 2:** Install squid

Later is this tutorial we will use squid3 together with SSL.
Unfornatly ubunut did not compile squid using the SSL options flag.

So we are going to this manually

The following commands will install the required software do the compilation of squid:

    sudo apt-get install build-essential fakeroot devscripts gawk gcc-multilib dpatch libssl-dev
    
    sudo apt-get build-dep squid3
    
    sudo apt-get build-dep openssl
    
    cd /usr/src
    
    sudo apt-get source squid3

We need to modify the build script. This will make it so when we go to configure the source files it will include SSL support.

    vi squid3*/debian/rules

Add `--enable-ssl` under the `DEB_CONFIGURE_EXTRA_FLAGS` section.

    ...
    DEB_CONFIGURE_EXTRA_FLAGS := --datadir=/usr/share/squid3 \
    		--sysconfdir=/etc/squid3 \
    		--mandir=/usr/share/man \
    		--with-cppunit-basedir=/usr \
    		--enable-inline \
            --enable-ssl \
    ...

Now let's compile and build a debian package again:

    cd squid3*
    debuild -us -uc -b
    
**NOTICE:** Here we use debuild that will automatically Configure, Make, and create an installable DEB package.

After this has been completed a DEB package will appear in the parent directory.

Now install the freshly build squid packages:

    cd ..
    apt-get install squid-langpack ssl-cert
    sudo dpkg -i squid3-common_3.1.19-1ubuntu3.12.04.2_all.deb
    sudo dpkg -i squid-common_3.1.19-1ubuntu3.12.04.2_all.deb
    sudo dpkg -i squid3_3.1.19-1ubuntu3.12.04.2_amd64.deb
    sudo dpkg -i squid_3.1.19-1ubuntu3.12.04.2_amd64.deb

Verify the instalation:

    squid3 -v | grep -q  enable-ssl && echo "Yes, you have openSSL Support" || echo "Whooo, looks like you dont have SSL enable"

## 2. Basic Configuration

***Step 1:*** Initializing Squid Cache

Ensure squid is not running first!

    sudo service squid3 stop

Initialize cache…

    sudo squid3 -z
 
***Step 2:*** Basic config file

Edit the file `/etc/squid3/squid.conf` remove all existing content and add the following:

    # ACL List
    acl manager proto cache_object
    acl localhost src 127.0.0.1/32 ::1
    acl merger_network src 192.168.25.0/24
    
    # Ports allowed through Squid
    acl Safe_ports port 80		# http
    acl Safe_ports port 21		# ftp
    acl Safe_ports port 443		# https
    
    acl SSL_ports port 443      # https
    
    acl SSL method CONNECT
    acl CONNECT method CONNECT
    
    # allow/deny
    http_access allow manager localhost
    http_access deny manager
    http_access allow merger_network
    http_access deny !Safe_ports
    http_access deny CONNECT !SSL_ports
    http_access allow localhost
    http_access deny all
    
    # proxy ports
    http_port 192.168.25.40:3128
    http_port 192.168.25.40:8080 intercept
    
    # caching directory
    cache_dir ufs /var/spool/squid3 2048 16 128
    cache_mem 1024 MB
    
    coredump_dir /var/spool/squid3
    refresh_pattern ^ftp:		1440	20%	10080
    refresh_pattern ^gopher:	1440	0%	1440
    refresh_pattern -i (/cgi-bin/|\?) 0	0%	0
    refresh_pattern (Release|Packages(.gz)*)$      0       20%     2880
    refresh_pattern .		0	20%	4320

Let’s step through this:

- **acl** = this tells squid which IP address and/or hosts to assign a certain Access List. For example home_network is any IP sourcing from 192.168.0.0/24 network.
- **Safe_ports** = tells squid which ports are allowed through the proxy, we have defined only FTP, HTTP and HTTPS.
- **SSL_ports** = tells squid which port is allowed when making an SSL connection
- **http_access** = defines which Access Lists (acl) are allowed to connect to the proxy.
- **http_port** = binds an IP and port of the Proxy server to listen for requests. We have two because one will be used for the Transparent Proxy, the other for browsers who explicit configure their browsers to connect to the proxy server.
- **intercept** = is required for transparency to work.
- **cache_dir** = defines where squid should store cached static files, how much space should it consume, and for how long. ufs is the type of storage system, /home/user/squidcache/ is the directory to use, 2048 defines 2048MB of capacity, 16 defines number of first-level subdirectories, and 128 defines second-level subdirectory. For more info, see here.
- **cache_mem** = defines how much memory should be allocated for Squid caching.
- = used to assign expirations, storage, retention, etc of static files. More info, here.

***Step 3:*** Open Firewall

First allow our proxy port:

    sudo iptables -A INPUT -i eth1 -p tcp -m tcp -m multiport --dports 3128,8080,8443 -j ACCEPT -m comment --comment "Allow Access to Squid proxy"

Next we also would like to support transparant proxy.
There are multiple ways to do transparant proxy but this is the oldest.

    sudo iptables -A FORWARD -s 192.168.25.0/24 -p tcp -m tcp -m multiport --dports 80,443 -j ACCEPT -m comment --comment 'Allow incomming connections HTTP / HTTPS connection'

This will accept packets for port 80 and port 443 which should go to the internet
BUT the following will make sure the requests are NOT forwarded to the internet YET.

    sudo iptables -t nat -A PREROUTING -i eth1 -p tcp -m tcp --dport 80  ! -d 192.168.25.0/24 -j DNAT --to-destination 192.168.25.40:8080
    sudo iptables -t nat -A PREROUTING -i eth1 -p tcp -m tcp --dport 443 ! -d 192.168.25.0/24 -j DNAT --to-destination 192.168.25.40:8443
    sudo iptables -t nat -A PREROUTING -i eth1 -p tcp -m tcp --dport 80 ! -d 192.168.25.0/24 -j REDIRECT --to-ports 8080
    sudo iptables -t nat -A PREROUTING -i eth1 -p tcp -m tcp --dport 443 ! -d 192.168.25.0/24 -j REDIRECT --to-ports 8443

This will redirect your traffic to the squid server.

***Step 4:*** Save firewall rules 

    sudo service iptables-persistent save

## 3. SSL inspection

This next section allows your Squid proxy server to intercept SSL connection made from your clients. 

***Warning!*** Doing so will most likely look like a man-in-the-middle attack. Clients will be connecting to your proxy server when trying to go to SSL protected sites, thus violating the SSL transaction. For example, a client opens up a connection to https://mail.google.com. This connection will be intercepted by the proxy server

I would like to also note that you should consider the behaviour you are trying to achieve with having SSL connections proxy’d through Squid. The nature of SSL does not allow us to easliy perform Proxy features, such as caching, content filter, content manipulation, etc.

Therefore, if you are setting up SSL passthrough with squid, then you are effectively doing the same thing that a router would. In conclusion, the only reasons I can think of for enabling SSL Interception would be for auditing and monitoring purposes.

Squid actually does a [MITMA][1] on the connection that is:

 1. Generate a matching certificate signed by an internal (SELF signed Certificate Authority) 
 2. Build an SSL tunnel between your proxy and your CLIENT using this certificate
 3. Client trust the internal Certificate authority so no SSL warnings...
 4. Proxy builds SSL tunnel between proxy and the actual destination (e.g. : https://mail.google.com)

If you don't know how TLS/SSL handshake is done look that this picture:
![ssl-handshake][2]

***Step 1 :*** Creating a Self Signed ROOT certificate

    sudo mkdir /etc/squid3/certs
    
    sudo openssl req \
      -x509 \
      -nodes \
      -days 3650 \
      -newkey rsa:2048 \
      -keyout /etc/squid3/certs/root-ca.pem \
      -out /etc/squid3/certs/root-ca.pem \
      -subj '/C=BE/ST=Antwerp/L=Mechelen/O=Merger.Local/CN=Syntra AB RootCA/emailAddress=administrator@merger.local'
    
    
Here is an explanation of all options:

 - **req** = certificate request and certificate generating utility
 - **x509** = outputs a self signed certificate instead of a certificate request
 - **nodes** = When the private key is created no passphrase is set
 - **days** = specifies the number of days to certify the certificate
 - **newkey** = generate a new private key RSA with a key size of 2048 bits
 - **keyout** = output the private key to this file
 - **out** = output the self signed certificate to this file
 - **subj** = Provide some defaults to the naming, in this case we are create a certificate with the name "Syntra AB RootCA"

***Step 2:*** Enabling SSL-bump

edit again the squid config file `/etc/squid3/squid.conf`
Add the following…

    http_port 192.168.25.40:8443 ssl-bump cert=/etc/squid3/certs/root-ca.pem generate-host-certificates=on dynamic_cert_mem_cache_size=4MB
    
    # SSL bumping - This means Do a MITMA
    always_direct allow all
    ssl_bump allow all

***Step 3:*** Exporting the RootCA and import in your browser

    openssl x509 -in /etc/squid3/certs/root-ca.pem -out /tmp/root-ca_to-import.pem
    
Now copy the file `/tmp/root-ca_to-import.pem` to your client and import this certificate in your browser.



## 4. Squid Log Analyze

`Squid Analyzer` parses Squid proxy access log and reports general statistics about hits, bytes, users, networks, top URLs, and top second level domains. Statistic reports are oriented toward user and bandwidth control.

***Step 1 :*** Download the source

The latest version of Squid Analyzer now is version 5.3. 
You can download it from here. After that, you can extract the source file.

    tar -zxf squidanalyzer-5.3.tar.gz

***Step 2 :*** Install the software

    cd squidanalyzer-5.3/
    perl Makefile.PL
    make
    sudo make install

***Step 3 :*** Install apache

    sudo apt-get install apache2
    
***Step 4 :*** Configure apache

First we want to make sure that apache is only listening on `eth1` 

Edit `/etc/apache2/ports.conf` and change the `Listen` paramenter to this:

    Listen 192.168.25.40:80
    
Next we have to disable the default apache config:

    sudo a2dissite default
    
Now create the following file : `/etc/apache2/sites-available/squidanalyzer`

    <VirtualHost *:80>
      ServerAdmin webmaster@localhost
    
      DocumentRoot /var/www
      Alias /squidreport /var/www/squidanalyzer
    
      <Directory /var/www/squidanalyzer>
        Options -Indexes FollowSymLinks MultiViews
        AllowOverride None
        AllowOverride None
        Order deny,allow
        Allow from all
      </Directory>
    
    </VirtualHost>

Now enable this new virtualhost

    a2ensite squidanalyzer

Reload apache2

    service apache2 restart

***Step 5 :*** Open up firewall

    sudo iptables -A INPUT -i eth1 -p tcp -m tcp --dport 80 -j ACCEPT -m comment --comment "Allow Access to squidanalyzer"
    
    sudo service iptables-persistent save

***Step 6 :*** First log collection

Just run the following command:

    /usr/local/bin/squid-analyzer 

***Step 7 :*** Cron job

Add the following cron job using this command: `sudo crontab -e -u root`

    # SquidAnalyzer log reporting daily
    0 2 * * * /usr/local/bin/squid-analyzer > /dev/null 2>&1

This will collect all logs and generate the reports


  [1]: http://en.wikipedia.org/wiki/Man-in-the-middle_attack
  [2]: https://raw.github.com/netdata/syntra-linux/master/tutorials/img/tls-handshake.png

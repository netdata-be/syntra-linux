# E-mail Server install

Create a new virtual machine with the following specs:

* Ubuntu Linux 12.04 LTS
* Minimum 1500Mb RAM
* 15GB disk space


***Step 1 :*** Install Dependencies

These packages will be necessary for Zimbra to be installed on your system.

    sudo apt-get install netcat libidn11 libpcre3 libgmp3c2 libexpat1 libstdc++6 libperl5.14 sysstat sqlite3 pax

***Step 2 :*** Download Zimbra ZCS

    cd /tmp
    wget http://files2.zimbra.com/downloads/8.0.6_GA/zcs-8.0.6_GA_5922.UBUNTU12_64.20131203103702.tgz 
    
This is a quite big file (742Mb) if you are doing this during this during the session at syntra you could also ask to instructor to provide you the tarbal.

***Step 3 :*** Extracting the tarbal

    tar -xzvf zcs-8.0.6_GA_5922.UBUNTU12_64.20131203103702.tgz
    cd zcs*

***Step 4 :*** Installing Zimbra

    sudo ./install.sh

Press Y and hit enter to agree to the license agreement.
Now, if you installed all the dependencies at the start of this tutorial, you should have everything you need!
If you aren’t sure what you want to install, then just install the items as suggested.

**zimbra-ldap** - Yes
**zimbra-logger** – Yes
**zimbra-mta** - Yes
**zimbra-snmp** - Yes
**zimbra-store** – Yes
**zimbra-apache** - Yes
**zimbra-spell**  - Yes
**zimbra-memcached** - No
**zimbra-proxy** - No

When asked if you want to continue, press Y and hit enter to proceed with the installation.

The Zimbra installer will take care of extracting and installing the packages for you.  This part of the process might take some time, depending on the speed of your machine.

In my case it took almost 40minutes

***Step 5 :*** Basic CLI configuration

Go through all of the items on the configuration and make sure they are what you want. 

The most important at this stage is the admin password, make sure to change it.

When you’re satisfied with all of your settings, press `S` and hit enter to write your settings to the configuration file.   The default file location is fine.   Then press `A` and hit enter to apply your settings and start the server!

Zimbra will ask you to confirm the changes to your system.  Type `Yes` and hit enter.

Zimbra will now run through an installation procedure, which may take a few minutes, depending on the speed of your machine.

If all went well, Zimbra is now installed on your server.  If it didn’t, you will be given a log file location, where you can look and see what might have gone wrong.

***Step 6 :***  Accessing Your Zimbra Server

You can login to your Zimbra accounts for webmail access at your Zimbra server’s IP or DNS on the standard https port 443.  You can access your Administration Console over HTTPS using port 7071.

Point your web browser to https://your.zimbra.server.ip:7071/

The Zimbra setup procedures in the web interface are very straightforward, and are up to you to play with on your own now that Zimbra is installed and working!


Still todo by WDH:

* Configuring incoming e-mail + add test procedure
* Configring outgoing e-mail + add test procedure


Things which could be done by the students:

* Add LDAP (use your samba install) authentication to Zimbra




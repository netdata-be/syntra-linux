# Samba 4 server install

**Step 1:** Install the Samba 4 Packages

    apt-get install samba4

The installation will throw out an error and apt will set the package to half installed. As the error isn’t relevant to us, we have to fix the package by manually setting the package to installed.

Edit `/var/lib/dpkg/status` and search for "`Package: samba4`"
Replace `half-configured` with `installed`
Now we are going to build the Active Directory Domain:

    rm /etc/samba/smb.conf
    /usr/share/samba/setup/provision \
    --realm=merger.local \ 
    --domain=MERGER \
    --adminpass='SyntraAB123' \
    --server-role=dc

This will set up all stuff needed for running a Domain (LDAP, Kerberos, …)

Next step is to start Samba:

    initctl start samba4

**Step 2:** Testing out our installation

    apt-get install samba4-clients
    smbclient -L localhost -U%

The last command should display the currently defined and served shares on the server. Should look something like:

    Sharename       Type       Comment
    ---------       ----       -------
    netlogon        Disk
    sysvol          Disk
    IPC$            IPC        IPC Service

## Bind Name Server

Active Directory uses DNS to discover a huge amount of services, mostly using `SRV records`
In the previous tutorial we already installed, and configured bind.

Therefore we have to undo some of the config we already did.
This is because DNS will mainly be managed by samba using 

**Step 1:** Adapt the AppArmor configuration

Edit `/etc/bind/named.conf.local` and add the following line at beginning of the line:

    include "/var/lib/samba/private/named.conf"

**Step 2:** Adapt the AppArmor configuration

As Ubuntu is securing it’s services using [AppArmor][1] we need to make sure that Bind has the rights to access the files provided by Samba.

Edit `/etc/apparmor.d/usr.sbin.named` and append the following entries:

    /var/lib/samba/private/** rkw,
    /var/lib/samba/private/dns/** rkw,
    /usr/lib/x86_64-linux-gnu/samba/bind9/** rm,
    /usr/lib/x86_64-linux-gnu/samba/gensec/** rm,
    /usr/lib/x86_64-linux-gnu/ldb/modules/ldb/** rm,
    /usr/lib/x86_64-linux-gnu/samba/ldb/** rm,


Now reload the configuration to take effect:

    /etc/init.d/apparmor reload

**Step 3:** Start and test Bind

Run the following command to start Bind:

    /etc/init.d/bind9 restart

To make sure that everything worked as expected, run the following commands and watch their output. It should return a result on every command:

    dig -t SRV _ldap._tcp.merger.local. @localhost

**Step 4:** Allow dynamic DNS updates

We want our clients to be able to update their DNS entries automatically. Edit `/etc/bind/named.conf.options` and append the following line:

    tkey-gssapi-keytab "/var/lib/samba/private/dns.keytab";

## Kerberos

**Step 1:** Install the Kerberos Utilities

    apt-get install krb5-user

When asked for the default realm, enter merger.local and as host take the hostname you used. 
Test out if Kerberos works by executing:

    kinit administrator@MERGER.LOCAL

The Domain Name needs to be written in UPPERCASE letters. If the command succeeds, run the following command to check if we have gotten a kerberos ticket:

    klist -e

## Network Time Protocol

As Samba provides the correct time to it’s domain members we want to make sure that our host has the correct time. We do so by installing and configuring NTP to retrieve the time from internet time servers.

**Step 1:** Install NTP

    apt-get install ntp

**Step 2:** Configure NTP

do a initial time setup:

    service ntp stop
    ntpdate -B ntp.belnet.be
    service ntp start

Check if everything works with:

    ntpq -p

## Other configuration items and Troubleshooting

**Joining the Domain**

Make sure that you use uppercase letters, like ‘DEMO.LOCAL’ as the domain name

**Testing the AD**

Run ‘dsa.msc’ on your Windows client (after you installed the Windows Remote Server Administration Tools)

If something did not work as expected (Domain not available), make sure that your DNS resolution works smooth.

**Creating shares**

To create shares you need to perform the following actions:

mkdir /data/global
chmod 777 /data/global
Then add an entry to `/etc/samba/smb.conf`:

    [global]
    comment = Global share for all users
    path = /data/global
    read only = No

Restart samba:

    initctl restart samba4

**Adding users**

When adding new uses, set their homedirectory to

    \\<HOSTNAME OF SAMBA SERVER>\users\

The directory will be created automatically.

**Adding new DNS entries**

Use the DNS Snap-In in the Management Console

**Error while copying**

If you copy files from a windows system to samba and get something like `Not enough memory`, this could be because of NTFS Streams within the files (Hidden Metadata). You can
remove them with the tool [streams][2].

and executing the following command:

    streams -s -d C:\data
    Permission problems

If you have problems with access to files created by different users (even if the permissions look correct), append the following in `/etc/samba/smb.conf` (in the share section):

    directory mask = 0777
    create mask = 0777

and restart samba:

    service samba4 restart


  [1]: http://en.wikipedia.org/wiki/AppArmor
  [2]: http://technet.microsoft.com/de-de/sysinternals/bb897440

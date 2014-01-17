## Open Virtual Desktop

**Step 1:** Create 3 Virtual Machines

 - Hostname: `ovd-session`
     - Ubuntu 12.04 LTS
     - IP: 192.168.25.50
 - Hostname: `ovd-linux-1`
     - Ubuntu 12.04 LTS
     - IP: 192.168.25.60
 - Hostname: `ovd-windows-1`
     - Microsoft® Windows® Server 2003/2008/2008 R2
     - IP: 192.168.25.61

All three machines must be on the `LAN` part of your `MERGER.LOCAL` domain
So all these machines will have your firewall as gateway
     
![OVD setup][1]

**Step 2:** Adding the ulteo Open Virtual Desktop repository

On both the session manager and on the app server (ovd-linux-1) do the following
Edit the `/etc/apt/sources.list.d/ovd.list` file and add these lines:

    deb http://archive.ulteo.com/ovd/3.0/ubuntu lucid main
        
Update the package database:

    sudo apt-get update

GPG errors given by the previous command will be fixed in the next installation steps. They won't prevent the installation to succeed.

    sudo apt-get install ulteo-keyring
    sudo apt-get update

## Session Manager installation and configuration

Package installation

The Session Manager is a LAMP (Linux Apache MySQL PHP) system and can be used on an exising LAMP server.

**Step 1:**  Install the `mysql-server` package:

    sudo apt-get install mysql-server

A password for `root` will be asked.

**Step 2:**  Now log in mysql and create a database:

    mysql -u root -p -e 'create database ovd'

**Step 3:** Install the ulteo-ovd-session-manager package:

    sudo apt-get install ulteo-ovd-session-manager
    
* The installer asks for an admin login

![admin login screenshot][2]

* And a password:

![Admin password][3]

* which has to be confirmed:

![comfirm admin pwd][4]

* According to the Archictecture documentation, a Susbsystem archive can be installed on the SM to simplify the Application Servers' deployements.

![enter image description here][5]

**Step 4:** Configuration

The first step is to go to `http://192.168.25.50/ovd/admin` and authenticate yourself with the login and password you provided during installation.
![admin login][6]

For the first time you log in, the system detects that it is not well configured so you are redirected to a basic setup page which will save a default configuration.

You have to set the MySQL configuration. For instance, if you install MySQL on the same host as described previously, here is the configuration:

![admin config init][7]

Then, you should be redirected to the main page:

![admin main page][8]

## Application Server and File Server installation (using Subsystem)


  [1]: https://raw2.github.com/netdata/syntra-linux/master/tutorials/img/ovd-setup.png
  [2]: https://raw2.github.com/netdata/syntra-linux/master/tutorials/img/sm_install_admin_login.png
  [3]: https://raw2.github.com/netdata/syntra-linux/master/tutorials/img/sm_install_admin_password.png
  [4]: https://raw2.github.com/netdata/syntra-linux/master/tutorials/img/sm_install_admin_confirm_password.png
  [5]: https://raw2.github.com/netdata/syntra-linux/master/tutorials/img/sm_install_chroot_location.png
  [6]: https://raw2.github.com/netdata/syntra-linux/master/tutorials/img/sm_www_admin_login.png
  [7]: https://raw2.github.com/netdata/syntra-linux/master/tutorials/img/sm_install_admin_config_init.png
  [8]: https://raw2.github.com/netdata/syntra-linux/master/tutorials/img/sm_admin_main.png

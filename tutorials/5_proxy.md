# Proxy Server install

After completing this tutural you should end up with a forward proxy server

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

    sudo dpkg -i squid_3.1.19-1ubuntu3.12.04.2_amd64.deb
    sudo dpkg -i squid3_3.1.19-1ubuntu3.12.04.2_amd64.deb
    sudo dpkg -i squid-common_3.1.19-1ubuntu3.12.04.2_all.deb
    sudo dpkg -i squid3-common_3.1.19-1ubuntu3.12.04.2_all.deb

Verify the instalation:

    squid3 -v | grep -q  enable-ssl && echo "Yes, you have openSSL Support" || echo "Whooo, looks like you dont have SSL enable"

## 2. Basic Configuration

## 3. SSL inspection







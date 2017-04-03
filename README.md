# PXE Boot

This project sets up a PXE/TFTP environment using Virtualbox and Ubuntu 16.04.

## Setup VirtualBox PXE Environment

All of my development will be done using MacOS Sierra.

### Prerequisites

#### Install the Following

* VirtualBox >= 5.1.14
* VirtualBox Extension Pack
* Vagrant >= 1.9.3
    * vagrant-hostmanager
    * vagrant-vbguest
    * vagrant-share

NOTE: VirtualBox will only TFTP from the private net of 10.0.2.x.  Therefore, you must include 2 adapters: NAT and Host-only.  More details to follow.

Once Vagrant is installed be sure to install the plugins:

```bash
./install_vagrant_plugins.sh
```

#### Enable the Hosts' Apache HTTP Server

The default `DocumentRoot` for Apache on MacOS is `/Library/WebServer/Documents`.  Let's own this file for our user.

```bash
sudo chown -R $USER /Library/WebServer/Documents
rm -rf /Library/WebServer/Documents/*
```

We need to enable directory browsing for the `DocumentRoot`.  Edit the `/etc/apache2/httpd.conf` and add `Indexes` to the `Options` line in the `DocumentRoot` stanza:

```
...
DocumentRoot "/Library/WebServer/Documents"

<Directory "/Library/WebServer/Documents">

    Options Indexes FollowSymLinks Multiviews
    MultiviewsMatch Any
...
```

```bash
sudo vi /etc/apache2/httpd.conf
```

Now start or restart Apache:

```bash
sudo apachectl start|restart
```

Browse to this [address](http://localhost) to see if it worked.

## Prepare the HTTP Directory

Download the Ubuntu 16.04 ISO:

```bash
wget http://releases.ubuntu.com/16.04.2/ubuntu-16.04.2-server-amd64.iso
```

NOTE: You will be unable to directly mount this ISO on MacOS due it being [hybrid](https://lists.ubuntu.com/archives/ubuntu-devel/2011-June/033495.html) (DVD/USB) ISO which can be directly dd'ed to a USB drive and booted.

```bash
hdiutil attach -nomount ubuntu-16.04.2-server-amd64.iso

# Take not of which 'disk' is mounted, use in the mount command below

mkdir ~/mnt

mount -t cd9660 /dev/disk2 ~/mnt

mkdir /Library/WebServer/Documents/ubuntu-16.04.2-server-amd64

# NOTE: Trailing backslashes are very important below...

rsync -av --progress ~/mnt/ /Library/WebServer/Documents/ubuntu-16.04.2-server-amd64/

```

This will put the entire CD contents on the [HTTP](http://localhost) server.

You will notice that your hosts's HTTP server is exposed on the NAT interface as 10.0.2.2.

Now symlink the `dbconf` file to the `Documents` folder:

``bash
cd /Library/WebServer/Documents
ln -s ~/code/pxe/htdocs/ubuntu-16.04.2-server-amd64.cfg /Library/WebServer/Documents
```

## Setup the TFTP Directory

This repository contains the necessary TFTP files to get you started.  Let's create a symbolic link to the `~/Library/VirtualBox/TFTP` location.  Your repository location may be different.

```bash
ln -s $HOME/code/github/rancher-dev/TFTP ~/Library/VirtualBox
```

Let's check:

```bash
$ tree ~/Library/VirtualBox/TFTP
/Users/gillies/Library/VirtualBox/TFTP
├── ldlinux.c32
├── libcom32.c32
├── libutil.c32
├── pxelinux.0
├── pxelinux.cfg
│   ├── default
│   └── pxe.cfg
├── ubuntu
│   ├── 16.04
│   │   └── amd64
│   │       ├── initrd.gz
│   │       └── linux
│   └── ubuntu.menu
└── vesamenu.c32

6 directories, 12 files
```

## Test a Box

Note that when you bring up a Virtual Box, the VirtualBox Guest Additions will be installed.  Obviously you would not do this in a bare metal environment, but this is needed in the development setting.  This is handled in the `Vagrantfile` so it's not a big deal.

On my machine it takes just under 10 minutes to provision a machine from scratch.  This is not normal in a VirtualBox/Vagrant config because we are actually doing a PXE install and starting from a blank .box file.  When you install from a normal .box file it is MUCH FASTER.  We are trying to emulate a bare metal environment as much as possible.

NOTE: You will have to enter your password for the `vagrant-hostmanager` to update the /etc/hosts file.

```bash
$ vagrant up p0.e.int --no-provision
```

Then you should be able to SSH to the box:

```bash
$ vagrant ssh r0.e.int
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.0-66-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Thu Mar 23 10:05:18 2017 from 10.0.2.2
ubuntu@rancher0:~$
```

## Vagrantfile

The `Vagrantfile` reads a data file called `nodes.yml`.  The following VMs are defined:

| Private IP     | Hostname | Base Box        | Time to Create |
|:---------------|:---------|:----------------|:---------------|
| 172.28.128.101 | p0.e.int | c33s/empty      | 10min          |
| 172.28.128.102 | p1.e.int | ubuntu/xenial64 | 3min           |

The default setup is for the `p0.e.int` is to use the `c33s/empty` box file which will cause it to PXE boot and install like a bare metal server.  If are satisfied with the PXE menu and DebConf setup, you can replace the `c33s/empty` with the `ubuntu/xenial64` for faster testing and rebuilding the environment.

# References

[Ubuntu 16.10 Server](http://releases.ubuntu.com/16.10/ubuntu-16.10-server-amd64.iso)
[Ubuntu 16.10 Desktop](http://releases.ubuntu.com/16.10/ubuntu-16.10-desktop-amd64.iso)

[Ubuntu 16.04 Server](http://releases.ubuntu.com/16.04.2/ubuntu-16.04.2-server-amd64.iso)
[Ubuntu 16.04 Desktop](http://releases.ubuntu.com/16.04.2/ubuntu-16.04.2-desktop-amd64.iso)

[Ubuntu 15.04 Server](http://releases.ubuntu.com/14.04/ubuntu-14.04.5-server-amd64.iso)
[Ubuntu 15.04 Desktop](http://releases.ubuntu.com/14.04/ubuntu-14.04.5-desktop-amd64.iso)

[Centos 7 Mimimal](http://centos.host-engine.com/7/isos/x86_64/CentOS-7-x86_64-Minimal-1611.iso )

[SquashFS Kickstarting Ubuntu](http://askubuntu.com/questions/763363/pxe-setup-for-xenial-prepends-squashfs-path-with-cdrom)


# Building Cluster From Scratch
Create a bootable USB from the latest Ubuntu Server LTS [I used Ubuntu 18.04.3 LTS] using Rufus on a Windows computer.

Hostname is $(your-hostname) and domain is $(your-domain). This is tied to the MAC address of the external network card. Should you ever need to change the network card, you have to contact ITS with the MAC address of the card you want $(your-hostname).$(your-domain) (or $(your-hostname2).$(your-domain)) to redirect to.

# Operating System Configuration
## Head Node

Boot from USB [Use F11 for a boot menu]
The bottom network card is the one to select for use. It will automatically setup once selected.
Create a manual partition setup.

| Drive | RAID |
| ----- | ---- |
| sda   | md0  |
| sdb   | md0  |
| sdc   | md1  |
| sdd   | md1  |

| Virtual Group | Virtual Drive Name | Mount point |
| ------------- | ------------------ | ----------- |
| vg0 (on md0)  | boot (1GB)         | /boot       |
|               | swap (16GB)        | swap        |
|               | root (remaining)   | /           |
| vt1 (on md1)  | data               | /data       |

I was prompted to install openssh. Approve this installation.

After installation: (run this periodically when updates need to be installed, which the system alerts you to when you login)

    $ sudo apt-get update
    $ sudo apt-get upgrade
    $ sudo apt-get dist-upgrade
    $ sudo apt-get autoremove
    $ sudo apt-get autoclean

Identify network cards.

    $ ip a

At this point, one network card will be connected to the internet. (For me it is called **eth1**). This will be the external network card. You need to also configure the internal network card (for me it is called **eth0**).

Make a backup copy of the Edit the netplan configuration file before editing.

    $ sudo cp /etc/netplan/01-netcf.yaml /etc/netplan/01-netcf.yaml.bak
    $ sudo vi /etc/netplan/01-netcf.yaml

The external network card should be listed, with dhcp4 marked yes. Add these lines below it:

    etho:
      dhcp4: no
      addresses: [10.42.0.0/8]
      gateway4: $(default-gateway-ip)
      nameservers:
             addresses: [$(ip)]

Note that the reason /8 is used rather than /24 is that the cluster has 7 compute nodes, and the head node is given a subaddress as well. This just ensures that only those nodes are specified.

Test if it works (and accept if it does).

    $ sudo netplan try

Turn on ssh.

    $ sudo systemctl status ssh

Install the firewall.

    $ sudo apt-get install ufw

Configure ufw to allow ssh and connections from the server itself.

    $ sudo ufw allow ssh
    $ sudo ufw allow 10.42.0.0/8

To see the status of the firewall rules:

    $ sudo ufw status

Create IP Masquerading so cluster can communicate with the outside via the head node.
First, packet forwarding needs to be enabled in *ufw*. Two configuration files will need to be adjusted, in /etc/default/ufw change the *DEFAULT_FORWARD_POLICY* to “ACCEPT”:

    DEFAULT_FORWARD_POLICY="ACCEPT"

Then edit /etc/ufw/sysctl.conf and uncomment:

    net/ipv4/ip_forward=1

Similarly, for IPv6 forwarding uncomment:

    net/ipv6/conf/default/forwarding=1

Now add rules to the /etc/ufw/before.rules file. The default rules only configure the *filter* table, and to enable masquerading the *nat*table will need to be configured. Add the following at the end:

    # nat Table rules
    *nat
    :POSTROUTING ACCEPT [0:0]
    
    # Forward traffic from etho through eth1.
    -A POSTROUTING -s 10.42.0.0/8 -o eth1 -j MASQUERADE
    
    # don't delete the 'COMMIT' line or these nat table rules won't be processed
    COMMIT

You need a COMMIT for each table that is altered. A *filter table already exists, so there should be two COMMIT lines.

##  Compute Node

The cluster has a KVM port in the back that can take two USB inputs. Plug in the bootable USB from before and a keyboard (as well as the VGA for the monitor). Each node is selected from the front by pushing the blue KVM button on the node of interest.

Before installation, we checked the RAM capacity of each node.

| Node | RAM   | Proposed SWAP |
| ---- | ----- | ------------- |
| 1    | 64 GB | 128 GB        |
| 2    | 48 GB | 96 GB         |
| 3    | 32 GB | 64 GB         |
| 4    | 28 GB | 56 GB         |
| 5    | 28 GB | 56 GB         |
| 6    | 28 GB | 56 GB         |
| 7    | 28 GB | 56 GB         |

Manually configure network card (this may prove a hassle to get to).

| 10.42.0.0 | $(your-hostname).$(your-domain) |
| --------- | ------------------------ |
| 10.42.0.1 | compute-0-0              |
| 10.42.0.2 | compute-0-1              |
| 10.42.0.3 | compute-0-2              |
| 10.42.0.4 | compute-0-3              |
| 10.42.0.5 | compute-0-4              |
| 10.42.0.6 | compute-0-5              |
| 10.42.0.7 | compute-0-6              |

Netmask: 255.0.0.0
Gateway: 10.42.0.0
Server addresses: $(name-server-1) $(name-server-2)
Hostname: compute-0-#
Domain: site

temp, $(password)

Manual partition:
750GB: Swap (see above), remaining is /
500GB: /boot (we don’t use this drive, so why the hell not?)

Install openssh

Create root password:

    $ sudo passwd root

Edit /etc/hosts (on head node and compute nodes)

    10.42.0.0    $(your-hostname).$(your-domain)    $(your-hostname)
    10.42.0.1    compute-0-0.site    compute-0-0
    10.42.0.2    compute-0-1.site    compute-0-1
    10.42.0.3    compute-0-2.site    compute-0-2
    10.42.0.4    compute-0-3.site    compute-0-3
    10.42.0.5    compute-0-4.site    compute-0-4
    10.42.0.6    compute-0-5.site    compute-0-5
    10.42.0.7    compute-0-6.site    compute-0-6


# Configuring System as a Server

I’m not sure where this really fits into the setup, but you need to edit /etc/ssh/sshd_config on all nodes. Find the line with:

    PermitRootLogin prohibit-password #or whatever else it might say

and change it to

    PermitRootLogin yes

Then run:

    $ sudo service ssh restart
## NFS 

This shares the file system between the head node and the compute nodes so that the data can be centralized.
**Headnode**

    $ sudo apt install nfs-kernel-server

Add these lines to /etc/exports: (for all the compute nodes)

    #DIR        ip        permssions read/write/sync/async etc(nospaces)
    /data   10.42.0.7(rw)
    /home   10.42.0.7(rw)
    /usr/local      10.42.0.7(ro)
    /opt/sge        10.42.0.7(ro)          # Skip until installing SGE

Restart the NFS server.

    $ sudo exportfs -a
    $ sudo systemctl restart nfs-kernel-server

Configure firewall:

    $ sudo ufw allow from 10.42.0.0/8 to any port nfs

**Compute Nodes**
Add this to /etc/fstab:

    10.42.0.0:/data    /data    nfs    auto    0 0
    10.42.0.0:/home    /home    nfs    auto    0 0
    10.42.0.0:/usr/local    /usr/local    nfs    auto    0 0
    10.42.0.0:/opt/sge      /opt/sge      nfs    auto    0 0 # skip until installing SGE

Install NFS-common

    $ sudo apt-get install nfs-common

Mount the shared folders:

    $ sudo mount 10.42.0.0:/data /data
    $ sudo mount 10.42.0.0:/home /home
    $ sudo mount 10.42.0.0:/usr/local /usr/local
    $ sudo mount 10.42.0.0:/opt/sge /opt/sge       # skip until installing SGE


## NIS

This shares the user login information between nodes so that you do not need to make a login for each individual node.
**Head Node**
Install NIS

    $ apt install nis

Make master server

    $ vi /etc/default/nis


    # line 6: change (set NIS master server)
    NISSERVER=master

Give access to client nodes

    $ vi /etc/ypserv.securenets


    255.0.0.0    10.0.0.0
    0.0.0.0      0.0.0.0


    $ vi /var/yp/Makefile


    # line 52: change
    MERGE_PASSWD=true
    
    # line 56: change
    MERGE_GROUP=true

Update the NIS database

    $ /usr/lib/yp/ypinit -m

Make sure that the host to add is $(your-hostname).$(your-domain)

Restart the server:

    $ systemctl restart nis
    $ cd /var/yp
    $ make

**Compute Node**
Install NIS:

    $ apt -y install nis

And have $(your-domain)

Configure as a Client:

    $ vi /etc/yp.conf


    # add the following
    domain $(your-domain) server $(your-hostname).$(your-domain)


    $ vi /etc/nsswitch.conf


    # line 7: add like follows
    passwd:    compat systemd nis
    group:     compat systemd nis
    shadow:    compat nis
    gshadow:   files
    
    hosts:     files dns nis

This will create home directories if there already aren’t any. (not certain this is needed)

    $ vi /etc/pam.d/common-session


    # add to the end
    session optional        pam_mkhomedir.so skel=/etc/skel umask=077

Restart your system when you are finished.

    $ systemctl restart rpcbind nis

Double check that ypbind is running

    $ /usr/sbin/ypbind

To test if the NIS is set up properly, create a user and password on the head node. Try to login with that user@compute-0-# and see if it lets you in to compute node #.

When installing on the head node, it says to run this on all slave nodes, and I didn’t do it until after all that other stuff. Not sure when is appropriate to do it.

    /usr/lib/yp/ypinit -s $(your-hostname).$(your-domain)
# Resource Management and Scheduler
## Son of Grid Engine

**Master**
The following notes are adapted from Jeff Gout.
He set it up so that you should create an account for managing SGE called sgeadmin (in this case, I made pw: $(password2)). But, I’m really installing it as root, because that’s how my system is working.

Create a home for SGE.

    $ mkdir /opt/sge/
    $ cd /opt/sge
    $ SGE_ROOT=/opt/sge/; export SGE_ROOT
    $ echo $SGE_ROOT
    $ chown -R sgeadmin $SGE_ROOT   ## may undo this

Download software.

    $ wget https://arc.liv.ac.uk/downloads/SGE/releases/8.1.9/sge-common_8.1.9_all.deb
    $ wget https://arc.liv.ac.uk/downloads/SGE/releases/8.1.9/sge_8.1.9_amd64.deb
    $ wget http://mirrors.kernel.org/ubuntu/pool/universe/j/jemalloc/libjemalloc1_3.6.0-11_amd64.deb

Install

    $ dpkg -i sge-common_8.1.9_all.deb
    $ dpkg -i sge_8.1.9_amd64.deb
    $ dpkg -i libjemalloc1_3.6.0-11_amd64.deb

I ran into dependency errors, like:

    dpkg: dependency problems prevent configuration of sge-common:
     sge-common depends on bsd-mailx | mailx; however:
      Package bsd-mailx is not installed.
      Package mailx is not installed.
     sge-common depends on tcsh | c-shell; however:
      Package tcsh is not installed.
      Package c-shell is not installed.

Install those dependencies.

    $ apt --fix-broken install

Mail will need to be configured. I selected local only, with a system mail name of $(your-hostname).$(your-domain)

Install Grid Engine

    $ ./install_qmaster

This opens up a program. I’m installing as root.

Using a network service.
I kept cell name as default.
$SGE_CLUSTER_NAME: $(your-hostname)
All defaults afterwards. Using classic spooling because less than 20 nodes.
I have not added any hosts at this time.

Add to .bashrc:

    # SGE settings
    export SGE_ROOT=/opt/sge
    export SGE_CELL=default
    if [ -e $SGE_ROOT/$SGE_CELL ]
    then
            . $SGE_ROOT/$SGE_CELL/common/settings.sh
    fi

Continue installation:

    $ ./install_execd

Run with all defaults.

**Execution Node**
The following is adapted from the [Oracle notes on installation.](https://docs.oracle.com/cd/E19957-01/820-0697/6ncduks45/index.html)

Make the execution nodes known to the master node.

     $ qconf -ah compute-0-0.site

All known nodes will be outputted at this command:

     $ qconf -sh

Note: We aren’t even going to be using this installation of SGE, but we need each node to have all the necessary libraries, so doing it this way will take care of all of that before we mount the head node’s installation folder.

Create a home for SGE.

    $ mkdir /opt/sge/
    $ cd /opt/sge
    $ SGE_ROOT=/opt/sge/; export SGE_ROOT
    $ echo $SGE_ROOT
    $ chown -R sgeadmin $SGE_ROOT   ## may undo this

Download software.

    $ wget https://arc.liv.ac.uk/downloads/SGE/releases/8.1.9/sge-common_8.1.9_all.deb
    $ wget https://arc.liv.ac.uk/downloads/SGE/releases/8.1.9/sge_8.1.9_amd64.deb
    $ wget http://mirrors.kernel.org/ubuntu/pool/universe/j/jemalloc/libjemalloc1_3.6.0-11_amd64.deb

Install

    $ dpkg -i sge-common_8.1.9_all.deb
    $ dpkg -i sge_8.1.9_amd64.deb
    $ dpkg -i libjemalloc1_3.6.0-11_amd64.deb
    $ apt --fix-broken install

Make sure that you have shared head node installation folder with NFS. (involves editing /etc/exports on head node and /etc/fstab on compute nodes, go back and add those now).

Mount original installation file.

     $ mount 10.42.0.0:/opt/sge/

Run install execution node script

    $ ./install_execd

It should identify the default masterq node. I opted for a local spool at /var/tmp/spool. Everything else, I just kept hitting RETURN to continue. I added a default queue.

**Verify Installation**
Notes from [Oracle](https://docs.oracle.com/cd/E19957-01/820-0697/6ncduks5n/index.html).

**Administration Guide**
How to configure your system, according to [Oracle](https://docs.oracle.com/cd/E19957-01/820-0698/index.html).
Parallel environment configured at the [recommendation of MPI](https://www.open-mpi.org/faq/?category=sge).

**User Guide**
How to use the system, according to [Oracle](https://docs.oracle.com/cd/E19957-01/820-0699/).

The following needs to be at the top of all of your submission scripts:

    #$ -S /bin/sh
    #$ -cwd
    #$ -pe mpi #of threads

Basic commands:

    # shows the queue
    qstat -f
## OpenMPI

Beforehand, install the GCC compiler.


    $ apt install build-essential


    $ mkdir /opt/openmpi/
    $ cd /opt/openmpi/
    $ gunzip -c openmpi-4.0.1.tar.gz | tar xf -
    $ cd openmpi-4.0.1
    $ ./configure --prefix=/opt/openmpi-4.0.1 --with-sge
    <...lots of output...>
    $ make all install


## Be sure to source the /opt/sge/default/common/settings.sh as the user. You cannot submit jobs as root.


----------
# General notes:

when adding a new user, be sure to do -m so that the home directory is made

    useradd -m $(username)
    cd /var/yp
    make

The head node kept going to sleep after recent updates, so I have done the following, per a random forum:

    sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target

And the same user said it could be undone by the following should we need to do that.

    sudo systemctl unmask sleep.target suspend.target hibernate.target hybrid-sleep.target

All of the compute nodes were listed as “au.”

    /opt/sge/default/common/sgeexecd

**Make sure new user’s have /bin/bash and not /bin/sh.**


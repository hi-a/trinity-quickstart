=================
Quick start guide
=================

We will be setting up a small cluster using virtual machines as described in the following diagram

.. image:: http://10.2.2.21:8080/misc/quickstart/raw/master/cluster.png

----------------------------
Virtual machines preparation
----------------------------
We'll be using KVM as a hypervisor for this setup.
Sample network and domain definitions are provided in the *virt_trinity* folder. We can use those as is to create our cluster.

    - Define networks::

        ~# virsh net-define <network>; virsh net-start <network>; virsh net-autostart <network>

    - Create a CentOS img for the master node::

        ~# virt-builder centos72 --format qcow2 --selinux-relabel --size 100G -o master.qcow2

    - Create empty disks for the other virtual machines::

        ~# qemu-img create -f qcow2 -o preallocation=metadata <vm>.qcow2 100G

    - Define virtual machines::

        ~# virsh define <vm>

After everything is defined we need to boot up the master node::

    ~# virsh start master
 

-------------
On the master
-------------

Let's first setup the master node that we'll use to configure the controller nodes.

System preparation
==================

- Install required repositories and tools::

    ~# rpm -ivh https://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
    ~# rpm -ivh https://repos.fedorapeople.org/repos/openstack/openstack-juno/rdo-release-juno-1.noarch.rpm
    ~# yum install -y yum-utils createrepo docker docker-registry

- Install xcat::

    http://www.4gh.net/hints/xcat/basic-install.html

- Enable NAT in iptables::

    ~# iptables -A FORWARD -i eth1 -j ACCEPT
    ~# iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

- Enable and start ntpd::

    ~# systemctl enable ntpd
    ~# systemctl start ntpd


xCAT setup
==========

- Update the following key values on the site table::

    "master","192.168.0.254",,
    "forwarders","8.8.8.8,8.8.4.4",,
    "nameservers","192.168.0.254",,
    "domain","kvm-cluster",,
    "dhcpinterfaces","eth1",,
    "timezone","Europe/Amsterdam",,

- Add master, kvmhost and VM nodes to xcat tables (https://sourceforge.net/p/xcat/wiki/XCAT_Virtualization_with_KVM/)
- Update xcat tables using the files in *./trinity/master/tables/*::

    ~# tabrestore ./trinity/master/tables/osimage
    ~# tabrestore ./trinity/master/tables/linuximage

- Setup name resolution and dhcp configuration::

    ~# makehosts -n
    ~# makedns -n
    ~# makedhcp -n
    ~# systemctl restart dhcpd
    ~# rndc reload

Trinity setup
=============

- Clone trinity::

    ~# git clone https://github.com/clustervision/trinity

- Download CentOS everything image

- Update the kickstart template used to configure the controllers and adjust LVM sizes and disk names::

    ~# cat ./trinity/controller/rootimg/install/custom/install/centos/controller.partitions

     part /boot --size 256 --fstype ext4 --ondisk /dev/sda
     part swap --recommended --ondisk /dev/sda
     part pv.01 --size 1 --grow --ondisk /dev/sda
     volgroup vg_root pv.01
     logvol / --vgname=vg_root --name=lv_root --size 50000 --fstype ext4
     logvol  /drbd  --vgname=vg_root --name=lv_drbd --size=40000

- Update openstack nova's configuration to allow for nested virtualization. Add the following line to *./trinity/controller/rootimg/install/postscripts/cv_install_nova_on_controller*::

    openstack-config --set /etc/nova/nova.conf libvirt virt_type qemu

- Run trinity update script to set up necessary configuration files and scripts in their expected paths::

    ~# ./trinity/update.sh master

- Create the repositories that'll be used to setup the controller nodes (otherwise scp from working master)::

    ~# ./master/create_repo_snapshot.sh

- Build docker images::

    ~# ./trinity/controller/rootimg/install/postscripts/cv_build_master_registry

- Build environment modules (otherwise scp from working master)::

    ~# ./trinity/controller/rootimg/install/postscripts/cv_build_master_modules

- Build the login image used to spawn login instances (otherwise scp from working master)::

    ~# ./trinity/controller/rootimg/install/postscripts/cv_build_master_login_image

- In order for a login instance to boot up in a nested virtualization context add the **no_timer_check** kernel option to the image::

    ~# virt-edit -a /trinity/qcows/login.qcow2 /boot/grub2/grub.cfg

- Move the CentOS everything iso file to */trinity/iso/*


Controllers setup
=================

- Assign the active and passive images to the first and second controllers respectively::

    ~# nodeset ha_ctrl1 osimage=centos7-x86_64-install-controller-active
    ~# nodeset ha_ctrl2 osimage=centos7-x86_64-install-controller-passive

- Boot up the first controller::

    ~# rpower node001 on

- After an hour or so, boot up the second controller::

    ~# rpower node002 on


-----------------------
On the main controller:
-----------------------

- To be able to access the dashboard on *http://localhost* we can double tunnel in::

    local# ssh -L 80:localhost:8089 root@kvmhost
    kvmhost# ssh -L 8089:localhost:80 root@ha_ctrl1

- Add compute nodes in xcat tables (mac, nodehm, hosts, hwinv, nodelist, vm)
- ** mkdef -t group -o hw-default,vc-a
- ** nodeadd c1-cx groups=hw-default
- ** add cpuinfo to hwinv table for compute nodes
- ** update /etc/trinity/trinity_api.conf to reflect the correct node_prefix (ha_compute)
- ** build first vc (nova network; nova boot; floatingip attach)
- nodeset computes the correct osimage
- rpower on the computes

---------------
Troubleshooting
---------------

- Trinity repository needs to be cleaned up of unused bits and pieces

Master
======
- update trinity dockerfile (entrypoint fails!!)
- sysconfig/docker has wrong registry address ???
- missing file /opt/xcat/share/xcat/install/scripts/pre.rh.rhel7 (has something to do with the xcat version i'm using)
- /trinity \*(rw,sync,no_root_squash,no_all_squash) must be appended to /etc/exports
- docker images on the master need to be retagged (most probably not!!)
- chdef ha_ctrl1 addkcmdline="selinux=0" ???
- ./otherpkgs: line 891: /usr/bin/logger: Argument list too long (had to comment out the line)
- No need for the ‘/rh/dracut_033’ symlinks in cv_install_controller, they already exist
- we need to be able to re-run postscripts without having to reset a node
- postscripts should provide some sort of error handling

Controller
==========
- make sure the cv_configure_storage refers to the correct disks
- cxx nodes are not automatically added to xcat db
- trinity-api dashboard needs to be restarted in order to reflect current xcat db
- nova network is not created at first (ERROR)
- had to restart pacemaker cluster on the ctrl2 before it could run properly
- when started, the second controller takes over the cluster resources
- if using xCAT 2.11+ trinity api needs to be updated (/usr/lib/python2.7/site-packages/trinity_api/api.py:966) password=>userPW

Login
=====
- slurmctld fails to start if the directory /cluster/var/slurm is missing (mkdir & restart slurm)
- slurm must be restarted when nodes are added or removed from a partition

Compute
=======
- edit /usr/sbin/trinity-start:6 to reflect the correct node prefix if using something other than *node*



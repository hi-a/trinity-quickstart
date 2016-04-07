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

    - Define and start networks::

        ~# (cd networks/; for DEF in $(ls); do virsh net-define $DEF && sudo virsh net-start ${DEF%.xml} && virsh net-autostart ${DEF%.xml}; done)

    - Create a CentOS img for the master node::

        ~# virt-builder centos-7.2 --format qcow2 --selinux-relabel --size 40G -o /opt/master.qcow2

    - Create empty disks for the other virtual machines::

        ~# (cd domains/; for VM in $(ls c* n*); do qemu-img create -f qcow2 -o preallocation=metadata /opt/${VM%.xml}.qcow2 50G; done)

    - Define virtual machines::

        ~# (cd domains/; for DEF in $(ls); do virsh define $DEF; done)

After everything is defined we need to boot up the master node::

    ~# virsh start master
 

-------------
On the master
-------------

Let's first setup the master node that we'll use to configure the controller nodes.

System preparation
==================

- Configure hostname and networking::

    ~#

- Install required repositories and tools::

    ~# rpm -Uvh https://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
    ~# rpm -Uvh https://repos.fedorapeople.org/repos/openstack/openstack-juno/rdo-release-juno-1.noarch.rpm
    ~# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
    ~# rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm

    ~# yum install -y iptables-services ntp wget bash-completion yum-utils createrepo docker docker-registry git

    ~# wget https://sourceforge.net/projects/xcat/files/yum/2.10/xcat-core/xCAT-core.repo -P /etc/yum.repos.d/
    ~# wget https://sourceforge.net/projects/xcat/files/yum/xcat-dep/rh7/x86_64/xCAT-dep.repo -P /etc/yum.repos.d/

    ~# yum install -y xCAT

- Enable NAT in iptables::

    ~# systemctl stop firewalld
    ~# systemctl disable firewalld
    ~# systemctl enable iptables
    ~# systemctl start iptables

    ~# iptables -A FORWARD -i eth1 -j ACCEPT
    ~# iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    ~# service iptables save

- Enable and start ntpd::

    ~# systemctl enable ntpd
    ~# systemctl start ntpd

- Make sure tyhe master node can ssh to the kvm host without a password

   

xCAT setup
==========

- Add xCAT commands to the execution path::

    ~# source /etc/profile.d/xcat.sh

- Update the following key values on the site table::

    ~# tabedit site

    "master","192.168.0.254",,
    "forwarders","8.8.8.8,8.8.4.4",,
    "nameservers","192.168.0.254",,
    "domain","kvm-cluster",,
    "dhcpinterfaces","eth1",,
    "timezone","Europe/Amsterdam",,
    "installdir","/install",,
    "tftpdir","/tftpboot",,
    "xcatdport","3001",,
    "xcatiport","3002",,
    "dhcplease","43200",,

- Add master, kvmhost and VM nodes to xcat tables::
    site, hosts, mac, vm, nodelist, noderes, nodehm, passwd, postscripts
- Setup name resolution and dhcp configuration::

    ~# makehosts -n
    ~# makedns -n
    ~# makedhcp -n
    ~# systemctl restart dhcpd
    ~# rndc reload

Trinity setup
=============

- Clone trinity and checkout latest release::

    ~# git clone https://github.com/clustervision/trinity
    ~# (cd trinity && git checkout r8)

- Update osimage and linuximage tables using the files in *./trinity/master/tables/*::

    ~# tabrestore ./trinity/master/tables/osimage.csv
    ~# tabrestore ./trinity/master/tables/linuximage.csv

- Update the kickstart template used to configure the controllers and adjust LVM sizes and disk names::

    ~# cat ./trinity/controller/rootimg/install/custom/install/centos/controller.partitions

     part /boot --size 256 --fstype ext4 --ondisk /dev/sda
     part swap --recommended --ondisk /dev/sda
     part pv.01 --size 1 --grow --ondisk /dev/sda
     volgroup vg_root pv.01
     logvol / --vgname=vg_root --name=lv_root --size 25000 --fstype ext4
     logvol  /drbd  --vgname=vg_root --name=lv_drbd --size=22000

- Update openstack nova's configuration to allow for nested virtualization. Add the following line to *./trinity/controller/rootimg/install/postscripts/cv_install_nova_on_controller*::

    openstack-config --set /etc/nova/nova.conf libvirt virt_type qemu

- Run trinity update script to set up necessary configuration files and scripts in their expected paths::

    ~# (cd trinity/ && ./update master)

- Create the repositories that'll be used to setup the controller nodes::

    ~# cat ./trinity/controller/rootimg/install/custom/install/centos/*pkg* ./trinity/controller/rootimg/install/custom/netboot/centos/*pkg* | grep -v ^# | grep -v ^$ | grep -v ^@ | sort -u > /tmp/pkglist
    ~# cat ./trinity/controller/rootimg/install/custom/install/centos/*pkg* ./trinity/controller/rootimg/install/custom/netboot/centos/*pkg* | grep ^@ | sort -u > /tmp/grplist
    ~# mkdir -p /install/post/otherpkgs/centos7/x86_64/Packages
    ~# cat /tmp/pkglist | xargs repotrack -p /install/post/otherpkgs/centos7/x86_64/Packages
    ~# cat /tmp/grplist | xargs yumdownloader --resolve --destdir /install/post/otherpkgs/centos7/x86_64/Packages
    ~# createrepo /install/post/otherpkgs/centos7/x86_64/

- Build docker images::

    ~# systemctl start docker docker-registry
    ~# systemctl enable docker-registry
    ~# ./trinity/controller/rootimg/install/postscripts/cv_build_master_registry

- Build environment modules (otherwise scp from working master)::

    ~# ./trinity/controller/rootimg/install/postscripts/cv_build_master_modules

- Build the login image used to spawn login instances (otherwise scp from working master)::

    ~# ./trinity/controller/rootimg/install/postscripts/cv_build_master_login_image

- In order for a login instance to boot up in a nested virtualization context add the **no_timer_check** kernel option to the image::

    ~# virt-edit -a /trinity/qcows/login.qcow2 /boot/grub2/grub.cfg

- Download CentOS DVD image::

    ~# mkdir /trinity/iso
    ~# wget http://mirror.amsiohosting.net/centos.org/7/isos/x86_64/CentOS-7-x86_64-DVD-1511.iso -P /trinity/iso

- Create initial centos repositories::

    ~# copycds -n centos7 -o /trinity/iso/CentOS-7-x86_64-DVD-1511.iso

Controllers setup
=================

- Assign the active and passive images to the first and second controllers respectively::

    ~# nodeset controller-1 osimage=centos7-x86_64-install-controller-active
    ~# nodeset controller-2 osimage=centos7-x86_64-install-controller-passive

- Boot up the first controller::

    ~# rpower controller-1 on

- After an hour or so, boot up the second controller::

    ~# rpower controller-2 on


-----------------------
On the main controller:
-----------------------

- To be able to access the dashboard on *http://localhost* we can double tunnel in::

    local# ssh -L 80:localhost:8089 root@kvmhost
    kvmhost# ssh -L 8089:localhost:80 root@vt_controller_1

- Add compute nodes in xcat tables (hosts, mac, nodehm, hwinv, nodelist, vm)
- ** add cpuinfo to hwinv table for compute nodes
- Add a new default group that will hold container members that we'll create in the next step::

   ~# mkdef -t group -o hw-default

- Add container definitions to xcat tables for trinity to be able to manage cluster partitions::

   ~# nodeadd c1-cx groups=hw-default

- Update trinity's config file */etc/trinity/trinity_api.conf* to reflect the correct node prefix if using a prefix other than *node* (vt_compute)
- Setup name resolution and dhcp configuration::

    ~# makehosts -n
    ~# makedns -n
    ~# makedhcp -n
    ~# systemctl restart dhcpd
    ~# rndc reload

- Assign the trinity netboot image to the compute nodes::

    ~# nodeset compute osimage=centos7-x86_64-netboot-trinity

- Boot up the compute nodes::

    ~# rpower compute on


---------------
Troubleshooting
---------------

- Trinity repository needs to be cleaned up of unused bits and pieces

Master
======
- update script needs to clean up any existing packages
- sysconfig/docker has wrong registry address ???
- missing file /opt/xcat/share/xcat/install/scripts/pre.rh.rhel7 (has something to do with the xcat version i'm using)
- /trinity \*(rw,sync,no_root_squash,no_all_squash) must be appended to /etc/exports && exportfs -a
- ./otherpkgs: line 891: /usr/bin/logger: Argument list too long (had to comment out the line)
- No need for the ‘/rh/dracut_033’ symlinks in cv_install_controller, they already exist
- we need to be able to re-run postscripts without having to reset a node
- postscripts should provide some sort of error handling

Controller
==========
- make sure the cv_configure_storage refers to the correct disks
- cxx nodes are not automatically added to xcat db
- trinity-api dashboard needs to be restarted in order to reflect current xcat db
- had to restart pacemaker cluster on the ctrl2 before it could run properly
- if using xCAT 2.10+ trinity api needs to be updated (/usr/lib/python2.7/site-packages/trinity_api/api.py:966) password=>userPW
- https://github.com/clustervision/trinity/blob/r8/controller/rootimg/install/postscripts/cv_ha_sentinel#L17 Error: Unable to find constraint - 'location-ip-controller-1.cluster-50'

Login
=====
- slurm must be restarted when nodes are added or removed from a partition

Compute
=======
- edit /usr/sbin/trinity-start:6 to reflect the correct node prefix if using something other than *node* (vt_compute)
- when reset, the compute nodes fail load docker daemon. docker pool has different UUID and disks are not reformated.


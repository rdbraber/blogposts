## Creating an iSCSI setup on Oracle Linux 7

As I’m currently studying for my Red Hat Certified Engineer (RHCE) 7 certification, one of the objectives is to configure a system as either an iSCSI target or initiator that persistently mounts an iSCSI target. Compared to Oracle Linux 6, this is one of the things that’s completely different on Oracle Linux 7. Since I had to write down the steps for study purposes I might as well write a blog post about it. 

First some terminology. For an iSCSI setup you need a target and an initiator, which both have an IQN.

* The target is the service on an iSCSI server that gives access to backend storage devices. 
* The initiator is the iSCSI client that connects to a target and is identified by an IQN
* The IQN is the iSCSI qualified name, which is a unique name that is used to identifying both targets.

In this post I use two servers: itarget01 & iclient01. I’ve used the Vagrant box [rafacas/ol70-plain](https://atlas.hashicorp.com/rafacas/boxes/oel70-plain).

* **Itarget01** will be our iSCSI target.
* **Iclient01** will be our iSCSI client (initiator). Here we will mount the iSCSI LUNs (Logical Unit Number).

### Installing and configuring the iSCSI target
To create an iSCSI target on OL7 we use the targetcli command. Normally this is not installed so make sure you install it with the yum install targetcli command:

~~~
[root@itarget01 ~]# yum -y install targetcli
~~~

Now we can use the target command line interface. Use the help command to show the available commands you can use:

~~~
[root@itarget01 ~]# targetcli
targetcli shell version 2.1.fb41
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.

/> help

GENERALITIES
============
This is a shell in which you can create, delete and configure
configuration objects.

The available commands depend on the current path or target
path you want to run a command in: different path have
different sets of available commands, i.e. a path pointing at
an iscsi target will not have the same availaible commands as,
say, a path pointing at a storage object.

The prompt that starts each command line indicates your
current path. Alternatively (useful if the prompt displays
an abbreviated path to save space), you can run the
pwd command to display the complete current path.

Navigating the tree is done using the cd command. Without
any argument, cd will present you wil the full objects
tree. Just use arrows to select the destination path, and
enter will get you there. Please try help cd for navigation
tips.

COMMAND SYNTAX
==============
Commands are built using the following syntax:

[TARGET_PATH] COMMAND_NAME [OPTIONS]

The TARGET_PATH indicates the path to run the command from.
If ommited, the command will be run from your current path.

The OPTIONS depend on the command. Please use help
COMMAND to get more information.


AVAILABLE COMMANDS
==================
The following commands are available in the
current path:

  - bookmarks action [bookmark]
  - cd [path]
  - clearconfig [confirm]
  - exit
  - get [group] [parameter...]
  - help [topic]
  - ls [path] [depth]
  - pwd
  - refresh
  - restoreconfig [savefile] [clear_existing]
  - saveconfig [savefile]
  - sessions [action] [sid]
  - set [group] [parameter=value...]
  - status
  - version
/>

~~~

The first command we can use is the ls command to show the current target configuration:

~~~
/> ls
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 0]
  | o- fileio ................................................................................................. [Storage Objects: 0]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 0]
  o- loopback ......................................................................................................... [Targets: 0]
/
~~~

As shown in the output above, nothing is configured yet. The first thing we are going to configure is the backing storage we want to use. For the purpose of this blog, we use a file-backed block device. We will create a sparse file of 100 Mb in the home directory of the root account. First go to the backstores branch, which you can do by using the cd command:

~~~
/> cd /backstores/
/backstores>
~~~

Next create the 100Mb file:

~~~
/backstores> fileio/ create lun01 /root/lun01 100M
'export_backstore_name_as_model' is set but emulate_model_alias
  not supported by kernel.
~~~

To show the result of the command, use the ls command again:

~~~
/backstores> ls
o- backstores ................................................................................................................ [...]
  o- block .................................................................................................... [Storage Objects: 0]
  o- fileio ................................................................................................... [Storage Objects: 1]
  | o- lun01 ....................................................................... [/root/lun01 (100.0MiB) write-back deactivated]
  o- pscsi .................................................................................................... [Storage Objects: 0]
  o- ramdisk .................................................................................................. [Storage Objects: 0]
~~~

Now we need to create a unique identifier for the iSCSI target. This is done in the iscsi branch. To go to this branch use the cd command again:

~~~
/backstores> cd /iscsi
/iscsi>
~~~

The name of an iqn must comply to a standard. The normal standard is iqn.\<YYYY>-\<MM>.\<reverse syntax of internet domain name>:\<unique name>. An example of such a name would be iqn.2016-01.com.example:iscsitarget.
Use the create command to create the iqn:

~~~
/iscsi> create iqn.2016-01.com.example:iscsitarget
Created target iqn.2016-01.com.example:iscsitarget.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
~~~

With the ls command you can show the newly created iqn:

~~~
/iscsi> ls
o- iscsi .............................................................................................................. [Targets: 1]
  o- iqn.2016-01.com.example:iscsitarget ................................................................................. [TPGs: 1]
    o- tpg1 ................................................................................................. [no-gen-acls, no-auth]
      o- acls ............................................................................................................ [ACLs: 0]
      o- luns ............................................................................................................ [LUNs: 0]
      o- portals ...................................................................................................... [Portals: 1]
        o- 0.0.0.0:3260 ....................................................................................................... [OK]
~~~

To make it possible for our iSCSI client to use the storage, we create an ACL for the storage.  First go to the iqn branch. Just like on the normal linux prompt, you can use the tab key for auto completion:

~~~
/iscsi> cd iqn.2016-01.com.example:iscsitarget/
~~~

Now create the ACL:

~~~
/iscsi/iqn.20...e:iscsitarget> tpg1/acls create iqn.2016-01.com.example:iclient01
Created Node ACL for iqn.2016-01.com.example:iclient01
/iscsi/iqn.20...e:iscsitarget> ls
o- iqn.2016-01.com.example:iscsitarget ................................................................................... [TPGs: 1]
  o- tpg1 ................................................................................................... [no-gen-acls, no-auth]
    o- acls .............................................................................................................. [ACLs: 1]
    | o- iqn.2016-01.com.example:iclient01 ........................................................................ [Mapped LUNs: 0]
    o- luns .............................................................................................................. [LUNs: 0]
    o- portals ........................................................................................................ [Portals: 1]
      o- 0.0.0.0:3260 ......................................................................................................... [OK]
/iscsi/iqn.20...e:iscsitarget>
~~~

Create the lun:

~~~
/iscsi/iqn.20...e:iscsitarget> tpg1/luns create /backstores/fileio/lun01
Created LUN 0.
Created LUN 0->0 mapping in node ACL iqn.2016-01.com.example:iclient01
/iscsi/iqn.20...e:iscsitarget> ls
o- iqn.2016-01.com.example:iscsitarget ................................................................................... [TPGs: 1]
  o- tpg1 ................................................................................................... [no-gen-acls, no-auth]
    o- acls .............................................................................................................. [ACLs: 1]
    | o- iqn.2016-01.com.example:iclient01 ........................................................................ [Mapped LUNs: 1]
    |   o- mapped_lun0 .................................................................................... [lun0 fileio/lun01 (rw)]
    o- luns .............................................................................................................. [LUNs: 1]
    | o- lun0 ......................................................................................... [fileio/lun01 (/root/lun01)]
    o- portals ........................................................................................................ [Portals: 1]
      o- 0.0.0.0:3260 ......................................................................................................... [OK]

~~~

The configuration is done now. Just to have a look go back to the root of the configuration and use the ls command again:

~~~
/iscsi/iqn.20...e:iscsitarget> cd /
/> ls
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 0]
  | o- fileio ................................................................................................. [Storage Objects: 1]
  | | o- lun01 ....................................................................... [/root/lun01 (100.0MiB) write-back activated]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 1]
  | o- iqn.2016-01.com.example:iscsitarget ............................................................................... [TPGs: 1]
  |   o- tpg1 ............................................................................................... [no-gen-acls, no-auth]
  |     o- acls .......................................................................................................... [ACLs: 1]
  |     | o- iqn.2016-01.com.example:iclient01 .................................................................... [Mapped LUNs: 1]
  |     |   o- mapped_lun0 ................................................................................ [lun0 fileio/lun01 (rw)]
  |     o- luns .......................................................................................................... [LUNs: 1]
  |     | o- lun0 ..................................................................................... [fileio/lun01 (/root/lun01)]
  |     o- portals .................................................................................................... [Portals: 1]
  |       o- 0.0.0.0:3260 ..................................................................................................... [OK]
  o- loopback ......................................................................................................... [Targets: 0]

~~~

When you type the exit command, the configuration is saved and written to the file /etc/target/saveconfig.json. If you want you can take a look at this file, to show the content.
The iSCSI target service is not active yet, so it must be started. Also we want it to start it when the server boots:

~~~
[root@itarget01 ~]# systemctl start target
[root@itarget01 ~]# systemctl enable target
ln -s '/usr/lib/systemd/system/target.service' '/etc/systemd/system/multi-user.target.wants/target.service'
[root@itarget01 ~]# systemctl status target
target.service - Restore LIO kernel target configuration
   Loaded: loaded (/usr/lib/systemd/system/target.service; enabled)
   Active: active (exited) since Sun 2016-01-24 20:30:01 CET; 14s ago
 Main PID: 13144 (code=exited, status=0/SUCCESS)

Jan 24 20:30:01 itarget01.example.com systemd[1]: Starting Restore LIO kernel target configuration...
Jan 24 20:30:01 itarget01.example.com systemd[1]: Started Restore LIO kernel target configuration.
~~~

If you have a firewall running on your iSCSI target server, you have to open port 3260:

~~~
[root@itarget01 ~]# firewall-cmd --add-port=3260/tcp --permanent
FirewallD is not running
[root@itarget01 ~]# firewall-cmd --reload
FirewallD is not running
~~~

## Installing and configuring the iSCSI client

On the iSCSI client we first must install the initiator software:

~~~
[root@iclient01 ~]# yum -y install iscsi-initiator-utils lsscsi
~~~

Once the software is installed, we need to set the iSCSI Initiatorname. This can be done in the file /etc/iscsi/initiatorname.iscsi. During the creation of the ACL we already set the initiatorname to iqn.2016-01.com.example:iclient01.  So make the change in the file and restart the iscsid service:
[root@iclient01 ~]# cat /etc/iscsi/initiatorname.iscsi
InitiatorName=iqn.2016-01.com.example:iclient01
[root@iclient01 ~]# systemctl restart iscsid

Now we discover the iSCSI LUNs on the iSCSI target with the iscsiadm command:

~~~
[root@iclient01 ~]# iscsiadm --mode discovery --type sendtargets --portal itarget01 --discover
192.168.10.210:3260,1 iqn.2016-01.com.example:iscsitarget
~~~

We can now login to the iSCSI target:

~~~
[root@iclient01 ~]# iscsiadm --mode node --targetname iqn.2016-01.com.example:iscsitarget --portal itarget01:3260 --login
Logging in to [iface: default, target: iqn.2016-01.com.example:iscsitarget, portal: 192.168.10.210,3260] (multiple)
Login to [iface: default, target: iqn.2016-01.com.example:iscsitarget, portal: 192.168.10.210,3260] successful.
~~~

The LUN should be available now, which we can check:

~~~
[root@iclient01 ~]# iscsiadm -m session -P3
iSCSI Transport Class version 2.0-870
version 6.2.0.873-30
Target: iqn.2016-01.com.example:iscsitarget (non-flash)
	Current Portal: 192.168.10.210:3260,1
	Persistent Portal: 192.168.10.210:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.2016-01.com.example:iclient01
		Iface IPaddress: 192.168.10.211
		Iface HWaddress: <empty>
		Iface Netdev: <empty>
		SID: 1
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		*********
		Timeouts:
		*********
		Recovery Timeout: 120
		Target Reset Timeout: 30
		LUN Reset Timeout: 30
		Abort Timeout: 15
		*****
		CHAP:
		*****
		username: <empty>
		password: ********
		username_in: <empty>
		password_in: ********
		************************
		Negotiated iSCSI params:
		************************
		HeaderDigest: None
		DataDigest: None
		MaxRecvDataSegmentLength: 262144
		MaxXmitDataSegmentLength: 262144
		FirstBurstLength: 65536
		MaxBurstLength: 262144
		ImmediateData: Yes
		InitialR2T: Yes
		MaxOutstandingR2T: 1
		************************
		Attached SCSI devices:
		************************
		Host Number: 2	State: running
		scsi2 Channel 00 Id 0 Lun: 0
			Attached scsi disk sdb		State: running
~~~
The last line shows that we have a device with the name sdb, which we can also check with the lsscsi:

~~~
[root@iclient01 ~]# lsscsi
[0:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sda
[2:0:0:0]    disk    LIO-ORG  FILEIO           4.0   /dev/sdb
~~~

We can now create a filesystem on the device:

~~~
[root@iclient01 ~]# mkfs.xfs /dev/sdb
~~~

Use the blkid command to find the UUID of the filesystem:

~~~
[root@iclient01 ~]# blkid /dev/sdb
/dev/sdb: UUID="38c0bea4-2edf-4591-a528-d37b809b8c36" TYPE="xfs"
~~~

We are going to use the UUID in the file /etc/fstab. Add an entry to this file simular to the following example. The most important part is the **_netdev** option. This is to make sure the device is mounted after the network is started:

~~~
UUID=38c0bea4-2edf-4591-a528-d37b809b8c36	/mnt/iscsi	xfs	_netdev	1 2
~~~

Create the mountpoint and use the mount command to mount the filesystem:

~~~
[root@iclient01 ~]# mkdir /mnt/iscsi
[root@iclient01 ~]# mount /mnt/iscsi
[root@iclient01 ~]# df -h |grep iscsi
/dev/sdb              97M  5.2M   92M   6% /mnt/iscsi
~~~

Last check would be to reboot the client to make sure the LUN is mounted while the system is booting.

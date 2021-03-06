# <a name="Top"></a>Creating a Managed Lustre ZFS Filesystem on a Vagrant virtual cluster

[**Software Contributor Documentation Table of Contents**](cd_TOC.md)

![zfs](md_Graphics/monitored_filesystem_sm.jpg)

## Prerequisites:

Please refer to [https://github.com/intel-hpdd/vagrantfiles](https://github.com/intel-hpdd/vagrantfiles) on how to create a virtual HPC storage cluster with vagrant before attempting to install IML.

## Download IML build:

Note: use vagrant ssh-config to get the port each server is running on. The commands below use ports that are specific to my vagrant environment.

1.  Verify the following vagrant plugins are installed:
    ```
    vagrant plugin install vagrant-shell-commander
    ```
2.  Download the latest IML build (tarball).
    from: [https://github.com/intel-hpdd/intel-manager-for-lustre/releases/download/v{{site.version}}/{{site.package_name}}.tar.gz](https://github.com/intel-hpdd/intel-manager-for-lustre/releases/download/v{{site.version}}/{{site.package_name}}.tar.gz)

## Installing IML:

1.  Copy the IML build to the `/tmp` directory in your admin node:
    ```
    scp -P 2222 ~/Downloads/{{site.package_name}}.tar.gz vagrant@127.0.0.1:/tmp/.
    # password is "vagrant"
    ```
1.  ssh into the admin box and install the build:
    ```
    vagrant ssh
    [vagrant@adm ~]$ sudo su - # (or "sudo -s")
    [vagrant@adm ~]# cd /tmp
    [vagrant@adm ~]# tar xvf <buildname>.tar.gz
    [vagrant@adm ~]# cd <build folder>
    [vagrant@adm ~]# ./install --no-dbspace-check
    ```
1.  Update the /etc/hosts file on your computer to include the following line:
    ```
    127.0.0.1 adm.lfs.local
    ```
1.  Test that a connection can be made to IML by going to the following link in your browser:
    [https://adm.lfs.local:8443](https://adm.lfs.local:8443)

## Adding and Configuring Servers

You should now be able to see IML when navigating to [https://adm.lfs.local:8443](https://adm.lfs.local:8443). Click on the login link at the top right and log in as the admin. Next, go to the server configuration page and add the following servers using the `Managed` profile:

```
mds[1,2].lfs.local,oss[1,2].lfs.local
```

This will take some time (around 20 to 30 minutes) but all four servers should add successfully.

## Configuring Interfaces

Once all servers have been added, each server will need to know which interface should be assigned the lustre network. Navigate to each server detail page by clicking on the server link. Scroll to the bottom of the server detail page where you will see a list of network interfaces. Click on the Configure button and you will be given the option to change the network driver and the network for each interface.

The vagrant file indicates that the lustre network will run on 10.73.20.x. If Lustre Network 0 is specified for a different IP address, you will need to change its interface to `Not Lustre Network` and update the network for 10.73.20.x to use `Lustre Network 0`. It is very important that Lustre Network 0 is specified on the correct interface; otherwise, creating a filesystem will fail. Make sure that all servers have been updated where appropriate.

## Setting up Power Control

[Follow these instructions to configure the Power Control.](cd_Setting_Up_Power_Control.md)

## Creating ZPools

Now that all managed servers have been added and the power control has been configured, the ZPools can be created.

1.  ssh into `mds1` and switch to the root account with `sudo -i`.
1.  In the GUI, navigate to `Configuration->Volumes`. Setup your web browser and console next to each other such that the result of running commands in your console can be viewed on the browser in real time.
1.  Create the MGS. On the volumes page, look for the mds with a file size of 512MB. This should be easy to spot as the other mds has a capacity of 5GB. Notice the numbers that come after `VBOX_HARDDISK`. This is the diskSerial. Enter the following command:

```
root@mds1 by-id]# zpool create mgs -o cachefile=none -o multihost=on /dev/disk/by-id/ata-VBOX_HARDDISK_MGS00000000000000000
[root@mds1 by-id]# zpool export mgs
```

Watch the volumes page and wait for the volumes associated with the mgs to change and become labeled "mgs."

1.  Create the MDS. On the volumes page, look for the mds with a file size of 5GB. This should be the only mds left. Take note of the diskSerial for this mds and enter the following command:

```
[root@mds1 by-id]# zpool create mds -o cachefile=none -o multihost=on /dev/disk/by-id/ata-VBOX_HARDDISK_MDT00000000000000000
[root@mds1 by-id]# zpool export mds
```

Watch the volumes page and wait for the volumes associated with the mds to change and become labeled "mds."

1.  Create OSS1. Exit out of `mds1` and ssh into `oss1`. On the volumes page, look for all occurrences of `oss1` and take note of their diskSerial's. Enter the following command:

```
[root@oss1 ~]# zpool create oss1 -o cachefile=none -o multihost=on raidz2 /dev/disk/by-id/ata-VBOX_HARDDISK_OST0PORT100000000000 /dev/disk/by-id/ata-VBOX_HARDDISK_OST2PORT300000000000 /dev/disk/by-id/ata-VBOX_HARDDISK_OST4PORT500000000000 /dev/disk/by-id/ata-VBOX_HARDDISK_OST6PORT700000000000
[root@oss1 ~]# zpool export oss1
```

Watch the volumes page and wait for the volumes associated with oss1 to change and become labeled "oss1."

1.  Create OSS2. On the volumes page, look for all occurrences of `oss2` and take note of their diskSerial's. Enter the following command:

```
[root@oss1 ~]# zpool create oss2 -o cachefile=none -o multihost=on raidz2 /dev/disk/by-id/ata-VBOX_HARDDISK_OST1PORT200000000000 /dev/disk/by-id/ata-VBOX_HARDDISK_OST3PORT400000000000  /dev/disk/by-id/ata-VBOX_HARDDISK_OST5PORT600000000000 /dev/disk/by-id/ata-VBOX_HARDDISK_OST7PORT800000000000
[root@oss1 ~]# zpool export oss2
```

Watch the volumes page and wait for the volumes associated with oss2 to change and become labeled "oss2."

## Creating a Filesystem

To create a filesystem, simply navigate to `Configuration->File Systems` and click the `Create` button. Make the following selections:

* Management Target / MGS -> `mgs` (512 MB)
* Metadata Target / MDS -> `mds`
* Object Storage Targets -> Select `oss1` and `oss2`

After the selections have been made, click the button to create the filesystem. If you have any issues creating the filesystem there is a good chance that the interface for 10.73.20.x is not assigned to Lustre Network 0. If this happens, stop the filesystem and update the interfaces accordingly.

## Setting up Clients

See [Setting up Clients](cd_Setting_Up_Clients.md).

---

[Top of page](#Top)

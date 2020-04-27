# Unattended installation of RHVH Hosts using Red Hat Satellite

## What you need

* A running Satellite server
* Some bare metal hosts to provision
* The latest RHVH ISO from the [Red Hat Customer Portal](https://access.redhat.com/downloads/content/415/ver=4.3/rhel---7/4.3/x86_64/product-software) (subscription needed)

## Setting up Satellite : the files

The next steps are all performed on the Satellite server itself for practical reasons.

Download the RHVH ISO and place it somewhere in the filesystem, i.e. `/var/tmp`

Create the needed directory structure on the Satellite server (or capsule) under the public directory for direct access:
```
mkdir -p /var/www/html/pub/rhvh/images/pxeboot /var/www/html/pub/rhvh/LiveOS
```

Mount the ISO file and extract the needed files in the public directory. We need the kernel, the initram image, the LiveOS squashfs image, and the RHVH squashfs image which is inside an rpm:
```
mkdir -p /mnt/rhvh
mount -t iso9660 -o ro /var/tmp/RHVH-4.3-*-RHVH-x86_64-dvd1.iso /mnt/rhvh/

mkdir -p /tmp/rhvh && cd /tmp/rhvh

rpm2cpio /mnt/rhvh/Packages/redhat-virtualization-host-image-update* | cpio -imdv  \*img
mv ./usr/share/redhat-virtualization-host/image/redhat-virtualization-host-*img /var/www/html/pub/rhvh/images/

cp -a /mnt/rhvh/images/pxeboot/{initrd.img,vmlinuz} /var/www/html/pub/rhvh/images/pxeboot/
cp -a /mnt/rhvh/images/product.img /var/www/html/pub/rhvh/images/
cp -a /mnt/rhvh/.treeinfo /var/www/html/pub/rhvh/
cp -a /mnt/rhvh/LiveOS/squashfs.img /var/www/html/pub/rhvh/LiveOS/

umount /mnt/rhvh

```

_Note_: Using a custom Product / Repository for this would seem a perfect fit for this need, so why haven't we used it?

Because custom repositories have no *directories* and a proper file structure is required, in this case. 

## Setting up Satellite : the objects

In the Satellite WebUI, create a new *InstallationMedia* named i.e. `RHVH` with the following specs:
* Name: RHVH
* Path: http://satellite.example.com/pub/rhvh/

The path here should point to the publicly accessible directory we created earlier. Please note it's *http* and **not** *https*

In the Satellite WebUI, create a new *OperatingSystem* named i.e. `RHVH 4.3` with the following specs:
 * Name: `RedHat`
 * Major: `4`
 * Minor: `3`
 * Description: `RHVH 4.3`
 * Family: `Red Hat`
 * Arch: `x86_64`
 * Partition Table: `Kickstart default thin`
 * Installation Media: `RHVH`

In the Satellite WebUI, associate the RHVH templates `Kickstart oVirt-RHVH` and `Kickstart oVirt-RHVH PXELinux` with the newly created *OperatingSystem* `RHVH 4.3`
* Hosts -> Provisioning Templates -> select the template -> Association

Of course you may use your own provisioning and pxelinux templates. In this repo you'll find a slightly modified version of the two templates:
* [Baremetal Kickstart oVirt-RHVH](baremetal_kickstart_ovirtrhvh.erb) providing automated remote execution user creation
* [Baremetal Kickstart oVirt-RHVH PXELinux](baremetal_kickstart_ovirtrhvh_pxelinux.erb) providing kernel cmdline parameters for setting up static ip during boot, when subnet is not set on DHCP.

Set the default templates in the *Operating System* `RHVH 4.3` and set some parameters:
 * PXELinux Template: `Kickstart oVirt-RHVH PXELinux`
 * Provisioning Template: `Kickstart oVirt-RHVH`
 * Parameters: 
   * `disable-firewall` (boolean) = `false` ; *RHVH hosts need Firewalld to be enabled*
   * `liveimg_name` (string) = `http://satellite.example.com/pub/rhvh/images/redhat-virtualization-host-4.3.9-20200324.0.el7_8.squashfs.img` ; *This is the file we extracted earlier. It is used in the provisioning template*
   * [optional] `kt_activation_keys` (string) = `myactivationkey` *Set this if you plan on subscribing hosts during installation*

You may also set the activation key parameter in a hostgroup, if you prefer.

[Optional] Create a new *HostGroup* to speed up host creation. 
Be sure to set all default values you need for this group, such as Location, OperatingSystem, Subnet, Domain, and activation key.

## Provisioning

Now create a new *Host*. Make sure you specify all information needed, especially the network info (mac, IP, etc.). Failing to do so may prevent from provisioning correctly or even accepting the new Host request at all.

After hitting submit, if you are using Satellite's DHCP, you may turn the machine on and it would provision itself automatically.

If you're not using DHCP, you may download the `Full Host image` ISO file and use it via OOB management interface. 

Assuming your host is named `mynewhost`, the image is available:
* via WebUI: Hosts -> mynewhost -> Boot Disk -> Full Host "mynewhost" image
* or via API: `curl -k -u mysatuser:mypassword -vL https://satellite.example.com/bootdisk/api/hosts/mynewhost.example.com?full=true -o mynewhost.example.com.iso`


# `udev` Notes

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

- [`udev` Notes](#udev-notes)
    - [Objective](#objective)
    - [History of `udev`](#history-of-udev)
        - [Static Device Files](#static-device-files)
        - [Dynamically Creating Device Files - `devfs`: An Imperfect Solution](#dynamically-creating-device-files---devfs-an-imperfect-solution)
    - [Using `udev`](#using-udev)
        - [Structure of `udev` Rules](#structure-of-udev-rules)
        - [`udev` Debug](#udev-debug)
    - [Creating `udev` rules](#creating-udev-rules)

<!-- markdown-toc end -->

## Objective

`udev`, currently part of `systemd`, is used mainly to dynamically create and
remove device files under `/dev`. Its predessors is `devfs`.

## History of `udev`

### Static Device Files

Traditionally Unix devices are represented by statically-created device
files. These are special files created using the `mknod` command, containing a
device type, major, and minor version numbers. The major number identifies the
driver applicable to the device in the kernel, while the minor number is used by
the driver itself, sometimes used to differentiate multiple similar devices.

The device type, major, minor numbers combination uniquely identifies a
device. The following are applicable device types

* `c` character device (buffered)
* `u` character device (unbuffered)
* `b` block device
* `p` FIFO

As an exception, device type `p` FIFO does not have major and minor version
numbers.

The Linux kernel maintains a list of registered device major and minor
numbers, shown in [`Documentation/admin-guide/devices.txt`][1]. For example, a
character device of major number 1, minor number 9 is the faster, less secure
random number generator normally called `urandom`:

```
$ file /dev/urandom
/dev/urandom: character special (1/9)
```

Using `mknod`, another device file representing this device can be created. When
this file is opened by a program, the program will interact with the
corresponding driver in the kernel. The `cat` command subsequently confirms the
device file is indeed working as it should.

```
# mknod ~/test-random-device c 1 9

$ cat ~/test-random-device
```

### Dynamically Creating Device Files - `devfs`: An Imperfect Solution

As the world becomes more complicated, it is no longer feasible to use static
device files to represent devices, as noted by kernel developer Greg
Korah-Hartman in a public email titled ["udev and devfs - The final word"][2] on
December 30, 2003:

> 1) A static /dev is unwiedly and big. It would be nice to only show the /dev
>    entries for the devices we actually have running in the system.
> 
> 2) We are (well, were) running out of major and minor numbers for devices.
>
> 3) Users want a way to name devices in a persistent fashion (i.e. "This disk
>    here, must _always_ be called "boot_disk" no matter where in the scsi tree
>    I put it", or "This USB camera must always be called "camera" no matter if
>    I have other USB scsi devices plugged in or not.")
>
> 4) Userspace programs want to know when devices are created or removed, and
>    what /dev entry is associated with them.

`devfs`, now deprecated in the kernel, is a temporary solution with its own
problems. Quoting from the same email:

> Problems:
> 1) devfs only shows the dev entries for the devices in the system. 
> 2) devfs does not handle the need for dynamic major/minor numbers
> 3) devfs does not provide a way to name devices in a persistent fashion.
> 4) devfs does provide a daemon that userspace programs can hook into to listen
>    to see what devices are being created or removed.
>
> Constraints:
> 1) devfs forces the devfs naming policy into the kernel. If you don't like
>    naming scheme, tough.
> 2) devfs does not follow the LSB device naming standard.
> 3) devfs is small, and embedded devices use it. However it is implemented in
>    non-pagable memory.

Thus, `udev` becomes the _Final Word_. Again, quoting from the same email:

> Problems:
> 1) using udev, the /dev tree only is populated for the devices that are
>    currently present in the system.
> 2) udev does not care about the major/minor number schemes. If the kernel
>    tomorrow switches to randomly assign major and minor numbers to different
>    devices, it would work just fine (this is exactly what I'm proposing to do
>    in  2.7...)
> 3) This is the main reason udev is around. It provides the ability to name
>    devices in a persistent manner. More on that below.
> 4) udev emits D-BUS messages so that any other userspace program (like HAL)
>    can listen to see what devices are created or removed. It also allows
>    userspace programs to query it's [sic] database to see what devices are
>    present and what they are currently named as (providing a pointer into the
>    sysfs tree for that specific device node.)
>
> Constraints:
> 1) udev removes _all_ naming policies out of the kernel and into userspace.
> 2) udev defaults to using the LSB device naming standard. If users want to
>    deviate away from this standard (for example when naming some devices in a
>    persistent manner), it is easily possible to do so.
> 3) udev is small (49Kb binary) and is entirely in userspace, which is
>    swappable, and doesn't have to be running at all times.


[1]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/devices.txt?h=v5.12-rc2
[2]: https://lwn.net/Articles/65197/

## Using `udev`

Modern `udev` runs as a userspace daemon and listens for kernel uevents through
a `netlink` socket. When a device is connected, `udev` captures the uevents
emitted by the kernel, and collects device attributes under `/sys` for the
corresponding device. It then searches for a set of rules to determine the name
of the device file to create under `/dev` and the permissions to set, as well as
to run external scripts and to emit `D-BUS` sigals for listeners including the
desktop environment. In early 2012, `udev` became part of `systemd`. The `udev`
daemon can be viewed using

```
$ ps -e  | grep "udev"
  317 ?        00:00:00 systemd-udevd
```

### Structure of `udev` Rules

`udev` rules are organized as rule files in system rules directories
`/usr/lib/udev/rules.d` and `/usr/local/lib/udev/rules.d`, the volatile runtime
directory `/run/udev/rules.d` and the local administration directory
`/etc/udev/rules.d`. The rule files take effect in order, sorted by their
name. Files in `/etc` have the highest priority, files in `/run` take precedence
over files with the same name under `/usr`. All rules files use the same
namespace.

Rule files contain a collection of rule lines. Rules are all one line and each
rule consists of matchers and actions. The former determines the prerequisites
for the latter. Rules can be broken with '\' just before the newline. All
matching rules in all files are executed in the order of appearance.

Matchers and actions are 'key' 'operator' 'value' triplets. The syntax for
`udev` rule files can be shown using

```
man udev
```

### `udev` Debug

`udev` is monitored by program `udevadm`. The following information is printed
after inserting a thumb drive.

```
# udevadm monitor

monitor will print the received events for:
UDEV - the event which udev sends out after rule processing
KERNEL - the kernel uevent

KERNEL[44093.027199] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2 (usb)
KERNEL[44093.027359] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0 (usb)
KERNEL[44093.029067] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6 (scsi)
KERNEL[44093.029159] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/scsi_host/host6 (scsi_host)
KERNEL[44093.029287] bind     /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0 (usb)
KERNEL[44093.029424] bind     /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2 (usb)
UDEV  [44093.039780] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2 (usb)
UDEV  [44093.044882] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0 (usb)
UDEV  [44093.046671] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6 (scsi)
UDEV  [44093.048527] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/scsi_host/host6 (scsi_host)
UDEV  [44093.050414] bind     /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0 (usb)
UDEV  [44093.060044] bind     /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2 (usb)
KERNEL[44094.035381] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0 (scsi)
KERNEL[44094.035436] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0 (scsi)
KERNEL[44094.035453] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0/scsi_disk/6:0:0:0 (scsi_disk)
KERNEL[44094.035474] bind     /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0 (scsi)
KERNEL[44094.035491] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0/scsi_device/6:0:0:0 (scsi_device)
KERNEL[44094.035762] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0/scsi_generic/sg2 (scsi_generic)
KERNEL[44094.036019] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0/bsg/6:0:0:0 (bsg)
KERNEL[44094.037924] add      /devices/virtual/bdi/8:16 (bdi)
UDEV  [44094.038393] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0 (scsi)
UDEV  [44094.039097] add      /devices/virtual/bdi/8:16 (bdi)
UDEV  [44094.041687] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0 (scsi)
KERNEL[44094.041758] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0/block/sdb (block)
KERNEL[44094.041802] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0/block/sdb/sdb1 (block)
UDEV  [44094.046560] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0/scsi_disk/6:0:0:0 (scsi_disk)
UDEV  [44094.048087] bind     /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0 (scsi)
UDEV  [44094.050143] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0/scsi_device/6:0:0:0 (scsi_device)
UDEV  [44094.050311] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0/scsi_generic/sg2 (scsi_generic)
UDEV  [44094.051005] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0/bsg/6:0:0:0 (bsg)
UDEV  [44094.132364] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0/block/sdb (block)
UDEV  [44094.214713] add      /devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0/block/sdb/sdb1 (block)

```

The following command lists the attributes that can be used to match a device,
such as a thumb drive.

```
# udevadm info --attribute-walk /dev/sdb

Udevadm info starts with the device specified by the devpath and then
walks up the chain of parent devices. It prints for every device
found, all possible attributes in the udev rules key format.
A rule to match, can be composed by the attributes of the device
and the attributes from one single parent device.

  looking at device '/devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0/block/sdb':
    KERNEL=="sdb"
    SUBSYSTEM=="block"
    DRIVER==""
    ATTR{alignment_offset}=="0"
    ATTR{hidden}=="0"
    ATTR{capability}=="51"
    ATTR{ext_range}=="256"
    ATTR{stat}=="     552     3720    14358      684        1        0        1        4        0      596      596        0        0        0        0"
    ATTR{events_async}==""
    ATTR{discard_alignment}=="0"
    ATTR{events_poll_msecs}=="-1"
    ATTR{ro}=="0"
    ATTR{events}=="media_change"
    ATTR{removable}=="1"
    ATTR{size}=="240353280"
    ATTR{range}=="16"
    ATTR{inflight}=="       0        0"

  looking at parent device '/devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0/6:0:0:0':
    KERNELS=="6:0:0:0"
    SUBSYSTEMS=="scsi"
    DRIVERS=="sd"
    ATTRS{iorequest_cnt}=="0x254"
    ATTRS{dh_state}=="detached"
    ATTRS{max_sectors}=="240"
    ATTRS{state}=="running"
    ATTRS{iocounterbits}=="32"
    ATTRS{queue_depth}=="1"
    ATTRS{eh_timeout}=="10"
    ATTRS{model}==" SanDisk 3.2Gen1"
    ATTRS{type}=="0"
    ATTRS{iodone_cnt}=="0x254"
    ATTRS{ioerr_cnt}=="0x1"
    ATTRS{scsi_level}=="7"
    ATTRS{timeout}=="30"
    ATTRS{rev}=="1.00"
    ATTRS{blacklist}==""
    ATTRS{evt_soft_threshold_reached}=="0"
    ATTRS{evt_inquiry_change_reported}=="0"
    ATTRS{evt_media_change}=="0"
    ATTRS{device_blocked}=="0"
    ATTRS{evt_capacity_change_reported}=="0"
    ATTRS{vendor}==" USB    "
    ATTRS{evt_lun_change_reported}=="0"
    ATTRS{inquiry}==""
    ATTRS{device_busy}=="0"
    ATTRS{evt_mode_parameter_change_reported}=="0"
    ATTRS{queue_type}=="none"

  looking at parent device '/devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6/target6:0:0':
    KERNELS=="target6:0:0"
    SUBSYSTEMS=="scsi"
    DRIVERS==""

  looking at parent device '/devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host6':
    KERNELS=="host6"
    SUBSYSTEMS=="scsi"
    DRIVERS==""

  looking at parent device '/devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0':
    KERNELS=="1-1.2:1.0"
    SUBSYSTEMS=="usb"
    DRIVERS=="usb-storage"
    ATTRS{bNumEndpoints}=="02"
    ATTRS{supports_autosuspend}=="1"
    ATTRS{bAlternateSetting}==" 0"
    ATTRS{bInterfaceNumber}=="00"
    ATTRS{bInterfaceProtocol}=="50"
    ATTRS{authorized}=="1"
    ATTRS{bInterfaceSubClass}=="06"
    ATTRS{bInterfaceClass}=="08"

  looking at parent device '/devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2':
    KERNELS=="1-1.2"
    SUBSYSTEMS=="usb"
    DRIVERS=="usb"
    ATTRS{devnum}=="9"
    ATTRS{maxchild}=="0"
    ATTRS{product}==" SanDisk 3.2Gen1"
    ATTRS{bDeviceSubClass}=="00"
    ATTRS{bConfigurationValue}=="1"
    ATTRS{rx_lanes}=="1"
    ATTRS{removable}=="unknown"
    ATTRS{serial}=="0101287b9ee652db333a1560368abfae4422b1627bbda1aa660dfd4eb912a25474d10000000000000000000058462866008b2a0081558107a7295366"
    ATTRS{speed}=="480"
    ATTRS{bmAttributes}=="80"
    ATTRS{bMaxPacketSize0}=="64"
    ATTRS{bMaxPower}=="224mA"
    ATTRS{bNumConfigurations}=="1"
    ATTRS{bcdDevice}=="0100"
    ATTRS{bNumInterfaces}==" 1"
    ATTRS{version}==" 2.10"
    ATTRS{authorized}=="1"
    ATTRS{avoid_reset_quirk}=="0"
    ATTRS{ltm_capable}=="no"
    ATTRS{urbnum}=="1792"
    ATTRS{devpath}=="1.2"
    ATTRS{quirks}=="0x0"
    ATTRS{manufacturer}==" USB"
    ATTRS{idProduct}=="5581"
    ATTRS{configuration}==""
    ATTRS{idVendor}=="0781"
    ATTRS{bDeviceClass}=="00"
    ATTRS{bDeviceProtocol}=="00"
    ATTRS{tx_lanes}=="1"
    ATTRS{busnum}=="1"

  looking at parent device '/devices/pci0000:00/0000:00:1a.0/usb1/1-1':
    KERNELS=="1-1"
    SUBSYSTEMS=="usb"
    DRIVERS=="usb"
    ATTRS{bMaxPacketSize0}=="64"
    ATTRS{tx_lanes}=="1"
    ATTRS{devnum}=="2"
    ATTRS{bcdDevice}=="0000"
    ATTRS{bNumInterfaces}==" 1"
    ATTRS{bConfigurationValue}=="1"
    ATTRS{devpath}=="1"
    ATTRS{speed}=="480"
    ATTRS{bDeviceClass}=="09"
    ATTRS{ltm_capable}=="no"
    ATTRS{avoid_reset_quirk}=="0"
    ATTRS{rx_lanes}=="1"
    ATTRS{bDeviceProtocol}=="01"
    ATTRS{bDeviceSubClass}=="00"
    ATTRS{urbnum}=="254"
    ATTRS{removable}=="unknown"
    ATTRS{configuration}==""
    ATTRS{busnum}=="1"
    ATTRS{bNumConfigurations}=="1"
    ATTRS{version}==" 2.00"
    ATTRS{idProduct}=="0024"
    ATTRS{authorized}=="1"
    ATTRS{maxchild}=="6"
    ATTRS{bmAttributes}=="e0"
    ATTRS{idVendor}=="8087"
    ATTRS{quirks}=="0x0"
    ATTRS{bMaxPower}=="0mA"

  looking at parent device '/devices/pci0000:00/0000:00:1a.0/usb1':
    KERNELS=="usb1"
    SUBSYSTEMS=="usb"
    DRIVERS=="usb"
    ATTRS{interface_authorized_default}=="1"
    ATTRS{bNumConfigurations}=="1"
    ATTRS{rx_lanes}=="1"
    ATTRS{idVendor}=="1d6b"
    ATTRS{version}==" 2.00"
    ATTRS{bMaxPacketSize0}=="64"
    ATTRS{product}=="EHCI Host Controller"
    ATTRS{urbnum}=="46"
    ATTRS{configuration}==""
    ATTRS{idProduct}=="0002"
    ATTRS{ltm_capable}=="no"
    ATTRS{bcdDevice}=="0419"
    ATTRS{tx_lanes}=="1"
    ATTRS{bDeviceProtocol}=="00"
    ATTRS{removable}=="unknown"
    ATTRS{quirks}=="0x0"
    ATTRS{bDeviceSubClass}=="00"
    ATTRS{speed}=="480"
    ATTRS{bNumInterfaces}==" 1"
    ATTRS{authorized}=="1"
    ATTRS{devpath}=="0"
    ATTRS{serial}=="0000:00:1a.0"
    ATTRS{manufacturer}=="Linux 4.19.0-14-amd64 ehci_hcd"
    ATTRS{devnum}=="1"
    ATTRS{bmAttributes}=="e0"
    ATTRS{bMaxPower}=="0mA"
    ATTRS{busnum}=="1"
    ATTRS{maxchild}=="3"
    ATTRS{authorized_default}=="1"
    ATTRS{bConfigurationValue}=="1"
    ATTRS{avoid_reset_quirk}=="0"
    ATTRS{bDeviceClass}=="09"

  looking at parent device '/devices/pci0000:00/0000:00:1a.0':
    KERNELS=="0000:00:1a.0"
    SUBSYSTEMS=="pci"
    DRIVERS=="ehci-pci"
    ATTRS{uframe_periodic_max}=="100"
    ATTRS{companion}==""
    ATTRS{local_cpulist}=="0-3"
    ATTRS{dma_mask_bits}=="32"
    ATTRS{broken_parity_status}=="0"
    ATTRS{driver_override}=="(null)"
    ATTRS{ari_enabled}=="0"
    ATTRS{subsystem_vendor}=="0x17aa"
    ATTRS{device}=="0x1c2d"
    ATTRS{consistent_dma_mask_bits}=="32"
    ATTRS{local_cpus}=="f"
    ATTRS{d3cold_allowed}=="1"
    ATTRS{vendor}=="0x8086"
    ATTRS{class}=="0x0c0320"
    ATTRS{msi_bus}=="1"
    ATTRS{numa_node}=="-1"
    ATTRS{enable}=="1"
    ATTRS{revision}=="0x04"
    ATTRS{subsystem_device}=="0x21dd"
    ATTRS{irq}=="16"

  looking at parent device '/devices/pci0000:00':
    KERNELS=="pci0000:00"
    SUBSYSTEMS==""
    DRIVERS==""
```

### Creating `udev` rules

A rule matching the above thumb drive is created in file
`/etc/udev/rules.d/000-test.rules`

```
KERNEL=="sdb", SUBSYSTEM=="block", ATTRS{idVendor}=="0781", ATTRS{idProduct}=="5581", SYMLINK+="john-drive"
```

The rule matches kernel event device "sdb" and subsystem "block". Once a match
with the vendor and product ID of the thumb drive, as instructed, `udev` adds
symlink `/dev/john-drive` pointing to the device.

After the rule file is saved, the following command simulates running the rule

```
# udevadm test $(sudo udevadm info -q path -n /dev/sdb)
```

The following command confirms the rule is working

```
ls -l /dev/john-drive

lrwxrwxrwx 1 root root 3 Mar 10 20:29 /dev/john-drive -> sdb
```

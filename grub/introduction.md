# Introduction

Grand Unified Bootloader.

## How to learn

Like any other tool, GRUB is magic until you start to analyse minimal examples with it.

The best way to play with GRUB is to:

- create a few hello world multiboot OSes as in <https://github.com/cirosantilli/x86-bare-metal-examples/tree/c1211ace3670f24f7e2c5e3f4ae24effc778e1bb/hello-world-multiboot-c>
- make a disk image with `grub-install`, `grub-update`, `grub-mkrescue`
- boot up the images `qemu -hda image`

## Introduction

If you have a Linux dual boot, and you see a menu prompting you to choose the OS, there is a good chance that this is GRUB, since it is the most popular bootloader today.

It allows you basic graphical interaction even before starting any OS.

Everything is configurable, from the menu entries to the background image. This is why Ubuntu's GRUB is purple.

The main job for GRUB userspace utilities such as `grub-install` and `update-grub` is to look at the input configuration files, interpret them and write the output configuration information to the correct locations on the hard disk so that they can be found at boot time.

GRUB has knowledge about filesystems, and is able to read configuration files and the disk image from it.

## GRUB versions

GRUB has 2 versions

-   0.97, usually known just as GRUB, or Legacy GRUB.

-   GRUB >= 2, which is backwards incompatible, and has more features.

    GRUB 2 is still beta.

Some distros like Ubuntu have already adopted GRUB 2, while others are still using GRUB for stability concerns.

Determine your GRUB version with:

    grub-install -v

Here we discuss GRUB 2.

## Supported architectures

x86 is of course the primary... ARM was recently added in 2.0.2 it seems: <https://wiki.linaro.org/LEG/Engineering/Kernel/GRUB>

## Configuration files

Input files:

-  `/etc/grub.d/*`
- `/etc/default/grub`

Generated files and data after `sudo update-grub`:

- `/boot/grub/grub.cfg`
- MBR bootstrap code

### /etc/default/grub

Shell script sourced by `grub-mkconfig`.

Can defined some variables which configure grub, but is otherwise an arbitrary shell script:

    sudo vim /etc/default/grub

-   `GRUB_DEFAULT`: default OS choice if cursor is not moved:

    Starts from 0, the order is the same as shown at grub OS choice menu:

        GRUB_DEFAULT=0

    The order can be found on the generated `/boot/grub/grub.cfg`: you have to count the number of `menuentry` calls.

    To select sub-menus, which are created with the `submenu` call on `/boot/grub/grub.cfg`, use:

        GRUB_DEFAULT='0>1'

    You can also use OS name instead of a number, e.g.:

        GRUB_DEFAULT='Ubuntu'

    For a line from `/boot/grub/grub.cfg` of type:

        menuentry 'Ubuntu'

-   `GRUB_TIMEOUT` : time before auto OS choice in seconds

-   `GRUB_CMDLINE_LINUX_DEFAULT`: space separated list of Kernel boot parameters.

    Sample:

        GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"

    The parameters will not be discussed here.

    Those parameters can also be edited from the boot menu for a single session by selecting the partition and clicking `e`.

	-   useless options on by default on Ubuntu 12.04 which you should really remove because they hide kernel state and potentially useful debug information:

        - `quiet`: suppress kernel messages.
        - `splash`: shows nice and useless image while the kernel is booting. On by default on Ubuntu 12.04. Remove this useless option,

### /etc/grub.d/

Contains executables.

Each one is called in alphabetical order, and its stdout is used by GRUB.

A common choice for custom scripts in Ubuntu 14.04 is `40_custom`.

Create a menu entry:

    #!/bin/sh -e
    echo "stdout"
    echo "stderr" >&2
    cat << EOF
    menuentry "menuentry title" {
    set root=(hd0,1)
    -- boot parameters --
    }
    EOF

You will see `stdout` when running `update-grub`. stderr is ignored.

`set root=(hd0,1)` specifies the partition, here `sda1`. `hd0` means first device,
`1` means first partition. Yes, one if 0 based, and the other is 1 based.

`-- boot parameters --` depends on your OS.

Linux example:

    linux /boot/vmlinuz
    initrd /boot/initrd.img

Windows example:

    chainloader (hdX,Y)+1

It is common to add one OS menu entry per file so that it is easy to change their order (just change alphabetical order).

## Configuration scripts

### update-grub

Just calls:

    grub-mkconfig -o /boot/grub/grub.cfg

### grub-mkconfig

Called by `update-grub` as:

    grub-mkconfig -o /boot/grub/grub.cfg

Important actions:

- sources `/etc/default/grub`
- sources `/etc/default/grub.d/*.cfg`, which may override options in `/etc/default/grub`
- runs scripts under `/etc/grub.d`, which use the variables defined in the above sourced files

### grub-install

Given a `/boot/grub/grub.cfg` in some filesystem, install GRUB to some hard disk.

Interpret input configuration files and update the MBR on the given disk:

    sudo grub-install /dev/sda

If for example you install a new Linux distro, and you want to restore your old distro's GRUB configuration, you must log into the old distro and do `grub-install`, therefore telling your system via the MBR to use the installation parameters given on the old distro.

TODO get a minimal example working using a minimal kernel from: <https://github.com/cirosantilli/x86-bare-metal-examples>:

    img="a.img"
    dd if=/dev/zero of="$img" bs=1024 count=64
    loop="$(sudo losetup -f --show "$img")"
    printf 'o\nn\np\n1\n\n\nw\n' | sudo fdisk "$loop"

    sudo kpartx -av "$img"
    ls /dev/mapper

    echo y | mke2fs -t ext4
    sudo mount "/dev/mapper/${loop}p1" d


    # Need a new Ubuntu.
    #sudo losetup --show -f -P test.img


    sudo grub-install /dev/loop0

    mkdir -p d
    mount /dev/loop0 d

    #grub-install --boot-directory=d /dev/sdb

### grub-mkrescue

Generates a rescue image from a root filesystem.

Example: <https://github.com/cirosantilli/x86-bare-metal-examples/blob/48614b45fa6edeb97adbaad942595a4c25216113/multiboot/hello-world/Makefile#L6>

You can then burn the output to an USB or CD

Vs `grub-install`: generates a live boot USB / CD, but does not use the USB as a filesystem.

Easier to setup however.

### os_prober

Looks for several OS and adds them automatically to GRUB menu.

Recognizes Linux and Windows.

TODO how to use it

## rescue prompt

If things fail really badly, you may be put on a `rescue >` prompt.

You are likely better off reinstalling things correctly in practice. But here go a few commands you can use from there.

<https://www.linux.com/learn/tutorials/776643-how-to-rescue-a-non-booting-grub-2-on-linux/>

-   `ls`

-   `ls (hd0,1)/`

-   `cat (hd0,1)/etc/issue`

-   Boot:

        set root=(hd0,1)
        linux /boot/vmlinuz-3.13.0-29-generic root=/dev/sda1
        initrd /boot/initrd.img-3.13.0-29-generic
        boot

## grub.cfg

TODO:

- where is the format documented?
- what is set? No relation to the Bash version: <http://unix.stackexchange.com/questions/197578/linux-set-command-for-local-variables>

Stuff I've deduced for 2.0:

### timeout

No timeout on boot menu:

    set timeout=0

### default

Default no Nth (zero based) entry of boot menu:

    set default="0"

### menuentry

The following commands can be used inside a menu entry, e.g.:

    menuentry "main" {
    }

Point to a multiboot file:

	multiboot /boot/main.elf

E.g.: <https://github.com/cirosantilli/x86-bare-metal-examples/blob/48614b45fa6edeb97adbaad942595a4c25216113/multiboot/hello-world/iso/boot/grub/grub.cfg>

Load a linux kernel with a given root filesystem:

	linux /boot/bzImage
	initrd /boot/rootfs.cpio.gz

You can pass kernel command line arguments with:

	linux /boot/bzImage BOOT_IMAGE=/boot/vmlinuz-3.19.0-28-generic root=UUID=2a49bac4-b9dd-466d-9c0c-c432aa4ca086 ro loop.max_part=15

You can then check that they've appeared under `cat /proc/cmdline`.

## Alternatives

- `syslinux`: Linux specific. Used by default by the kernel, e.g. on 4.2 `make isoimage`.
- LILO: old popular bootloader, largely replaced by GRUB now.

## Legacy

Documentation: <http://www.gnu.org/software/grub/manual/legacy/grub.html>

### kernel

Directive used to boot *both* multiboot and Linux.

Got split up more or less into `multiboot` and `linux` directives.

## Bibliography

-   <https://www.gnu.org/software/grub/grub-documentation.html>

-   <http://www.dedoimedo.com/computers/grub-2.html>

    Great configuration tutorial.

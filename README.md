# thinkpad-yoga-3rd-gen-acpi

An attempt to patch the ACPI for the ThinkPad X1 Yoga, 3rd Generation laptop, following [this gist][gist] put together
by [@javanna][@javanna].

My environment is Ubuntu 16.04 (xenial), running the latest HWE kernel from Ubuntu 18.04, which presently is
`4.15.0-38-generic`.

## Steps

First [obtain and compile a recent `iasl`][iasl-download]. This is mandatory. It straight up won't work otherwise.
To do this, you'll likely need to install a bunch of packages: `build-essential`, `m4`, `bison`, `flex`, and some
other packages. Compile with `make`. If it fails, read the error messages and try to find packages that provide
the utilities it's lacking.

Next, boot into at least Ubuntu 18.04 on a live CD/USB. I'm not sure if this is necessary, but I'll explain in more
detail below.

Having compiled a recent `iasl` and having a runtime in Ubuntu 18.04, dump the ACPI DSDT table from the hardware:

```
cat /sys/firmware/acpi/tables/DSDT > dsdt.dsl
```

Next, decompile the table into Assembly using `iasl`:

```
bin/iasl -d dsdt.aml
```

Download the patch:

```
wget http://kernel.dk/acpi.patch
```

Apply the patch:

```
patch --verbose < acpi.patch
```

There will be 1-2 conflicts. View the `*.rej` file to see the patches that failed and manually apply them.

After fixing the patching, compile back into bytecode:

```
bin/iasl -ve -tc dsdt.dsl
```

Let's create a CPIO archive containing our new ACPI DSDT table:

```
mkdir -p kernel/firmware/acpi
cp dsdt.aml kernel/firmware/acpi
find kernel | cpio -H newc --create > acpi_override
```

Create a directory to hold the override file:

```
mkdir -p /lib/acpi
cp acpi_override /lib/acpi
```

Install the initramfs hook provided here to bake the changes into the initramfs:

```
cp etc/initramfs-tools/hooks/acpi-override /etc/initramfs-tools/hooks/
```

To make sure that you don't get into an unable-to-boot state, make a backup of your current bootable initramfs
and friends:

```
find /boot -iname "*$(uname -r)*" -exec cp {} {}-safe \;
```

Bake the initramfs and reboot into it:

```
update-initramfs -k all -c
```

The above command rebakes _all_ kernels currently installed. You can update only the current kernel via
`update-initramfs -u -k $(uname -r)`.

After this, `reboot` into the new kernel and observe that it all works :tada:

```
$ dmesg | grep ACPI | grep supports
ACPI: (supports S0 S3 S4 S5)
$ cat /sys/power/mem_sleep
s2idle [deep]
```

Praise the Sun! It works! :raised_hands: :sun_with_face:

This also fixed a resume bug for me which broke all of my pointer inputs on the laptop like the touchscreen, the
Wacom pointer, the trackpoint, and the track pad.

## License

Licensed at your discretion under either:

 - [MIT License](./LICENSE-MIT)
 - [Apache License, Version 2.0](./LICENSE-APACHE)

 [gist]: https://gist.github.com/javanna/38d019a373085e1ba0c784597bc7ec73
 [iasl-download]: https://acpica.org/downloads
 [@javanna]: https://github.com/javanna

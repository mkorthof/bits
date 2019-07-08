
# BITS BUILD INSTRUCTIONS

You probably want to apply patches from 'bits-patches' dir first.

Source: <https://github.com/intel/luv-yocto/tree/master/meta-luv/recipes-bsp/bits/bits>

## Flags

CFLAGS="-Wno-error"
CFLAGS+="-fno-PIC"
CFLAGS+="-fno-PIE"
LDFLAGS="-no-pie"

and to speeds things up: '-j4' (or higher)

## Compile

First try the usual `make`

## Issues

If compiling using the bits Makefile doesnt work properly:

- git clone to ~/src/bits
- set buildnum manually, see below
- in case of compile errors manually build grub and/or python
- edit bits Makefile: comment `$(Q)rm -rf '$(workdir)'` (2x)

``` bash
buildnum=$(( $(cd ~/src/bits && GIT_CEILING_DIRECTORIES=~src/bits git rev-list HEAD 2>/dev/null | wc -l) + 2000 ))
````

## Grub

Make sure to use these flags:

``` bash
CFLAGS="-Wno-error -fno-PIC -fno-PIE" LDFLAGS="-no-pie" ./configure
CFLAGS="-Wno-error -fno-PIC -fno-PIE" LDFLAGS="-no-pie" make
```

### Compiling build-grub-i386-pc

this runs `autogen.sh` :

``` bash
make build-grub-i386-pc
```

In case of compile errors, manually compile from 'build/grub' dir instead of using bits Makefile

Optionally after running once comment it out before building same buildnum again:

``` Makefile
       $(Q)cd '$(grub-src)' && ./autogen.sh
```

and

``` Makefile
       $(Q)mkdir '$(workdir)/grub-build-$*'
       $(Q)cd '$(workdir)/grub-build-$*' && '$(grub-src)/configure' --prefix='$(grub-prefix)-$*' --libdir='$(grub-prefix)/lib' --program-prefix= --target=$(firstword $(subst -, ,$*)) --with-platform=$(lastword $(subst -, ,$*)) --disable-nls --disable-efiemu --disable-grub-emu-usb --disable-grub-emu-sdl --disable-grub-mkfont --disable-grub-mount --disable-device-mapper --disable-libzfs MAKEINFO=/bin/true TARGET_CFLAGS='-Os -Wno-discarded-array-qualifiers'
       $(Q)cd '$(workdir)/grub-build-$*' && $(MAKE) install
```

instead run this by hand:

``` bash
for dir in i386-efi i386-pc x86_64-efi; do
  mkdir bits-${buildnum}/boot/grub/$dir
  for suffix in img lst mod; do
    cp grub-inst/lib/grub/${dir}/*.$suffix bits-${buildnum}/boot/grub/${dir}/
  done
done

```
### Compiling i386-efi-img x86_64-efi-img

``` bash
CFLAGS="-Wno-error -fno-PIC -fno-PIE" LDFLAGS="-no-pie" make build-grub-i386-efi build-grub-x86_64-efi
````

### Core.img fix

 Running `make fixup-grub-i386-pc` compiles 'build-grub-i386-pc' which we already did so to only  create 'lnxcore.img':

``` bash
cat build/bits-${buildnum}/boot/grub/i386-pc/lnxboot.img build/bits-${buildnum}/boot/grub/core.img > build/bits-${buildnum}/boot/grub/lnxcore.img
rm build/bits-${buildnum}/boot/grub/core.img
```

## Python

In case of compile errors, manually compile from 'build/python-host' dir instead of using bits Makefile and use these flags again:

``` bash
CFLAGS="-Wno-error -fno-PIC -fno-PIE" LDFLAGS="-no-pie" ./configure
make CFLAGS='-Wno-error -fno-PIC -fno-PIE' LDFLAGS='-no-pie'
```

### python-host

``` bash
cd build/python-host
CFLAGS="-Wno-error -fno-PIC -fno-PIE" LDFLAGS="-no-pie" make build-grub-i386-efi build-grub-x86_64-efi
```

Again, optionally you can comment out these lines now:

``` Makefile
       $(Q)cd '$(python-host-src)' && ./configure
       $(Q)cd '$(python-host-src)' && $(MAKE)
```

### Check

Make sure these compiled OK

Grub:

- i386-pc-img: boot/grub/core.img
- i386-efi-img: efi/boot/bootia32.efi
- x86_64-efi-img: efi/boot/bootx64.efi
- i386-pc-extra-modules: biosdisk
- common-modules: fat part_msdos part_gpt iso9660

Python:

- build/python-host
- build/python-lib

### Install

``` bash
  make install-syslinux install-doc install-grub-cfg install-log install-bitsversion install-bitsconfigdefaults \
    install-toplevel-cfg install-bits-cfg install-readme install-news install-src-bits install-pylib \
    bytecompile-pylib install-bits-python bytecompile-bits-python install-install install-copying
```

- `make install-src-readme` creates 'build/bits-${buildnum}/README.txt'
- `make install-src-bits` creates 'build/bits-${buildnum}/boot/src
- ...etc

## Create .iso

``` bash
build/grub-build-x86_64-efi/grub-mkrescue -o bits-${buildnum}.iso build/bits-${buildnum}
```

## Create USB image

First create sparse file as loop device for the image then use fdisk to create a bootable partition:

``` bash
truncate -s 512M /tmp/bits-${buildnum}.img
fdisk /tmp/bits-${buildnum}.img
```

now press 'n' then 4x \<ENTER\>, 'a', 'w'

``` bash
losetup -P /dev/loop2 /tmp/bits-${buildnum}.img
mkfs.fat /dev/loop2p1
mkdir /mnt/loop /mnt/iso
mount /dev/loop2p1 /mnt/loop
mount bits-${buildnum}.iso /mnt/iso
cp -r /mnt/iso/boot /mnt/iso/efi /mnt/loop
syslinux /dev/loop2p1
cat /usr/lib/syslinux/mbr/mbr.bin > /dev/loop2
umount /mnt/loop
mv /tmp/bits-${buildnum}.img ~/
```

So mostly like INSTALL.txt but replace 'sdb' with '/dev/loop2' and 'sdb1' with '/dev/loop2p1' and some extra steps added

# Testing

Virtualization can be used for a quick test:

``` bash
kvm -cdrom bits-${buildnum}.iso -nographic -vnc 0.0.0.0:0
kvm -hda bits-${buildnum}.img -nographic -vnc 0.0.0.0:0
```

( or use 'qemu-system-x86_64' )

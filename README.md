
# BIOS Implementation Test Suite (BITS)

The reason for the existance of this fork is that manufacturers often do not support older motherboards/BIOS with updates.
This leads to vulnerabilities like Meltdown/Spectre not getting patched while the microcode *is* available.
This fork can be used to automatically load microcode before loading OS (using GRUB).

I have successfullly tested this by booting from USB stick and Windows 10 (and Linux).

The last 2 entries in the menu use:
- AIO Boot to boot Windows (https://www.aioboot.com/en/boot-windows-grub2/)
- Sample GRUB script to autodetect operating systems

Original documentation:
- [INSTALL](INSTALL)
- [README.txt](README.txt)
- [Documentation](Documentation)

Changes:
- automatically load microcode 
- auto detect OS using various methods and boot it

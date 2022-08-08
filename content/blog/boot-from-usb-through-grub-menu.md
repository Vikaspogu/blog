+++ 
date = 2020-12-01
title = "Boot from USB through grub menu"
description = "Boot OS from grub menu"
slug = "" 
tags = ["boot", "grub"]
categories = []
externalLink = ""
series = []
socialShare=true
+++

First make sure you have secure boot disabled from the firmware settings. Once you are in grub command line type `ls` to list all partitions

```bash
grub>ls 
(hd0) (hd0,gpt1) (hd1) (hd1,gpt8) (cd0))
```

Type `ls (cd0)` to get UUID of device

```bash
grub>ls (hd0,gpt1) 
Partition hd0,gpt1: Filesystem type fat - Label `CES_X64FREV`, UUID 4099-DBD9 Partition start-512 Sectors...
```

Note the UUID of you usb drive, shown in above command

Type the following commands

```bash
insmod part_gpt
insmod fat
insmod search_fs_uuid
insmod chain
search --fs-uuid --set=root 409-DBD9
```

Replace `UUID` of your device

Now we select the efi file to boot from

```bash
chainloader /efi/boot/bootx64.efi
boot
```

That's it, That should boot the usb drive

# Kernel driver for Chinese clone of DS2413: 3A 2100H
-------------------------------------------------------------------------------
I've found a set of 10pcs of DS2413 on aliexpress for very good price (about 10USD for 10pcs...) - almost 2,5 times cheaper than in my country from electronic warehouse - cool, so I ordered a one set...
Surprise, surprise, I got 3A 2100H chips - package identical, I connected for quick test using probes to my 1-wire scanner... I got some serial number (I didn't pay much attention to in at this moment) so I think: ok, works, might be some custom marking, I'll check it later... 
Few week later :smile: , I made a PCB, hooked DS2413-clone to Raspberry Pi, enabled 1-wire overlay, etc... Everything is connected Pi is running let's see if it "still works" :smile:. Quick check:

```pi@piv2:~ $ ls /sys/bus/w1/devices
85-1003c073b2be  w1_bus_master1```

Great, there is something: 85-xxxxxx . Now, If I remember there is already kernel driver for ds2413... Let's inspect "device directory" and kernel modules:
```pi@piv2:~ $ ls /sys/bus/w1/devices/85-1003c073b2be
driver  id  name  rw  subsystem  uevent```
So I have `rw` file for accessing, but nothing with ds2413 in modules. Let's do some google-fu how to handle...
A two hours later, I'm not happy anymore :disappointed:, looks like I got clones of DS2413 with own "Family code" od 0x85.. ehhhh...
One solution is hex-editing ds2413 module - too ugly, so I decided I'll try to write own module (or more likely copy-paste :smile:).

So I "git"-ed latest raspberry pi kernel sources :wink: .
After looking a little around in 1-wire driver code, adding support for 2100H was rather easy: edit few files, rename some functions, add family code to master 1-wire module.

**KERNEL VERSION: 4.4.y (4.4.7) from 2016-04-17**

## Edited files
### ```linux/drivers/w1/w1_family.h```
Under line 47:
```#define W1_THERM_DS28EA00	0x42```

Add line:
```#define W1_FAMILY_2100H		0x85 // DS2413-2100H family```

### ```linux/drivers/w1/slaves/Makefile```
under line 8 :
```obj-$(CONFIG_W1_SLAVE_DS2413)   += w1_ds2413.o```

Add line:
```obj-$(CONFIG_W1_SLAVE_2100H)    += w1_ds2413_2100h.o```

### ```linux/drivers/w1/slaves/Kconfig```
Under block that start at line 35 :
```config W1_SLAVE_DS2413
        tristate "Dual Channel Addressable Switch 0x3a family support (DS2413)"
        help
          Say Y here if you want to use a 1-wire
          DS2413 Dual Channel Addressable Switch device support
```
Add block:

```config W1_SLAVE_2100H
        tristate "Dual Channel Addressable Switch 0x85 family support (clone DS2413: 3A 2100H)"
        help
          Say Y here if you want to use a 1-wire
          DS2413 clone: 3A 2100H - Dual Channel Addressable Switch device support
```

## Copy file
File: **w1_ds2413_2100h.c** copy to directory:
```linux/drivers/w1/slaves/```


# Compiling module

I didn't managed to compile module,so I didn't want to waste more time on tries, I just go for whole kernel compilation. Anyway it took less than hour :smile: - while my attempts  to compile only module (and make it working) took me 2 or 3 hours :smile:.

After calling ```make mrproper``` or ```make bcm2709_defconfig``` copy my kernel configuration file ```.config``` . 
Now follow standard guide for compiling kernel for Raspberry Pi - I have Pi2 so I did:
```
cd linux
KERNEL=kernel7
make bcm2709_defconfig
rm .config
cp config-my .config
make -j4 zImage modules dtbs
sudo make modules_install
sudo cp arch/arm/boot/dts/*.dtb /boot/
sudo cp arch/arm/boot/dts/overlays/*.dtb* /boot/overlays/
sudo cp arch/arm/boot/dts/overlays/README /boot/overlays/
sudo cp /boot/kernel7.img /boot/kernel7old.img
sudo rm /boot/kernel7.img
sudo scripts/mkknlimg arch/arm/boot/zImage /boot/kernel7.img
sudo reboot```
I had no errors while compiling and running commands :smile:
# Testing
Rebot done, I logged into ssh - kernel works ok :smile:
So, let's check what I have got in my device directory nad what is in modules...
```pi@piv2:~ $ lsmod
Module                  Size  Used by
cfg80211              419000  0
rfkill                 16082  1 cfg80211
snd_bcm2835            25076  0
snd_pcm                75278  1 snd_bcm2835
w1_ds2413_2100h         1450  0
bcm2835_rng             1911  0
snd_timer              19224  1 snd_pcm
bcm2835_gpiomem         3040  0
bcm2835_wdt             3225  0
spi_bcm2835             7286  0
i2c_bcm2708             4770  0
w1_gpio                 3657  0
snd                    51621  3 snd_bcm2835,snd_timer,snd_pcm
wire                   25091  2 w1_gpio,w1_ds2413_2100h
cn                      4374  1 wire
uio_pdrv_genirq         3100  0
uio                     8064  1 uio_pdrv_genirq
i2c_dev                 5923  0
fuse                   83525  1
ipv6                  345491  36```
Cool **w1_ds2413_2100h** is loaded :thumbsup: ! , go next:
```pi@piv2:~ $ ls /sys/bus/w1/devices/85-1003c073b2be
driver  id  name  output  state  subsystem  uevent```
No **rw** :grin: , there is **output** and **state** :grinning: .
Let's test it, I hooked 2 LEDs to 3,3V with resistors to PIOA & PIOB...
It took me actually about an hour to figure out how to read and write (and understand what I see, what I need to write) from/to 2100H PIOs .

But to save you troubles I've written a bash script.

# Test script
Just run script without parameters:
```pi@piv2:~ $ ./2100h
Small script for testing driver for 1-wire 2100H (clone of DS2413)
Author: saper_2 (Przemyslaw W.)
Version: 0.1 , 2016-04-18 , License: GNU GPL v.XYZ / MPL / CCA :)


Usage:
[sudo] ./2100h cmd [addr] [state PIOA] [state PIOB]
Parameters:
  cmd - required:
        s - print all found devices in format: [device_class]-[device_id]
        r - read GPIO state and print in binary
            (see DS2413 cmd PIO ACCESS READ for bit description)
        w - write new GPIO state. For more info see
            DS2413 datasheet, command PIO ACCESS WRITE
  addr - device address, required for cmd: 'r' and 'w'.
         Format: 85-xxxxxxxxxxxx  (xxx - device ID returned by 's')
  state - required for 'w', 0-1:
          0 - PIOx=0
          1 - PIOx=1```

## Device list
```pi@piv2:~ $ ./2100h s
Small script for testing driver for 1-wire 2100H (clone of DS2413)
Author: saper_2 (Przemyslaw W.)
Version: 0.1 , 2016-04-18 , License: GNU GPL v.XYZ / MPL / CCA :)


Found 1-Wire devices:
85-1003c073b2be```

## Read PIOs
```pi@piv2:~ $ ./2100h r 85-1003c073b2be
Small script for testing driver for 1-wire 2100H (clone of DS2413)
Author: saper_2 (Przemyslaw W.)
Version: 0.1 , 2016-04-18 , License: GNU GPL v.XYZ / MPL / CCA :)


Reading device 2100H '85-1003c073b2be' IOs (xxd output):
0000000: 00001111                                               .
   Bits: 76543210
         |||||||+- PIOA Pin state
         ||||||+-- PIOA Out Latch state
         |||||+--- PIOB Pin state
         ||||+---- PIOB Out Latch state
         |||+----- Bit 0 inverted
         ||+------ Bit 1 inverted
         |+------- Bit 2 inverted
         +-------- Bit 3 inverted
```
For more information check DS2413 datasheet "PIO ACCESS READ [F5h]".

## Write PIOs

To write you need a root privileges, if you don't then:
```pi@piv2:~ $ ./2100h w 85-1003c073b2be 1 1
Small script for testing driver for 1-wire 2100H (clone of DS2413)
Author: saper_2 (Przemyslaw W.)
Version: 0.1 , 2016-04-18 , License: GNU GPL v.XYZ / MPL / CCA :)


Trying to set PIOs to: PIOA=1 , PIOB=1 (DS2413_CMD=0x5A param=0x03)
To write to hardware device you need root privileges.
Run script with sudo or under root account.```

Using sudo:
```pi@piv2:~ $ sudo ./2100h w 85-1003c073b2be 1 1
Small script for testing driver for 1-wire 2100H (clone of DS2413)
Author: saper_2 (Przemyslaw W.)
Version: 0.1 , 2016-04-18 , License: GNU GPL v.XYZ / MPL / CCA :)


Trying to set PIOs to: PIOA=1 , PIOB=1 (DS2413_CMD=0x5A param=0x03)
Everything looks fine on this side, check PIOs state...```

# End photos
![alt Photos](rpi-2100h-f1.jpg?raw=true)
![alt Photos](rpi-2100h-f2.jpg?raw=true)
![alt Photos](rpi-2100h-f3.jpg?raw=true)


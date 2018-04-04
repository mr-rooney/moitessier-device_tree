Introduction
============

The nav.HAT driver needs the proper device tree information for initialization.
There are several approaches to store this information:
  1. Add the information to the proper device tree file for your plattform in
     the linux source directory <KERNEL_SRC>/arch/<ARCHITECTURE>/boot/dts/<PLATFORM>.dts.
     For the Raspberry Pi Model B it is <KERNEL_SRC>/arch/arm/boot/dts/bcm2710-rpi-3-b.dts.
  2. Create a device tree overlay file and tell the kernel to use it during booting. This is
     done for the Raspberry Pi using the config.txt file located on the boot partition of the
     SD card.
  3. Create a device tree file and flash it into the ID EEPROM located on the HAT.

The practical implementation of the approaches is explained below. The nav.HAT is designed for
the Raspberry Pi 3 Model B, however you might use the description with minor adaptations for
other plattforms as well.

Placeholders for paths/names are identified by <...>. This is not valid for the device tree information
itself, as <...> is part of the device tree syntax. 

Approach 1
==========

This approach is not really recommended, as you would need to change the device tree whenever new kernel
sources are used. 

Add the following lines to the end of the platform device tree <KERNEL_SRC>/arch/arm/boot/dts/bcm2710-rpi-3-b.dts

```
**** CONTENT TO ADD TO BCM2710-RPI-3-B.DTS - START ****

&spi0 {
    /delete-node/ spidev@0;
    navidev0: navidev@0{
		compatible = "pe,navidev";
		#address-cells = <1>;
		#size-cells = <0>;
        reg = <0>;
		spi-max-frequency = <12000000>;
        spi-min-frequency = <1000000>;
        status = "okay";

        boot: boot{
            gpios = <&gpio 17 GPIO_ACTIVE_LOW>;
            pe,name = "IMC BOOT";
            pe,direction = "output";
            status = "okay";   
        };

        reset: reset{
            gpios = <&gpio 18 GPIO_ACTIVE_LOW>;
            pe,name = "IMC RESET";
            pe,direction = "output";
            status = "okay";   
        };

        imc_req: imc_req{
            gpios = <&gpio 24 GPIO_ACTIVE_LOW>, <&gpio 22 GPIO_ACTIVE_LOW>;
            pe,name = "IMC REQ";
            pe,direction = "output";
            status = "okay";   
        };

        imc_irq: imc_irq{
            gpios = <&gpio 23 GPIO_ACTIVE_LOW>, <&gpio 27 GPIO_ACTIVE_LOW>;
            pe,name = "IMC IRQ";
            pe,direction = "input";
            status = "okay";   
        };
	};
};

**** CONTENT TO ADD TO BCM2710-RPI-3-B.DTS - END ****
```

Cross compile the device tree:
    ```    
    host> cd <KERNEL_SRC>
    host> make -j2 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- dtbs
    ```

Copy the device tree to the Raspberry Pi SD card. If SSH is enabled, you can use the scp command, otherwise
use a card reader.
    ```
    host> sudo scp arch/arm/boot/dts/*.dtb pi@<RASPI_IP_ADDR>:/media/pi/boot
    ```

You need to reboot the Raspberry Pi for the changes to take effect.

Approach 2
==========

You might also use the device tree overlay functionality that is supported by the Raspberry Pi. Device 
tree overlays are located on the boot partion in the folder *overlays* of the Raspberry Pi SD card.
This approach might be used if you either don't have a HAT supporting the ID EEPROM or if you don't want to 
create an EEPROM image all the time during development or debugging.

First you need to create the overlay file (*.dts) with the following content:

```
**** CONTENT NAVIDEV.DTS - START ****

/dts-v1/;
/plugin/;

/ {
    compatible = "brcm,bcm2708";

    fragment@0 {
        target = <&spi0>;
        __overlay__ {
            /delete-node/ spidev@0;
            navidev0: navidev@0{
        		compatible = "pe,navidev";
        		#address-cells = <1>;
        		#size-cells = <0>;
                reg = <0>;
        		spi-max-frequency = <12000000>;
                spi-min-frequency = <1000000>;
                status = "okay";

                boot: boot{
                    gpios = <&gpio 17 1>;
                    pe,name = "IMC BOOT";
                    pe,direction = "output";
                    status = "okay";   
                };

                reset: reset{
                    gpios = <&gpio 18 1>;
                    pe,name = "IMC RESET";
                    pe,direction = "output";
                    status = "okay";   
                };

                imc_req: imc_req{
                    gpios = <&gpio 24 1>, <&gpio 22 1>;
                    pe,name = "IMC REQ";
                    pe,direction = "output";
                    status = "okay";   
                };

                imc_irq: imc_irq{
                    gpios = <&gpio 23 1>, <&gpio 27 1>;
                    pe,name = "IMC IRQ";
                    pe,direction = "input";
                    status = "okay";   
                };
        	};
        };
    };
};

**** CONTENT NAVIDEV.DTS - END ****
```

Cross compile the device tree overlay:
    ```
    host> <KERNEL_SRC>/scripts/dtc/dtc -@ -I dts -O dtb -o naviDev.dtbo naviDev.dts
    ```

Copy the device tree overlay to the Raspberry Pi SD card. If SSH is enabled, you can use the scp command, otherwise use a card reader.
    ```
    host> sudo scp naviDev.dtbo pi@<RASPI_IP_ADDR>:/media/pi/boot/overlays
    ```

Modify the file *config.txt* located on the boot partition of the Raspberry Pi SD card and add the following
line at the end of this configuration file. This will ensure, that the proper overlay file will be loaded
during booting.
    ```
    dtoverlay=naviDev
    ```

You need to reboot the Raspberry Pi for the changes to take effect.

Note: You should definitely avoid using approach 1 and 2 simultaneously.


Approach 3
==========

You should use this approach if your HAT supports an ID EEPROM. This is definitely the case for the nav.HAT.
This approach is similar to approach 2. You use the same device tree overlay information and you compile
it exactly the same as described above.
The main difference to approach 2 is, that we flash this compiled device tree blob into the EEPROM. The
EEPROM is read in a very early booting stage of the linux kernel. If the information is valid it is used,
otherwise it is ignored.

Complete the following steps.
1. Create the device tree overlay file as described in approach 2.
2. Cross compile the device tree overlay file as described in approach 2.
3. Create file eeprom_settings.txt with the content below.
4. Create the EEPROM image:
       ```
       host> eeprom/eepmake eeprom_settings.txt naviDev.eep naviDev.dtbo
       ```
5. Add the following lines to *config.txt* on the boot partition of the Raspberry Pi SD card
   (1) to enable the I2C0 interface for EEPROM writing and (2) loading the device tree overlay file 
   to access the HAT at all:
       ```
       dtparam=i2c_vc=on
       dtoverlay=naviDev
       ```
   Note: The device tree overlay is only required, if the HAT related device tree information is not
   available through another source (e.g. previously loading from ID EEPROM).
6. Reboot the Raspberry Pi for changes to take effect.
7. Disable the write protection for the ID EEPROM (requires nav.HAT driver and control application):
       ```
       raspi> sudo insmod naviDev.ko              (if not already loaded)
       raspi> ./naviDev_ctrl.o /dev/naviDev.ctrl 6 0
       ```
8. Write the EEPROM:
       ```
       raspi> sudo ./eepflash.sh -w -f=naviDev.eep -t=24c32 -d=0 -a=50
       ```
9. Restore *config.txt* and reboot. The device tree will be loaded from the ID EEPROM upon reboot.

The device tree information for the HAT can be read from /proc/device-tree/hat and /proc/device-tree/soc/spi@7e204000/navidev@0


```
**** CONTENT EEPROM_SETTINGS.TXT - START ****

########################################################################
# EEPROM settings text file
#
# Edit this file for your particular board and run through eepmake tool,
# then use eepflash tool to write to attached HAT ID EEPROM 
#
# Tools available:
#  eepmake   Parses EEPROM text file and creates binary .eep file
#  eepdump   Dumps a binary .eep file as human readable text (for debug)
#  eepflash  Write or read .eep binary image to/from HAT EEPROM
#
########################################################################

########################################################################
# Vendor info

# 128 bit UUID. If left at zero eepmake tool will auto-generate
# RFC 4122 compliant UUID
product_uuid 00000000-0000-0000-0000-000000000000

# 16 bit product id
product_id 0x0000

# 16 bit product version
product_ver 0x0000

# ASCII vendor string  (max 255 characters)
vendor "CSOFT"

# ASCII product string (max 255 characters)
product "Moitessier Navigation HAT"


########################################################################
# GPIO bank settings, set to nonzero to change from the default.
# NOTE these setting can only be set per BANK, uncommenting any of
# these will force the bank to use the custom setting.

# drive strength, 0=default, 1-8=2,4,6,8,10,12,14,16mA, 9-15=reserved
gpio_drive 0

# 0=default, 1=slew rate limiting, 2=no slew limiting, 3=reserved
gpio_slew 0

# 0=default, 1=hysteresis disabled, 2=hysteresis enabled, 3=reserved
gpio_hysteresis 0

# If board back-powers Pi via 5V GPIO header pins:
# 0 = board does not back-power
# 1 = board back-powers and can supply the Pi with a minimum of 1.3A
# 2 = board back-powers and can supply the Pi with a minimum of 2A
# 3 = reserved
# If back_power=2 then USB high current mode will be automatically 
# enabled on the Pi
back_power 0

########################################################################
# GPIO pins, uncomment for GPIOs used on board
# Options for FUNCTION: INPUT, OUTPUT, ALT0-ALT5
# Options for PULL: DEFAULT, UP, DOWN, NONE
# NB GPIO0 and GPIO1 are reserved for ID EEPROM so cannot be set

#         GPIO  FUNCTION  PULL
#         ----  --------  ----
#setgpio  2     INPUT     DEFAULT
#setgpio  3     INPUT     DEFAULT
#setgpio  4     INPUT     DEFAULT
#setgpio  5     INPUT     DEFAULT
#setgpio  6     INPUT     DEFAULT
#setgpio  7     INPUT     DEFAULT
setgpio  8     ALT0     DEFAULT
setgpio  9     ALT0     DEFAULT
setgpio  10    ALT0     DEFAULT
setgpio  11    ALT0     DEFAULT
#setgpio  12    INPUT     DEFAULT
#setgpio  13    INPUT     DEFAULT
#setgpio  14    INPUT     DEFAULT
#setgpio  15    INPUT     DEFAULT
#setgpio  16    INPUT     DEFAULT
setgpio  17    OUTPUT     DEFAULT
setgpio  18    OUTPUT     UP
#setgpio  19    INPUT     DEFAULT
#setgpio  20    INPUT     DEFAULT
#setgpio  21    INPUT     DEFAULT
setgpio  22    OUTPUT     DEFAULT
setgpio  23    INPUT     DEFAULT
setgpio  24    OUTPUT     DEFAULT
#setgpio  25    INPUT     DEFAULT
#setgpio  26    INPUT     DEFAULT
setgpio  27    INPUT     DEFAULT

**** CONTENT EEPROM_SETTINGS.TXT - END ****
```

# Somfy Awning Controller

A small program designed to control Somfy Telis RTS screening devices (blinds, awning etc). The program is written in LUA as a module in the UBUS infrastructure of the OpenWRT operating system. The program controls the Somfy Telis RTS driver using GPIO exported via SYSFS.

The API is adapted for use with homebridge-cmd4 (https://www.npmjs.com/package/homebridge-cmd4).

The program is HW platform-independent. No RPI utils and programs need anymore :-).

## Installation

1. Add repository to OpenWRT *feeds.conf* (`src-git somfy https://github.com/zokl/openwrt-homebridge-sac.git`)
2. Run: `./script/feeds update somfy && ./script/feeds install sac`
2. Choose **sac** in menuconfig *Extra package*
3. Compile

## Configuration

Application **sac** has a configuration file in `/etc/config/sac`. In a configuration file could be specified several parameters. The restart of the **sac** process is necessary after editing the configuration file. 

### Default configuration file
```
config section 'logging'
    option level 'info'

config section 'general'
    option time_to_open '10'

config section 'gpio'
    option active_low '1'
    option up '351'
    option down '361'
    option stop '371'
```
Configuration file items description:
* logging
  * level - logging level (info .. debug)
* general
  * time_to_open - time to device open in seconds
* gpio - gpio address in sysfs (`/sys/class/gpioXY/`)
  * active_low - switch 1/0 output polarity
  * up - GPIO port definition for UP key 
  * down - GPIO port definition for WORN key
  * stop - GPIO port definition for STOP key
  
### How to determine usable GPIO

A lot of Linux HW platforms have a different approach for GPIO identify and control. The mapping between physically GPIO pins and its system interpretation could be done by the reading of sysfs file by `cat /sys/kernel/debug/pinctrl/*/pinmux-pins`.


The actual GPIO configuration for Olimex A20 MICRO follows:
```
root@ZoklHomeBridge:~# cat /sys/kernel/debug/gpio
gpiochip0: GPIOs 0-287, parent: platform/1c20800.pinctrl, 1c20800.pinctrl:
 gpio-35  (                    |sysfs               ) out hi
 gpio-36  (                    |sysfs               ) out hi
 gpio-37  (                    |sysfs               ) out hi
 gpio-40  (                    |ahci-5v             ) out hi
 gpio-41  (                    |usb0-vbus           ) out lo
 gpio-225 (                    |cd                  ) in  lo IRQ ACTIVE LOW
 gpio-226 (                    |a20-olinuxino-micro:) out hi
 gpio-227 (                    |usb2-vbus           ) out hi
 gpio-228 (                    |usb0_id_det         ) in  hi IRQ
 gpio-229 (                    |usb0_vbus_det       ) in  lo IRQ
 gpio-230 (                    |usb1-vbus           ) out hi
 gpio-235 (                    |cd                  ) in  hi IRQ ACTIVE LOW

gpiochip1: GPIOs 413-415, parent: platform/axp20x-gpio, axp20x-gpio, can sleep:
```


## How it Works

The program controls the GPIO outputs of the platform, which are connected to the RF controller Somfy Trelis RTS. Communication with the controller is only one-way. It is not possible to determine the current position of the device. The position between the closed and open states is determined from the set motor run time (time_to_open). Therefore, if the running time of the motor is 10 seconds, the engine is running for 5 seconds when the 50% opening setting is set.

### UBUS Integration

**sac** can be control by UBUS commands:
```
root@HomeBridge:~# ubus -v list somfy
'somfy' @0b015743
	"version":{}
	"set":{"TargetVerticalTiltAngle":"String","TargetPosition":"String","TargetHorizontalTiltAngle":"String"}
	"status":{}
	"control":{"rolling":"String"}
	"get":{"action":"String"}
 ```
 
**Awning UP**
Pulling out the awning.

* `ubus call somfy control '{"rolling":"up"}'`
 
**Awning DOWN**
Unfolding the awning.

* `ubus call somfy control '{"rolling":"down"}'`

**Awning STOP**
Stop moving of awning device.

* `ubus call somfy control '{"rolling":"down"}'`

**Awning to 50%**
* `ubus call somfy set '{"TargetPosition":50}'`

### homebridge-cmd4 integration

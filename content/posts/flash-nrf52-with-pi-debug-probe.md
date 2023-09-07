+++
title = "Programming an nRF52 with the Raspberry Pi Debug Probe"
description = """

"""
type = ["posts","post"]
tags = [
    "raspberry pi",
    "openocd",
    "zephyr",
    "nrf52",
    "flashing",
    "nrf connect sdk"
]
date = "2023-09-07T00:00:00+01:00"
categories = [
    "Development",
    "Hardware",
]
series = ["Hardware Development Course"]
+++
## TL;DR

```
west build -- -DOPENOCD_NRF5_INTERFACE=cmsis-dap
west flash --runner openocd
```
should have you covered, as long as you can run OpenOCD [without sudo]({{< ref "#no-sudo" >}} "Running OpenOCD without sudo").
## Acknowledgements

Thanks to Jeremy Bentham from iosoft for responding to my query.

## Updates

1. I have added information on how to erase the flash memory using the openocd runner (2023-09-07).

## Introduction

Earlier this year I was designing a board which would be powered by the [nRF52833][nrf52833].
Like all other SoC's in the nRF52 series, it can be programmed through a SWD interface.
Since the project had a short timeline, I could not afford to explore other options, and so
I rushed to order an [nRF52833 development kit][nrf52833dk], which costed me £47.54 with VAT
from Digi-Key.
Fortunately, the development kit worked quite well at programming the board I had designed for the
duration of the project. I will not delve into the specific setup I used for this right now.

However, in the middle of development of that specific project, the Raspberry Pi foundation released
the [Debug Probe][rpi-news] for just £10 (or $12 in the US). As soon as I found out it offered SWD
support, I knew I had to give it a shot.
The opportunity would not come until I visited Cambridge, and the official Raspberry Pi store
(a few weeks ago), where I was able to get my hands on one.

## Hardware Setup

**WARNING:**
The Pi Debug Probe is only meant to program boards which run between 1.8v to 3.3v.
Do not connect the debug probe to a board running at a higher voltage, such as 5v,
it may damage the probe.


{{< image src="https://www.raspberrypi.com/documentation/microcontrollers/images/the-probe.png" alt="The debug probe and its accessories" position="center" style="border-radius: 8px;margin:20px 0 20px 0;" >}}
The debug probe comes out of the box with three different cables that serve the same purpose.
You should choose the one which is more convenient for you. My board exposed the programming
interface as male pins so I used the cable with female headers.
Connect the plug end to the *DBUG* port, or the one in the right in this illustration.
Then, connect the other end as outlined in this table.

| Debug Probe | nRF52  |
|-------------|--------|
| SC (Orange) | SWDCLK |
| SD (Yellow) | SWDIO  |
| GND         | GND    |

Make sure your debug probe is connected to the host computer, and that your board has a power source.

## Software Setup

You will need a working installation of the [nRF Connect SDK][ncs],
which can be installed with the [nRF Connect for Desktop][ncd] app.

### OpenOCD
Also known as the [open on-chip debugger][ocd], is a tool created by Dominic Rath,
which 'provides on-chip programming and debugging support', and supports
different interfaces such as telnet, TCL, and GDB.
To work with the Raspberry Pi Debug Probe, or other Raspberry Pi boards,
you should download the fork of openocd maintained by the Raspberry Pi Foundation.
Instructions for that are available [here][ocd-install].
On Linux, the following will do:
```bash
$ sudo apt install automake autoconf build-essential texinfo libtool libftdi-dev libusb-1.0-0-dev
$ git clone https://github.com/raspberrypi/openocd.git --branch rp2040-v0.12.0 --depth=1 --no-single-branch
$ cd openocd
$ ./bootstrap
$ ./configure
$ make -j4
$ sudo make install
```

The main thing to know about openocd is that it operates through commands.
Commands may be passed directly with the `-c "command here"` argument,
or instead, commands can be placed in a file, which are usually `.cfg` files.
These are passed to openocd with the `-f file.cfg` argument.

## Building and Flashing
To illustrate, I will use the *blinky* sample from Zephyr, which should
exist in your nRF Connect SDK installation under `zephyr/samples/basic/blinky`.
This sample only makes the board LED blink (huh).
Inside a terminal with access to the SDK, navigate to the project directory.
If you are using the blinky sample, I suggest working on a copy of it.

Then, issue this command, where *X* is the name of your board.
If you are using an nRF development kit, this might be something like
"nrf52833dk\_nrf52833".

```bash
west build --board X # Replace with the board you will program
```

The build process should create a `zephyr.elf` file under `build/zephyr`,
which will be uploaded to the board with openocd.
The following command will load the configuration file needed to interact with the
debug probe, which uses the CMSIS-DAP protocol, it will also load the configuration
needed to program the nRF52 SoC. Finally, it will issue the command to
upload the program, reset the board, and exit the session. 
The description for these and other commands can be found in the [documentation][ocd-cmd].

```bash
sudo openocd -f interface/cmsis-dap.cfg -f target/nrf52.cfg -c "program build/zephyr/zephyr.elf verify reset exit"
```

The configuration files are included with the RPi installation of openocd, so the command
shoud work as it is. To view the source of the configuration files:
[nRF52.cfg][nRF52.cfg] and [cmsis-dap.cfg][cmsis-dap.cfg].


### Integration with west
Since I am used to programming the board with `west flash` when using the development kit,
I also wanted to do so with the debug probe. This is possible due to the fact that west is highly
configurable, and can use different "runners" to flash a board, one of which is openocd
To investigate this possibility, let's take a look at the board.cmake file for the nRF52833dk, or
the board of your preference.


`zephyr/boards/arm/nrf52833dk_nrf52833/board.cmake`:

```cmake
# SPDX-License-Identifier: Apache-2.0

board_runner_args(jlink "--device=nRF52833_xxAA" "--speed=4000")
board_runner_args(pyocd "--target=nrf52833" "--frequency=4000000")
include(${ZEPHYR_BASE}/boards/common/nrfjprog.board.cmake)
include(${ZEPHYR_BASE}/boards/common/jlink.board.cmake)
include(${ZEPHYR_BASE}/boards/common/pyocd.board.cmake)
include(${ZEPHYR_BASE}/boards/common/openocd-nrf5.board.cmake)
```

Pay attention to the last line in this file, it includes the file below:

`zephyr/boards/common/openocd-nrf5.board.cmake`

```cmake
# SPDX-License-Identifier: Apache-2.0

# Infer nrf51 vs nrf52 etc from the BOARD name. This enforces a board
# naming convention: "nrf5x" must appear somewhere in the board name
# for this to work.
#
# Boards which don't meet this convention can set this variable before
# including this script.
if (NOT DEFINED OPENOCD_NRF5_SUBFAMILY)
  string(REGEX MATCH nrf5. OPENOCD_NRF5_SUBFAMILY "${BOARD}")
endif()
if("${OPENOCD_NRF5_SUBFAMILY}" STREQUAL "")
  message(FATAL_ERROR
    "Can't match nrf5 subfamily from BOARD name. "
    "To fix, set CMake variable OPENOCD_NRF5_SUBFAMILY.")
endif()

if (NOT DEFINED OPENOCD_NRF5_INTERFACE)
  set(OPENOCD_NRF5_INTERFACE "jlink")
endif()

# We can do the right thing for supported subfamilies using a generic
# script, at least for openocd 0.10.0 and the openocd shipped with
# Zephyr SDK 0.10.3.
set(pre_init_cmds
  "set WORKAREASIZE 0x4000"	# 16 kB RAM used for flashing
  "source [find interface/${OPENOCD_NRF5_INTERFACE}.cfg]"
  "transport select swd"
  "source [find target/${OPENOCD_NRF5_SUBFAMILY}.cfg]"
)

foreach(cmd ${pre_init_cmds})
  board_runner_args(openocd --cmd-pre-init "${cmd}")
endforeach()

include(${ZEPHYR_BASE}/boards/common/openocd.board.cmake)

```
You might have noticed that this file configures the arguments that
will be passed to openocd, including an interface and a target:

```cmake
set(pre_init_cmds
  "set WORKAREASIZE 0x4000"	# 16 kB RAM used for flashing
  "source [find interface/${OPENOCD_NRF5_INTERFACE}.cfg]"
  "transport select swd"
  "source [find target/${OPENOCD_NRF5_SUBFAMILY}.cfg]"
)
```

The target is already defined to be the nRF52.
The interface is the only thing left to change, which by default is set to jlink.

```cmake
if (NOT DEFINED OPENOCD_NRF5_INTERFACE)
  set(OPENOCD_NRF5_INTERFACE "jlink")
endif()
```

This can be overriden by setting the `$OPENOCD_NRF5_INTERFACE` cmake variable.
The variable can be set when issuing the `west build` command.
This brings us back to where we started:

```
west build -- -DOPENOCD_NRF5_INTERFACE=cmsis-dap
west flash --runner openocd
```

[See documentation][west-flash].

However, you must make sure that the current user can access cmsis-dap devices
without sudo. I have adapted these instructions from [here][thecore].

### Running OpenOCD without sudo {#no-sudo}

1. Add user to *plugdev*:
```bash
sudo useradd -G plugdev $(whoami)
```

2. Create a custom udev rule to change the file mode of CMSIS-DAP devices.
To do this, create a file at `/etc/udev/rules.d/99-openocd.rules`.
Creating the file will require root privileges.

```
ACTION!="add|change", GOTO="openocd_rules_end"
SUBSYSTEM!="usb|tty|hidraw", GOTO="openocd_rules_end"

# CMSIS-DAP compatible adapters
ATTRS{product}=="*CMSIS-DAP*", MODE="664", GROUP="plugdev"

LABEL="openocd_rules_end"
```

This udev rule will only allow non-root access to cmsis-dap devices.
If you want generalised access to openocd without being root, please read [this][thecore],
but keep in mind it is better to restrict unnecessary access.

### Update: Erasing the flash memory (2023-09-07)

I commonly need to run `west flash --erase` to erase the nRF52 flash memory, which can contain
Bluetooth bonding metadata. This works only with the jlink runner, with the openocd
runner, you will need to run this instead:

```bash
west flash --runner openocd --cmd-pre-load "reset halt; nrf5 mass_erase"
```

This adds a custom openocd command after running `init` but before loading the program
to the chip. First, the board is reset and halted. Then, it runs a specific routine that will
erase the code memory and user configuration registers of the chip [(documentation)][ocd-flash-cmd].


## Closing thoughts

- If you want to use a different Raspberry Pi board, such as a RPi 2, Lean2 covered
that [here][lean2]. Also, it might be worth looking at [this config file][rpi-native]
from the raspberry pi fork of openocd.

- If you want to program an nRF51 device instead, I would try to use the `target/nrf51.cfg`
target, but alas I have no way of testing this.


## Related material

- Raspberry Pi Debug Probe Documentation: [link][probe-docs]
- Programming an nRF52 using a Raspberry Pi 2: [link][lean2]
- Using OpenOCD in the Raspberry Pi: [link][lean2-2]
- OpenOCD official website: [link][ocd]



<!-- LINKS -->
[rpi-news]: https://www.raspberrypi.com/news/raspberry-pi-debug-probe-a-plug-and-play-debug-kit-for-12/
[segger]: https://shop.segger.com/debug-trace-probes/?p=1
[nrf52833]: https://www.nordicsemi.com/products/nrf52833 
[nrf52833dk]: https://www.nordicsemi.com/Products/Development-hardware/nRF52833-DK
[lean2]: https://iosoft.blog/2019/09/28/arm-gcc-lean-nordic-nrf52/
[lean2-2]: https://iosoft.blog/2019/01/28/raspberry-pi-openocd/
[ncs]: https://www.nordicsemi.com/Products/Development-software/nrf-connect-sdk
[ncd]: https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-Desktop/Download
[ocd]: https://openocd.org/
[ocd-install]: https://www.raspberrypi.com/documentation/microcontrollers/debug-probe.html#installing-openocd
[probe-docs]: https://www.raspberrypi.com/documentation/microcontrollers/debug-probe.html
[ocd-cmd]: https://openocd.org/doc/html/General-Commands.html
[cmsis-dap.cfg]: https://github.com/raspberrypi/openocd/blob/rp2040-v0.12.0/tcl/interface/cmsis-dap.cfg
[nRF52.cfg]: https://github.com/raspberrypi/openocd/blob/rp2040-v0.12.0/tcl/target/nrf52.cfg
[west-flash]: https://docs.zephyrproject.org/latest/develop/west/build-flash-debug.html
[thecore]: https://forgge.github.io/theCore/guides/running-openocd-without-sudo.html
[rpi-native]: https://github.com/raspberrypi/openocd/blob/rp2040-v0.12.0/tcl/interface/raspberrypi-native.cfg
[ocd-flash-cmd]: https://openocd.org/doc/html/Flash-Commands.html

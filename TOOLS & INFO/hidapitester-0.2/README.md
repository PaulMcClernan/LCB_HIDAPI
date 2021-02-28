# hidapitester

Simple command-line program to exercise HIDAPI

[![Build Status linux](https://api.cirrus-ci.com/github/todbot/hidapitester.svg?task=linux)](https://cirrus-ci.com/github/todbot/hidapitester)
[![Build Status macosx](https://api.cirrus-ci.com/github/todbot/hidapitester.svg?task=macosx)](https://cirrus-ci.com/github/todbot/hidapitester)
[![Build Status windows](https://api.cirrus-ci.com/github/todbot/hidapitester.svg?task=windows)](https://cirrus-ci.com/github/todbot/hidapitester)


The point of `hidapitester` is to provide a simple low-dependency
command-line tool to test out every API call in
[hidapi](https://github.com/libusb/hidapi).

<img src="./docs/screencast1a.gif" width="500">



## Prebuilt binaries

See the [hidapitester releases page](https://github.com/todbot/hidapitester/releases)
for builds for:

- Mac OS X
- Linux (Ubuntu x64)
- Windows

## Usage

`hidapitester` works by parsing a list of arguments as commands it executes in order.
Those commands are:

```
  --vidpid <vid/pid>          Filter by vendorId/productId (comma/slash delim)
  --usagePage <number>        Filter by usagePage
  --usage <number>            Filter by usage
  --list                      List HID devices (by filters)
  --list-detail               List HID devices w/ details (by filters)
  --open                      Open device with previously selected filters
  --open-path <pathstr>       Open device by path (as in --list-detail)
  --close                     Close currently open device
  --send-feature <datalist>   Send Feature report (1st byte reportId, if used)
  --read-feature <reportId>   Read Feature report (w/ reportId, 0 if unused)
  --send-output <datalist>    Send Ouput report to device
  --read-input [reportId]     Read Input report (w/ opt. reportId, if unused)
  --read-input-forever [rId]  Read Input reports in a loop forever
  --length <len>, -l <len>    Set buffer length in bytes of report to send/read
  --timeout <msecs>           Timeout in millisecs to wait for input reads
  --base <base>, -b <base>    Set decimal or hex buffer print mode
  --quiet, -q                 Print out nothing except when reading data
  --verbose, -v               Print out extra information
```

### Listing and Opening Devices
The `--list-detail` shows all available information,
including usagePage, usage, and path.
Specifying `--vidpid`, `--usagePage`, or `--usage` will filter the output
of `--list` and `--list-detail`.

You must `--open` before you can `--read-input`. You can also `--read-input`
multiple times, or `--open` one device, `--close` it, and `--open` another.

The `--vidpid` commmands allows full or partial specification of the
Vendor Id and Product Id.  These are all valid:

```
--vidpid 16C0:FFAB  # specify both vid 0x16C0 and pid 0xFFAB
--vidpid 16C0       # just specify the vid
--vidpid 0:FFAB     # just specify the pid
--vidpid 16C0:FFAB  # use colon instead of slash
```

The `--open` command will take whichever of VID, PID, usagePage, and usage are
specified.  So these are valid:

```
hidapitester --vidpid 16C0 --usagePage FFAB --open      # specify vid and usagePage
hidapitester --usage FFAB --open                        # specify only usagePage
hidapitester --0/0486  --open                           # specify only pid
hidapitester --vidpid 16C0/486 --usagePage FFAB --open  # specify vid,pid,usagePage
```

### Reading and Writing Reports

Send Output reports to devices with `--send-output`. The argument to the command
is the data to send: `--send-output 1,2,0xff,30,40,0x50`.
If using reportIds, the first byte is the reportId.
If not using reportIds, the first byte should be `0`.
The length of the actual report is set by `--length <num>`.

Thus to send a 16-byte report on reportId 3 with only the 1st byte set to "42":
```
hidapitester [...] --length 16 --send-output 3,42
```

Send Feature reports the same way with `--send-feature`.

Read Input reports from device with `--read-input`.  If using reportIds,
the argument should be the reportId number: `--read-input 1`.  The length to read is
specified by the `--length` argument.  If using reportIds, this length should be one
more than the buffer to read (e.g. if the report is 16-bytes, length is 17).

So to read a 16-byte report on reportId 3:
```
hidapitester [...] --length 17 --read-input 3
```



## Examples

Get version info from a blink(1):

```
hidapitester --vidpid 0x27b8/0x1ed --open --length 9 --send-feature 1,99,0,255,0  --read-feature 1 --close
Opening device at vid/pid 27b8/1ed
Set buflen to 9
Writing 9-byte feature report...wrote 9 bytes
Reading 9-byte feature report, report_id 1...read 8 bytes
Report:
0x0, 0x63, 0x0, 0xff, 0x0, 0x0, 0x0, 0x0, 0x0,
Closing device
```

Send data to/from "TeensyRawHid" sketch:
```
hidapitester --vidpid 16C0 --usagePage 0xFFAB --open --send-output 0x4f,33,22,0xff  --read-input
Opening device, vid/pid:0x16C0/0x0000, usagePage/usage: FFAB/0
Device opened
Writing output report of 64-bytes...wrote 64 bytes:
 4F 21 16 FF 00 00 00 00 00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
Reading 64-byte input report, 250 msec timeout...read 64 bytes:
 AB CD 01 67 01 6F 01 93 01 94 01 A6 01 AA 01 67
 01 82 01 7D 01 79 01 18 01 0B 00 00 00 00 00 00
 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00 00 00 00 00 00 00 09 91
Closing device
```




### Testing

- The "TeensyRawHid" directory contains an Arduino sketch for Teensy.
The sketch sends 64-byte Input reports every second, with no reportId.
The sketch receives 64-byte Output reports, and prints them
to Serial Monitor.

- The "ProMicroRawHID" directory contains an Arduino sketch for any board
supported by NicoHood's [HID Project](https://github.com/NicoHood/HID)
This sketch sends a 64-byte Input report every 2 seconds, with no reportId.
The sketch recives 64-byte Output or Feature reports, and prints them
to Serial Monitor



## To build
Building `hidapitester` is done via a very simple Makefile.

```
git clone https://github.com/libusb/hidapi
git clone https://github.com/todbot/hidapitester
cd hidapitester
make
```

`hidapitester` will use a copy of `hidapi` located next to it in the directory hierarchy.
If you install `hidapi` in a different directory, you can set the Makefile
variable `HIDAPI_DIR` before invoking `make`:

```
# hidapi is in dir 'hidapi-libusb-test'
cd hidapitester
HIDAPI_DIR=../hidapi-libusb-test make clean
HIDAPI_DIR=../hidapi-libusb-test make
./hidapitester --list
```


### Platform-specific requirements

On Mac:
- Install XCode

On Windows
- Install MSYS
- Build in a MinGW window

On Linux
- Install udev, pkg-config, and python
```
sudo apt install libudev1 libudev-dev pkg-config python
```

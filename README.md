
# Overview

The **i2cblock** is a Linux utility to test the large IO chunks' capabilities
and benchmark performance of the I2C controllers and Linux kernel I2C bus
drivers.

The i2cblock was originally written to save me the hassle of utilizing the
Linux i2c tools like i2ctransfer, i2cdump, i2cset, and i2cget to debug and test
the hid-ft260.ko Linux kernel driver for the FTDI FT260 chip, which implements
the USB HID to I2C bus bridge. And specifically, to issue the data transfers
exceeding a single USB HID report size or maximum size of the EEPROM Page Write
operation. The maximum transfer size in the Page Write mode differs between
EEPROM chips. For example, it's 8 bytes for the 24C02 and 128 bytes for the
24C512. The size of the internal address, also known as offset, is different
for small and large EEPROMs. It can be either one or two bytes depending on
the EEPROM size.

The eeprog utility of the i2c-toots package provides a very limited EEPROM
read-write functionality not sufficient to exercise any I2C bus controller or
I2C bus driver. Unlike the I2C standard Linux tools like i2cget, i2cset,
i2cdump, and i2ctransfer, it is not a part of the Linux distributions and
requires downloading, compilation, and i2c-dev driver loading. Then, it can
perform a single byte I2C IO transactions only. It adds a significant overhead
slowing down the maximum possible IO rate by about an order of magnitude. It
neither generates test patterns nor measures performance and is helpless for
the I2C bus driver evaluation and benchmarking.

This bash script leverages the standard Linux I2C tools simplifying their
configuration to issue predefined I2C IO patterns allowing performance
benchmarking of the I2C controller and Linux kernel I2C bus driver by
calculating the transmitted data rate at different workloads.

```
 Usage: i2cblock [-h] [-v] -S] [-f MODE] [-d MODE] [-o SIZE] [-s SIZE] [-r FIRST-LAST] BUS ADDRESS

         BUS                    I2C bus master device number
         ADDRESS                EEPROM device address (0x50 - 0x57)

         -h, --help             Show this help message and exit
         -v, --verbose          More verbose output
         -S, --stat             Print statistics

         -d, --dumpmode MODE    Read data mode:
                                1 - Read block via i2cdump byte by byte
                                2 - Read block via i2ctransfer by chunks

         -f, --fillmode MODE    Write data mode:
                                1 - Fill block with zeros via i2ctransfer
                                2 - Fill block with increment via i2ctransfer by chunks
                                3 - Fill block with increment via i2cset byte by byte
                                4 - Fill block with pseudo random via i2ctransfer by chunks

         -r, --range FIRST-LAST Address range to operate
         -s, --xfersize SIZE    Data chunk size in bytes

         -o, --offsize SIZE     Offset (addr in chip addr space) size:
                                1 - One byte offset like in 24C01 - 24C16 EEPROMs
                                2 - Two bytes offest, for larger than 256B EEPROMs
```

Tested with Ubuntu 16.04 on PC and Ubuntu 20.04.3 in VirtualBox with:
- 24c02, 24c32, 24c512 with UMFT260EV1A EVB [1] and hid-ft260 kernel driver [2]
- 24c32, 24c512  with CH341A black board [3] and 2c-ch341-usb kernel driver [4]

# Examples

## Fill 1KB block with zeros via i2ctransfer by 32 bytes chunks
Setup: 24c512 EEPROM, UMFT260EV1A EVB, hid-ft260 driver, PC (i7-4790K CPU)
```
$ ./i2cblock -f 1 -o 2 -s 32 -r 0-0x3ff 13 0x51 -S

Performance stats:
  Fill block with zeros via i2ctransfer
  baudrate   datarate(bps)  efficiency(%)  datasize(B)  totalIOs    IOsize(B)
  ---------------------------------------------------------------------------
  31796      26315          82             1024         32          32
```

## Write 1KB data block with pseudo random via i2ctransfer by 16 bytes chunks
Setup: 24c512 EEPROM, CH341A black board, i2c-ch341-usb driver, PC (i7-4790K CPU)
```
$ ./i2cblock -f 4 -o 2 -s 16 -r 0-0x3ff 13 0x50 -S

Performance stats:
  Fill block with pseudo random via i2ctransfer by chunks
  baudrate   datarate(bps)  efficiency(%)  datasize(B)  totalIOs    IOsize(B)
  ---------------------------------------------------------------------------
  32894      24390          74             1024         64          16
```

## Write 1KB data block with increment via i2cset byte by byte
Setup: 24c512 EEPROM, CH341A black board, i2c-ch341-usb driver, PC (i7-4790K CPU)
```
$ ./i2cblock -f 3 -o 2 -r 0-0xff 13 0x50 -S

Performance stats:
  Fill block with increment via i2cset byte by byte
  baudrate   datarate(bps)  efficiency(%)  datasize(B)  totalIOs    IOsize(B)
  ---------------------------------------------------------------------------
  6899       1453           21             1024         1024        1
```

## Read 256B block via i2ctransfer by chunks of 16 bytes size
Setup: 24c512 EEPROM, CH341A black board, i2c-ch341-usb driver, PC (i7-4790K CPU)
```
$ ./i2cblock -d 2 -o 2 -s 16 -r 0-0xff 13 0x50 -S
0x0000: 0x00 0x01 0x02 0x03 0x04 0x05 0x06 0x07 0x08 0x09 0x0a 0x0b 0x0c 0x0d 0x0e 0x0f
0x0010: 0x10 0x11 0x12 0x13 0x14 0x15 0x16 0x17 0x18 0x19 0x1a 0x1b 0x1c 0x1d 0x1e 0x1f
0x0020: 0x20 0x21 0x22 0x23 0x24 0x25 0x26 0x27 0x28 0x29 0x2a 0x2b 0x2c 0x2d 0x2e 0x2f
0x0030: 0x30 0x31 0x32 0x33 0x34 0x35 0x36 0x37 0x38 0x39 0x3a 0x3b 0x3c 0x3d 0x3e 0x3f
0x0040: 0x40 0x41 0x42 0x43 0x44 0x45 0x46 0x47 0x48 0x49 0x4a 0x4b 0x4c 0x4d 0x4e 0x4f
0x0050: 0x50 0x51 0x52 0x53 0x54 0x55 0x56 0x57 0x58 0x59 0x5a 0x5b 0x5c 0x5d 0x5e 0x5f
0x0060: 0x60 0x61 0x62 0x63 0x64 0x65 0x66 0x67 0x68 0x69 0x6a 0x6b 0x6c 0x6d 0x6e 0x6f
0x0070: 0x70 0x71 0x72 0x73 0x74 0x75 0x76 0x77 0x78 0x79 0x7a 0x7b 0x7c 0x7d 0x7e 0x7f
0x0080: 0x80 0x81 0x82 0x83 0x84 0x85 0x86 0x87 0x88 0x89 0x8a 0x8b 0x8c 0x8d 0x8e 0x8f
0x0090: 0x90 0x91 0x92 0x93 0x94 0x95 0x96 0x97 0x98 0x99 0x9a 0x9b 0x9c 0x9d 0x9e 0x9f
0x00a0: 0xa0 0xa1 0xa2 0xa3 0xa4 0xa5 0xa6 0xa7 0xa8 0xa9 0xaa 0xab 0xac 0xad 0xae 0xaf
0x00b0: 0xb0 0xb1 0xb2 0xb3 0xb4 0xb5 0xb6 0xb7 0xb8 0xb9 0xba 0xbb 0xbc 0xbd 0xbe 0xbf
0x00c0: 0xc0 0xc1 0xc2 0xc3 0xc4 0xc5 0xc6 0xc7 0xc8 0xc9 0xca 0xcb 0xcc 0xcd 0xce 0xcf
0x00d0: 0xd0 0xd1 0xd2 0xd3 0xd4 0xd5 0xd6 0xd7 0xd8 0xd9 0xda 0xdb 0xdc 0xdd 0xde 0xdf
0x00e0: 0xe0 0xe1 0xe2 0xe3 0xe4 0xe5 0xe6 0xe7 0xe8 0xe9 0xea 0xeb 0xec 0xed 0xee 0xef
0x00f0: 0xf0 0xf1 0xf2 0xf3 0xf4 0xf5 0xf6 0xf7 0xf8 0xf9 0xfa 0xfb 0xfc 0xfd 0xfe 0xff

Performance stats:
  Read block via i2ctransfer by chunks
  baudrate   datarate(bps)  efficiency(%)  datasize(B)  totalIOs    IOsize(B)
  ---------------------------------------------------------------------------
  27487      19230          69             256          16          16
```

## Read 256B block via i2ctransfer by 32 bytes chunks
Setup: 24c512 EEPROM, UMFT260EV1A EVB, hid-ft260 driver, PC (i7-4790K CPU)
```
swuser@comex-usb01:~/sw/i2cblock$ ./i2cblock -d 2 -o 2 -s 32 -r 0-0xff 13 0x51 -S
0x0000: 0x00 0x01 0x02 0x03 0x04 0x05 0x06 0x07 0x08 0x09 0x0a 0x0b 0x0c 0x0d 0x0e 0x0f 0x10 0x11 0x12 0x13 0x14 0x15 0x16 0x17 0x18 0x19 0x1a 0x1b 0x1c 0x1d 0x1e 0x1f
0x0020: 0x20 0x21 0x22 0x23 0x24 0x25 0x26 0x27 0x28 0x29 0x2a 0x2b 0x2c 0x2d 0x2e 0x2f 0x30 0x31 0x32 0x33 0x34 0x35 0x36 0x37 0x38 0x39 0x3a 0x3b 0x3c 0x3d 0x3e 0x3f
0x0040: 0x40 0x41 0x42 0x43 0x44 0x45 0x46 0x47 0x48 0x49 0x4a 0x4b 0x4c 0x4d 0x4e 0x4f 0x50 0x51 0x52 0x53 0x54 0x55 0x56 0x57 0x58 0x59 0x5a 0x5b 0x5c 0x5d 0x5e 0x5f
0x0060: 0x60 0x61 0x62 0x63 0x64 0x65 0x66 0x67 0x68 0x69 0x6a 0x6b 0x6c 0x6d 0x6e 0x6f 0x70 0x71 0x72 0x73 0x74 0x75 0x76 0x77 0x78 0x79 0x7a 0x7b 0x7c 0x7d 0x7e 0x7f
0x0080: 0x80 0x81 0x82 0x83 0x84 0x85 0x86 0x87 0x88 0x89 0x8a 0x8b 0x8c 0x8d 0x8e 0x8f 0x90 0x91 0x92 0x93 0x94 0x95 0x96 0x97 0x98 0x99 0x9a 0x9b 0x9c 0x9d 0x9e 0x9f
0x00a0: 0xa0 0xa1 0xa2 0xa3 0xa4 0xa5 0xa6 0xa7 0xa8 0xa9 0xaa 0xab 0xac 0xad 0xae 0xaf 0xb0 0xb1 0xb2 0xb3 0xb4 0xb5 0xb6 0xb7 0xb8 0xb9 0xba 0xbb 0xbc 0xbd 0xbe 0xbf
0x00c0: 0xc0 0xc1 0xc2 0xc3 0xc4 0xc5 0xc6 0xc7 0xc8 0xc9 0xca 0xcb 0xcc 0xcd 0xce 0xcf 0xd0 0xd1 0xd2 0xd3 0xd4 0xd5 0xd6 0xd7 0xd8 0xd9 0xda 0xdb 0xdc 0xdd 0xde 0xdf
0x00e0: 0xe0 0xe1 0xe2 0xe3 0xe4 0xe5 0xe6 0xe7 0xe8 0xe9 0xea 0xeb 0xec 0xed 0xee 0xef 0xf0 0xf1 0xf2 0xf3 0xf4 0xf5 0xf6 0xf7 0xf8 0xf9 0xfa 0xfb 0xfc 0xfd 0xfe 0xff

Performance stats:
  Read block via i2ctransfer by chunks
  baudrate   datarate(bps)  efficiency(%)  datasize(B)  totalIOs    IOsize(B)
  ---------------------------------------------------------------------------
  24715      19607          79             256          8           32
```

# Source
https://github.com/MichaelZaidman/i2cblock.git


# References
[1] https://www.ftdichip.com/Support/Documents/DataSheets/Modules/DS_UMFT260EV1A.pdf

[2] https://github.com/MichaelZaidman/hid-ft260

[3] https://www.onetransistor.eu/2017/08/ch341a-mini-programmer-schematic.html

[4] https://github.com/allanbian1017/i2c-ch341-usb


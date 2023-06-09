17/02/2012

# Bootstrapping SRAM from FLASH on the Papilio Plus

## Introduction

As flexible as an FPGA can be, it is sometimes limited by the amount of resources it has on chip. Some projects for example, require access to large amounts of memory, be it RAM or ROM or both. It is for that reason that it is handy to have external memory attached to the FPGA. The Papilio Plus is based on a Xilinx Spartan 6 family FPGA, specifically the LX9, which has a total of 589824 bits of BRAM internally, which translates to 72K bytes of internal BRAM, however the Papilio Plus has an external 256Kbx16 fast SRAM attached to the FPGA in the form of an IS61WV25616BLL chip which significantly increases the amout of RAM the FPGA has access to. While it is true that the Papilio also has 4M bits of FLASH attached in the form of the SST25VF040B, that memory is significantly slower to access because its contents must be accessed serially and clocked out one bit at a time.

It would be nice to have a fast, parallel access ROM attached, but how does one turn a SRAM which loses its contents at power off into a permanent non volatile ROM? This project addresses that issue!

The basic priciple is to store the ROM contents into the comparatively slow serial flash then copy them at power on into the SRAM before handing over control to the user. We need to bootstrap the SRAM, so there are two issues that need to be addressed, how to upload arbitrary user data into the flash and how to copy that data from serial flash to parallel SRAM.

## Storing arbitrary data in flash

There are tools like the Papilio Loader that uploads FPGA bitstream (.bit files) into the flash and other tools like Xilinx's data2mem which can insert user provided data into the BRAM memory locations inside a .bit file, but none of these actually help us. Our user ROM data is not stored inside the FPGA BRAM and as such, it is not part of the .bit file. If the FPGA had enough BRAM to hold our ROM, we would not need the external storage after all.

This is where bitmerge.py comes in. This is a Python script which will take a valid .bit file and a user provided binary file and merge them together into another .bit file which can be uploaded to the flash using the Papilio Loader. This will all make sense once we understand what a .bit file really is.

## FPGA bitstream files

Once a user design is error free and synthesized, the FPGA compiler can produce a .bit file, which is really a binary file that contains a small header followed by the actual FPGA binary bitstream. The Papilio Loader does not in fact write the entire .bit file into the flash, it only writes the FPGA bitstream contained within the .bit file. The format of the .bit file is [explaned here](https://www.fpga-faq.com/FAQ_Pages/0026_Tell_me_about_bit_files.htm) but basically it consists of a magic number and 5 sections labeled a, b, c, d, e. The first four sections contain strings such as the design name, the device type, date and time the .bit file was compiled. Section e contains the actual bitstream. What the bitmerge.py script does is append the user binary ROM data to the FPGA bitstream data and rewrite the section e length to be the total combined length of both the FPGA bitstream and user ROM data. The Papilio Loader will happily parse this new .bit file and write section e to the flash which includes both the FPGA bitstream and the appended user ROM data.

But won't the FPGA get confused now because new data is appended to its bitstream? In actual fact, no. On power on, the LX9 FPGA will serially shift in from the flash only the FPGA bitstream and then stop. Provided these bits are a valid FPGA bitstream, which they are, because we haven't modified the FPGA bitstream in the .bit file, the FPGA will happily run the design.

As a form or sanity checking, the bitmerge.py script will parse the original .bit file and print out the contents of sections a through d, to ensure the input file is a valid .bit file. It will also do some basic checking to ensure the files do not exceed the flash capacity, then it will print out the address in flash where the user ROM data begins.

The example below shows a merge operation and expected output:

```
 >bitmerge.py bootstrap_top.bit rom.bin output.bit  
 Section 'a' (size     36) 'bootstrap_top.ncd;UserID=0xFFFFFFFF '
 Section 'b' (size     12) '6slx9tqg144 '
 Section 'c' (size     11) '2012/02/17 '
 Section 'd' (size      9) '00:40:54 '
 Section 'e' (size 340604) 'FPGA bitstream'
 Merged user data begins at FLASH address 0x05327C
 ```

## Copying data from flash to SRAM

Now that we've stored the ROM contents in non volatile flash, we need to copy them to SRAM on every power on so that they are available for fast access. An example project is attached which shows how this can be accomplished. At power on or external reset, the signal "bootstrap_busy" is asserted and a state machine initializes the flash by sending it the address where the user data begins. This is the address we obtained from bitmerge.py and is hardcoded in the VHDL top level code as constant "user_address". Because we maintain the chip select to the flash, we do not need to send further addresses to the flash, we simply keep clocking it and every 8 clocks we've shifted a new byte out of the flash. The flash auto-increments the address for us, and if the maximum address is reached, it rolls over to zero though that won't happen here as we stop reading the flash when we've reached the end. As each byte is read from flash, the state machine stores in the low byte of the SRAM's 16 bit data bus, whereas the high byte is set to zero since it's not used. When the flash has finished copying to SRAM, the signal "bootstrap_busy" is de-asserted. This signal can be directly used as an active high reset by the user design. Finally, the signal "bootstrap_busy" drives a multiplexer which "steals" the SRAM data, address and control lines during bootstrap and connectes them to "bs_*" signals then releases them once it's done. The user can access the SRAM via the signals "user_*"

During bootstrap, the flash data range from address "user_address" to the top of flash (0x3FFFF) is copied to the SRAM starting at address zero, even if the user ROM data does not in fact occupy that entire range. The performance penalty for copying the extra data is negligible.

In the example project, the user portion of the design reads the SRAM sequentially and sends the bytes out as rs232 data at 115200 baud 8N1, formatted as 16 hex bytes per line. This is just an example and the user can replace this part of the design with whatever they wish.

## Limitations
The FPGA bit stream for an LX9 is about 333K bytes and the serial flash is 4M bits, or 512K bytes so the amount of space available for user ROMs is about 179K bytes.
The SRAM is significantly bigger than 179K bytes so there will not be enough space in the flash to fill the entire SRAM. Also the SRAM data bus is 16 bits or 2 bytes wide.
It's conceivable that a user could use the low byte lane of the SRAM, for example, to store the ROM image while using the high byte lane as actual RAM. The limitation here is that one cannot access different "ROM" and RAM addresses simultaneously, so a staggered access must be implemented.

Also, because the SRAM LB and UB selectors are tied together, byte access to the SRAM is disabled, so the SRAM data can only be accessed 16 bits wide at a time. For that reason, if one wants to write to the SRAM, care must be taken to not overwrite the "ROM" contents. This is accomplished with a read-modify-write cycle. The SRAM contents at a given address are read to obtain both high (RAM) and low (ROM) bytes, the high byte is updated while preserving the low byte, then the word (2 bytes) is written to SRAM again.

Alex

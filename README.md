# pdd.sh

A [TPDD client](http://tandy.wiki/TPDD_client) implemented in pure\* bash.

It's pure bash except for the following:  
* "stty" is needed once at startup to configure the serial port.  
* "mkfifo" is used once at startup for _sleep() without /usr/bin/sleep .  

That's it. There are no other external commands or dependencies, not even any child forks (no backticks or pipes).

## Usage
```pdd [tty_device] [command [args...]]```

With no arguments, it will run in interactive command mode.  
You get a ```TPDD($mode)>``` prompt where you can enter commands.  
"help" is still not one of them ;) Sorry.

**tty_device** will be auto detected in most cases.  
Failing that, you'll be shown a list of possible choices to select from.  
Or you may specify one as the first argument on the command line.  

There are two groups of commands, "operation mode" and "FDC mode".  

**"operation mode" commands**  
| command | arguments | description |
| --- | --- | --- |
| status | | Report the drive/disk status |
| ls&#160;\|&#160;list&#160;\|&#160;dir | | Directory listing |
| rm&#160;\|&#160;del | disk_filename | Delete a file |
| cp&#160;\|&#160;copy | disk_src_filename&#160;disk_dest_filename | Copy a file on-disk to another file on-disk |
| mv&#160;\|&#160;ren | disk_src_filename&#160;disk_dest_filename | Rename a file on-disk |
| load | disk_src_filename&#160;\[local_dest_filename\] | Copy a file from the disk |
| save | local_src_filename&#160;\[disk_dest_filename\] | Copy a file to the disk |
| format | | Format the disk - 64-byte sector size |
| fdc | | Switch to FDC mode |

**"FDC mode" commands**
| command | arguments | Description |
| --- | --- | -- |
| M&#160;\|&#160;mode | 0\|1 | Select operation(0) or fdc(1) mode |
| D&#160;\|&#160;condition | | Report the drive/disk status |
| F&#160;\|&#160;ff&#160;\|&#160;fdc_format | \[0-6\] | Format disk, sector size 64 80 128 256 512 1024 1280. (default 1280 if not specified) |
| R&#160;\|&#160;rl&#160;\|&#160;read_logical | \[0-79\]&#160;\[1-20\]&#160;\[local_filename\] | Read one logical sector at address: physical(0-79) logical(1-20). Save to local_filename if given, else display on screen.<br>default physical 0 logical 1 |
| A&#160;\|&#160;ri&#160;\|&#160;read_id | \[0-79\]&#160;\[local_filename\] | Read Sector ID Data [(may not be correct yet)](confusing_observations.md#sector-id-section)<br>default physical sector 0 |
| S&#160;\|&#160;si&#160;\|&#160;search_id | | not yet implemented |
| B&#160;\|&#160;wi&#160;\|&#160;write_id | | not yet implemented |
| W&#160;\|&#160;ws&#160;\|&#160;write_sector | | not yet implemented |
| rp&#160;\|&#160;read_physical | \[0-79\] \[h\|b:filename\] | Read all logical sectors in a physical sector<br>default physical sector 0<br>default display on screen<br>**h:filename** writes a hex dump to **filename**<br>**b:filename** writes binary to **filename** |
| dd&#160;\|&#160;dump_disk | \[h\|b:filename\] | Read all logical sectors in all physical sectors<br>default display on screen<br>**h:filename** writes a hex dump to **filename**<br>**b:filename** writes binary to **filename** |

**general/other commands**  
| command | arguments | Description |
| --- | --- | -- |
| q&#160;\|&#160;quit \| bye \| exit | | Order Pizza |
| debug | \[0-3\] | Debug/verbose level - Set the specified debug level, or toggle between 0 & 1 of no level given |

There are also a bunch of low level raw/debugging commands not shown here. See do_cmd() in the script.

Multiple commands may be given at once, seperated by ';' to form a pre-loaded sequence.  

Additionally some behavior may be modified by setting environment variables.
| variable | value | effect |
| --- | --- | --- |
| DEBUG | # | same as debug command above |
| FLOPPY_COMPAT | true\|false | (default is true) automatically pad & un-pad filenames between the natural form and the space-padded 6.2 form needed to be compatible with "Floppy" & "Flopy2". Disabling allows you to see the actual on-disk file names like <pre>"A     .BA               "</pre> and allows you to use the entire 24-byte filename field however you want |

You generally don't need to explicitly use the "fdc" or "mode" commands to switch to operation-mode or fdc-mode. All mode-specific commands switch the mode as necessary on the fly.

No built-in help yet.

## Examples
The same commands can be given either on the command line, or at the interactive prompt.  
Example, to list the directory, where the command is: ```ls```, can be used either of these ways:  
```$ ./pdd ls``` or ```TPDD(opr)> ls```

**Copy a file from the disk**  
```$ ./pdd load DOSNEC.CO```

**Copy a file from the disk and save to a different local name**  
```$ ./pdd load DOSNEC.CO ts-dos_4.1_nec.co```

**Copy a file to the disk**  
```$ ./pdd save ts-dos_4.1_nec.co DOSNEC.CO```

**Stacked Commands: Delete File, then Directory List**  
In interactive mode:  
```TPDD(opr)> rm DOSNEC.CO ;ls```  
In non-interactive mode, quote the list because of the ";"  
```$ ./pdd "rm DOSNEC.CO ;ls"```  
...or you could use backslash to escape it:  
```$ ./pdd rm DOSNEC.CO \;ls```

**FDC-mode drive condition**  
Interactive:  
```
$ ./pdd
TPDD(opr)> fdc
TPDD(fdc)> condition
Disk Inserted, Writable
TPDD(fdc)>
```
Non-interactive:  
```
$ ./pdd D
Disk Inserted, Writable
$ 
```

**Verbose/debug mode**  
```$ DEBUG=1 ./pdd ...``` or ```TPDD(opr)> debug 1```

Log raw serial port traffic:  
Make every call to tpdd_read() or tpdd_write() also create a local file with a copy of whatever was actually read from or written to the serial port.  
```$ DEBUG=3 ./pdd ...``` or ```TPDD(opr)> debug 3```

**Find out a disk's logical sector size**  
Most disks are formatted with 20 64-byte logical sectors per physical sector, since that's what the operation-mode format function in the firmware does, but there are exceptions. The TPDD1 Utility Disk seems like a normal disk, but it's actually formatted with 1 1280-byte logical sector per physical sector. You need to know this to use some FDC-Mode commands.  
The logical sector size that a disk is formatted with can be seen by running the read_physical, read_logical, or read_id commands on any sector.  
The quickest is to run either ```ri``` or ```rl``` with no arguments:  
```$ ./pdd ri``` or ```$ ./pdd rl```

**Read the Sector ID Data for all 80 physical sectors**  
(using bash shell expansion to brute-force something the program doesn't provide itself)  
```$ ./pdd ri\ {0..79}\;```

**Hex dump a physical sector to file**
```$ ./pdd rp 3 h:mydisk_p3.hex```

**Binary dump a physical sector to file**
```$ ./pdd rp 0 b:mydisk_p0.bin```

**Dump entire disk**  
```$ ./pdd dd```

**Hex dump entire disk to file mydisk.hex**  
```$ ./pdd dd h:mydisk.hex```

**Binary dump entire disk to file mydisk.bin**  
```$ ./pdd dd b:mydisk.bin```

# Status
All the "operation mode" commands work.  
Half way through the "FDC mode" commands  

# References
http://tandy.wiki/TPDD  
https://archive.org/details/TandyPortableDiskDriveSoftwareManual26-3808s/  
https://www.ordersomewherechaos.com/rosso/fetish/m102/web100/docs/pdd2-sector-0.html  
https://www.ordersomewherechaos.com/rosso/fetish/m102/web100/docs/pdd-sector-access.html

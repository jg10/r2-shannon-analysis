# Radare2 Analysis of Shannon Baseband

## Initial Steps

* Load the extracted firmware in radare2:

```bash
$ r2 -m 0x40010000 -e asm.arch=arm -b 16 -e asm.cpu=cortex modem-2780.bin
```

where:

*  `-m 0x40010000` specifies the loading address of the firmware image
*  `-e asm.arch=arm` specifies it's an ARM processor type
*  `-b 16` specifies the number of bits of the processor (to accommodate Thumb instructions)
*  `-e asm.cpu=cortex` specifies that the processor is a Cortex ARM processor 
*  `modem-2780.bin` specifies the baseband firmware image after extraction

which results in...

```bash
WARNING: using oba to load the syminfo from different mapaddress.
TODO: Must use the API instead of running commands to speedup loading times.
WARNING: bin_strings buffer is too big (0x024e60bc). Use -zzz or set bin.maxstrbuf (RABIN2_MAXSTRBUF) in r2 (rabin2)
 -- Seek at relative offsets with 's +<offset>' or 's -<offset>'
[0x40010000]>
```

* In theory you could try `aaa` to perform automatic analysis and function identification but in my experience I ran out of RAM and it took too long on this firmware image... this normally works ok though although the creator of radare2 suggests there are better ways ;-) (http://radare.today/posts/analysis-by-default/)

* You can print the first 10 lines of disassembly like so:

```
[0x40010000]> pd 10
    ⁝⁝⁝⁝⁝   0x40010000      6cf09fe5       invalid
    ⁝⁝⁝⁝    0x40010004      6cf09fe5       invalid
    ⁝⁝⁝     0x40010008      6cf09fe5       invalid
    ⁝⁝      0x4001000c      6cf09fe5       invalid
    ⁝       0x40010010      6cf09fe5       invalid
            0x40010014      6cf09fe5       invalid
            0x40010018      6cf09fe5       invalid
            0x4001001c      6cf09fe5       invalid
            0x40010020      0045           cmp r0, r0
            0x40010022      544c           ldr r4, [0x40010174]        ; [0x40010174:4]=0xeb5511f6
[0x40010000]> 
```

However they're not parsed correctly cause they are being parsed as 16-bit Thumb mode.

* Just like in IDA where you use Alt+G to change from Thumb mode, you can use hints in radare2 with the ```ahb``` command:

```
[0x40010000]> ahb 32 @ $$
[0x40010000]> pd 10
            0x40010000      6cf09fe5       ldr pc, [0x40010074]        ; [0x40010074:4]=0x40010108
            0x40010004      6cf09fe5       ldr pc, [0x40010078]        ; [0x40010078:4]=0x40010094
            0x40010008      6cf09fe5       ldr pc, [0x4001007c]        ; [0x4001007c:4]=0x400100a0
            0x4001000c      6cf09fe5       ldr pc, [0x40010080]        ; [0x40010080:4]=0x400100ac
            0x40010010      6cf09fe5       ldr pc, [0x40010084]        ; [0x40010084:4]=0x400100bc
            0x40010014      6cf09fe5       ldr pc, [0x40010088]        ; [0x40010088:4]=0x400100cc
            0x40010018      6cf09fe5       ldr pc, [0x4001008c]        ; [0x4001008c:4]=0x400100d0
            0x4001001c      6cf09fe5       ldr pc, [0x40010090]        ; [0x40010090:4]=0x400100e4
            0x40010020      0045544c       mrrcmi p5, 0, r4, r4, c0
            0x40010024      00524556       strbpl r5, [r5], -r0, lsl 4
[0x40010000]> 
```

Where:

* ```ahb 32``` is force 32 bit
* ```@ $$``` at current address

Then you can see the disassembly is parsed correctly.

* If you want to inspect the disassembly at one of those initial loaded addresse such as 0x40010108, you can with the following:

```
[0x40010000]> pd 10 @ 0x40010108
            0x40010108      00000fe1       mrs r0, apsr
            0x4001010c      010cc0e3       bic r0, r0, 0x100
            0x40010110      00f02fe1       invalid
            0x40010114      00104fe1       mrs r1, spsr
            0x40010118      0d20a0e1       mov r2, sp
            0x4001011c      0e30a0e1       mov r3, lr
            0x40010120      d1f021e3       msr iapsr, 0xd1
            0x40010124      01001ce3       tst ip, 1                   ; 1
            0x40010128      0370a001       moveq r7, r3
            0x4001012c      0260a001       moveq r6, r2
[0x40010000]> 

```

Or go to the address first with seek

```
[0x40010000]> s 0x40010108
[0x40010108]> pd 10
            0x40010108      00000fe1       mrs r0, apsr
            0x4001010c      010cc0e3       bic r0, r0, 0x100
            0x40010110      00f02fe1       invalid
            0x40010114      00104fe1       mrs r1, spsr
            0x40010118      0d20a0e1       mov r2, sp
            0x4001011c      0e30a0e1       mov r3, lr
            0x40010120      d1f021e3       msr iapsr, 0xd1
            0x40010124      01001ce3       tst ip, 1                   ; 1
            0x40010128      0370a001       moveq r7, r3
            0x4001012c      0260a001       moveq r6, r2
 
```

You can see the first issue here with an invalid instruction at 0x40010110 which I'm not sure about yet...

* In order to define a function you can use the ```af``` command. As an example, the ```cc_DecodeStatusIndMsg``` function at ```0x40a48ab8``` we can seek to with ```s``` and then define as a function with ```af``` and then name with ```afn```:

```
[0x40010108]> s 0x40a48ab8
[0x40a48ab8]> af
[0x40a48ab8]> afn cc_DecodeStatusIndMsg
[0x40a48ab8]> pd 10
┌ (fcn) cc_DecodeStatusIndMsg 2134
│   cc_DecodeStatusIndMsg (int arg2);
│           ; var int local_0h @ sp+0x0
│           ; var int local_4h @ sp+0x4
│           ; var int local_38h @ sp+0x38
│           ; var int local_3ch @ sp+0x3c
│           ; var int local_40h @ sp+0x40
│           ; var int local_44h @ sp+0x44
│           ; var int local_48h @ sp+0x48
│           ; var int local_4ch @ sp+0x4c
│           ; var int local_50h @ sp+0x50
│           ; var int local_54h @ sp+0x54
│           ; var int local_58h @ sp+0x58
│           ; var int local_5ch @ sp+0x5c
│           ; var int local_60h @ sp+0x60
│           ; var int local_64h @ sp+0x64
│           ; var int local_68h @ sp+0x68
│           ; var int local_6ch @ sp+0x6c
│           ; var int local_70h @ sp+0x70
│           ; var int local_74h @ sp+0x74
│           ; var int local_78h @ sp+0x78
│           ; var int local_7ch @ sp+0x7c
│           ; var int local_80h @ sp+0x80
│           ; var int local_84h @ sp+0x84
│           ; var int local_88h @ sp+0x88
│           ; var int local_8ch @ sp+0x8c
│           ; var int local_90h @ sp+0x90
│           ; var int local_94h @ sp+0x94
│           ; var int local_98h @ sp+0x98
│           ; var int local_9ch @ sp+0x9c
│           ; var int local_a0h @ sp+0xa0
│           ; var int local_a4h @ sp+0xa4
│           ; var int local_a8h @ sp+0xa8
│           ; var int local_b0h @ sp+0xb0
│           ; var int local_cch @ sp+0xcc
│           ; arg int arg2 @ r1
│           0x40a48ab8      2de9f04f       push.w {r4, r5, r6, r7, r8, sb, sl, fp, lr}
│           0x40a48abc      dff8b4a0       ldr.w sl, [0x40a48b74]      ; [0x40a48b74:4]=0x415c8c29
│           0x40a48ac0      b7b0           sub sp, 0xdc
│           0x40a48ac2      9af80000       ldrb.w r0, [sl]
│       ┌─< 0x40a48ac6      58b1           cbz r0, 0x40a48ae0
│       │   0x40a48ac8      4749           ldr r1, [0x40a48be8]        ; [0x40a48be8:4]=0x41f2db18
│       │   0x40a48aca      0091           str r1, [sp]
│       │   0x40a48acc      411c           adds r1, r0, 1
│       │   0x40a48ace      4520           movs r0, 0x45               ; 'E'
│       │   0x40a48ad0      40ea8140       orr.w r0, r0, r1, lsl 18

```

As you can see the arguments are detected and using the list all functions command ```afl``` you can see we have our first function defined:

```
[0x40a48ab8]> afl
0x40a48ab8  130 2360 -> 2134 cc_DecodeStatusIndMsg
[0x40a48ab8]>
```

* There are a couple of decompiling options in radare2 that we can try, retdec, r2dec and the built in ```pdc``` command. I've only got the r2dec and ```pdc``` command working at the moment:

```c
[0x40a48ab8]> pdc
function cc_DecodeStatusIndMsg () {    //  130 basic blocks

    loc_0x40a48ab8:

       push (r4, r5, r6, r7, r8, sb, sl, fp, lr)
       sl = [pc + 0xb4]         //[0x40a48b74:4]=0x415c8c29
       sp -= 0xdc
       ldrb.w r0,[sl] 
       if !r0 jmp 0x40a48ae0    //likely
       {
        loc_0x40a48ae0:

           r0 = [pc + 0x108]        //[0x40a48bec:4]=0x41f2db34
           r1 = [pc + 0x88]         //[0x40a48b6c:4]=0xfecdba98
           [sp] = r0
           r0 = [pc + 0xb8]         //[0x40a48ba0:4]=0x40045
           [sp + 4] = r0
           r0 = sp                  //r13
           0x405e9f54()             //CALL: 0x7f96, 0xa5169610, 0x7fff, 0xa7a8da60
       do
       {
            loc_0x40a48af0:

          //CODE XREF from cc_DecodeStatusIndMsg (0x40a48ade)
               r0 = 0
               r1 = 0xff
               [sp + 0x68] = r0
``` 

```c
[0x40a48ab8]> pdd
/* r2dec pseudo C output */
#include <stdint.h>
 
void cc_DecodeStatusIndMsg () {
    sl = 0x40a48b74;
    r0 = sl;
    if (r0 != 0) {
        r1 = 0x40a48be8;
        *(sp) = r1;
        r1 = r0 + 1;
        r0 = 0x45;
        r0 |= (r1 << 18);
        r1 = 0x40a48b6c;
        *((uint32_t*)((uint8_t*) sp + local_4h)) = r0;
        r0 = sp;
        0x405e9f54 ();
    } else {
        r0 = 0x40a48bec;
        r1 = 0x40a48b6c;
        *(sp) = r0;
        r0 = 0x40a48ba0;
        *((uint32_t*)((uint8_t*) sp + local_4h)) = r0;
        r0 = sp;
        0x405e9f54 ();
    }
    r0 = 0;
    r1 = 0xff;
    *((uint32_t*)((uint8_t*) sp + local_68h)) = r0;
    r5 = r0;
    *((uint32_t*)((uint8_t*) sp + local_7ch)) = r0;
    r4 = 0;
    *((uint32_t*)((uint8_t*) sp + local_90h)) = r0;
```

* So that's as far as I've gotten and I haven't yet looked at the function finder script but that can be next. I assume that adapting the logic of the script to find functions in radare2 and then using the ```af``` command will be the way to go using r2pipe. Also the invalid instructions may need to be addressed as well...
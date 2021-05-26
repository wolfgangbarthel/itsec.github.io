# Local SMEP bypass via Ropchain Windows10 64bit


We first need the kernel base adress. With WinDbg this can be achieved by looking for the `nt` loaded modules via `lm sm`.

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

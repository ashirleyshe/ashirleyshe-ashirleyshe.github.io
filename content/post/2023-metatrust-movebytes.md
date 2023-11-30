---
title: "Metatrust CTF Writeup - BytesMove"
date: 2023-09-18
draft: false
subtitle:    ""
description: ""
author:      "Ashley Hsu"
image:       ""
tags:        ["CTF"]
categories:  ["CTF"]
---

> move the bytes to see the truth!  
> code: https://github.com/MetaTrustLabs/ctf/tree/master/bytesMove  
> you need to crack this file and find the flag.  
> tips: This is a sui bytecode

To disassemble the moveBytes.mv file in the MetaTrust CTF, we used the sui client command-line interface. The sui move disassemble command allowed us to view the assembly code for the Move bytecode.
```
sui move disassemble moveBytes.mv
```

If you're interested in viewing the assembly code for the moveBytes.mv file, you can check out the code at the following link:
https://gist.github.com/ashirleyshe/04a5ed08727a5b06fcda2fc545ed56b8

The file contains several functions, including compute, byte_to_u64, slice, and solve. The byte_to_u64 and slice functions are not particularly important.

Check the value of the constants:
```
Constants [
	0 => u64: 0000000000000000
	1 => u64: 0100000000000000
	2 => vector<u64>: 00
	3 => vector<u8>: 046374667b     // ctf{
	4 => vector<u8>: 017d           // }
	5 => vector<u64>: 20c0000000000000000c000000000000004500000000000000c200000000000000b800000000000000d600000000000000780000000000000010000000000000003a000000000000000b00000000000000f9000000000000004c00000000000000ae00000000000000ba00000000000000a20000000000000083000000000000004d00000000000000d100000000000000f200000000000000a7000000000000000e00000000000000df000000000000002e0000000000000047000000000000000100000000000000a00000000000000080000000000000001900000000000000f500000000000000260000000000000086000000000000004a00000000000000
    // [192, 12, 69, 194, 184, 214, 120, 16, 58, 11, 249, 76, 174, 186, 162, 131, 77, 209, 242, 167, 14, 223, 46, 71, 1, 160, 128, 25, 245, 38, 134, 74]
]
```

The `solve` function is the entry point for the program and accepts a vector as its argument, which represents the flag. The function checks the format of the input, computes a value using the compute function, and then asserts that the input is equal to the ciphertext value.
```
entry public solve(Arg0: vector<u8>) {
    ...
    check the format of the input
    compute
    assert the input is equal to the ciphertext (constants 5)
}
```

A loop, check `i < 32`.
```
	115: LdU64(0)
	116: StLoc[4](loc3: u64)
B14:
	117: CopyLoc[4](loc3: u64)
	118: LdU64(32)
	119: Lt
	120: BrFalse(163)
    ...
    158: MoveLoc[4](loc3: u64)
	159: LdU64(1)
	160: Add
	161: StLoc[4](loc3: u64)
	162: Branch(117)
```

An inner loop, check `j < 8`.
```
	124: LdU64(0)
	125: StLoc[5](loc4: u64)
B17:
	126: CopyLoc[5](loc4: u64)
	127: LdU64(8)
	128: Lt
	129: BrFalse(155)
    ...
    150: MoveLoc[5](loc4: u64)
	151: LdU64(1)
	152: Add
	153: StLoc[5](loc4: u64)
	154: Branch(126)
```

The compute function takes the i/4th element of the vector as its first argument and the 2290631716 as its second argument.
```
B19:
	131: MutBorrowLoc[19](loc18: vector<u64>)
	132: CopyLoc[4](loc3: u64)
	133: LdU64(4)
	134: Div
	135: VecMutBorrow(2)
	136: StLoc[10](loc9: &mut u64)
	137: CopyLoc[10](loc9: &mut u64)
	138: ReadRef
	139: LdU64(2290631716)
	140: Call compute(u64, u64): u64 * u64      // compute(vec[i/4], 2290631716)
```

```
    141: StLoc[8](loc7: u64)
	142: MoveLoc[10](loc9: &mut u64)
	143: WriteRef
	144: MoveLoc[20](loc19: u64)
	145: LdU8(1)
	146: Shl                        // var << 1
	147: MoveLoc[8](loc7: u64)
	148: Xor
	149: StLoc[20](loc19: u64)
```
I construct the script like this:
```
output = []
for i in range(32):
    tmp = 0
    for j in range(8):
        (vec[i/4], out) = compute(vec[i/4], 2290631716)
        tmp = (tmp << 1) ^ out
    output.append(tmp)
```

Finally, you'll find it is a LFSR.
```
def compute(arg0, arg1):
    loc0 = (arg0 << 1) & 4294967295
    loc1 = arg0 & arg1 & 4294967295
    loc2 = 0
    while loc1 != 0:
        loc2 ^= (loc1 & 1)
        loc1 >>= 1
    loc0 ^= loc2
    return (loc0, lco2)
```

To crack the flag, I wrote a Python script:
```python
ct = [192, 12, 69, 194, 184, 214, 120, 16, 58, 11, 249, 76, 174, 186, 162, 131, 77, 209, 242, 167, 14, 223, 46, 71, 1, 160, 128, 25, 245, 38, 134, 74]

def rev_compute(vec, mask):
    curr = vec >> 1
    b = vec & 1
    vec >>= 1
    while mask:
        if mask & 1:
            b ^= vec & 1
        vec >>= 1
        mask >>= 1
    return curr | (b << 31)


mask = 2290631716
s0 = int.from_bytes(ct[:4], "big")
s1 = int.from_bytes(ct[4:8], "big")
s2 = int.from_bytes(ct[8:12], "big")
s3 = int.from_bytes(ct[12:16], "big")
s4 = int.from_bytes(ct[16:20], "big")
s5 = int.from_bytes(ct[20:24], "big")
s6 = int.from_bytes(ct[24:28], "big")
s7 = int.from_bytes(ct[28:32], "big")

for _ in range(32):
    s0 = rev_compute(s0, mask)
    s1 = rev_compute(s1, mask)
    s2 = rev_compute(s2, mask)
    s3 = rev_compute(s3, mask)
    s4 = rev_compute(s4, mask)
    s5 = rev_compute(s5, mask)
    s6 = rev_compute(s6, mask)
    s7 = rev_compute(s7, mask)

print(s7.to_bytes(4, "big") + s6.to_bytes(4, "big") + s2.to_bytes(4, "big") + s5.to_bytes(4, "big") + s4.to_bytes(4, "big") + s3.to_bytes(4, "big") + s1.to_bytes(4, "big") + s0.to_bytes(4, "big"))
```
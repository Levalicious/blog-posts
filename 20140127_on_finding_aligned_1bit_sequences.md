# Principles

Given is an arbitrary integer variable. How to find the index of the least significant bit (LSB) of the first `1`-bit sequence of length `>= n`? Assuming `n=4`, let's consider the following example of a random 32-bit integer value. The index we're looking for is 10 in this case (indicated by the `V` character).
```
    31      24 | 23      16 | 15    V 8 | 7      0
MSB  01000111  |  11111101  |  10111100 | 01101001  LSB
```

Using a series of bitwise *and* and *shift-right* operations the index of the LSB of the first `1111` sequence in the integer `x` can be found with the following trick.

``` cpp
x &= x>>1;
x &= x>>2;
index = __builtin_ffs(x)-1; // use _BitScanForward in Visual C++
```

After the first statement every `1` in `x` indicates the start of a `11` sequence. After the second statement every `1` in `x` indicates the start of a `1111` sequence. In the last statement the GCC intrinsic `__builtin_ffs()` (use `_BitScanForward()` if you're on Visual C++) returns the bit position of the first set bit, starting from the LSB.
Note, that it doesn't work to shift by four bits at once because it's necessary to *combine* neighboring `1`-bits to make sure that there are no `0`-bits in-between. The following example illustrates how shifting by 3 bits wrongly yields two isolated `1`-bits. In contrast, shifting by 2 bits correctly yields a sequence of 2 bits which can be further reduced into a single `1`-bit indicating the start of the `1111` sequence.

```
 shift by 2    shift by 3
  01111010      01111010
& 00011110    & 00001111
= 00011010    = 00001010
    ok           wrong
```

# Arbitrary sequence lengths

By cleverly choosing the number of bits to shift, it's even possible to extend this construction to find bit sequences which length is not a power of two. As the order of the *and-shift-right* operations has no relevance, the following algorithm can be used to compute the number of bits to shift in order to find the `n`-bit sequence index. The sum of shifted bits must be equal to `n-1` and the number of bits to shift is halved in each iteration. Therefore, the total number of executed iterations is `ceil(log2(n))`.

``` cpp
int FindBitSeqIndexLsb(int x, int n)
{
    assert(n >= 0 && n <= 32);

    while (n > 1)
    {
        const int shiftBits = n>>1;
        x &= (unsigned)x>>shiftBits; // shift in zeros from left
        n -= shiftBits;
    }

    return __builtin_ffs(x)-1; // use _BitScanForward in Visual C++
}
```

# Exact sequence length

The described method finds bit sequences of length `>= n`. In case you're looking for a bit sequence of exactly `n` bits, the following statement can be inserted right before the LSB scan is performed. This statement masks out any `1`-bit which has a `1` on its left or right side. All the remaining `1`-bits are isolated and indicate the start of a sequence of exactly `n` bits.

``` cpp
mask = (~(x<<1))&(~((unsigned)x>>1)); // shift in zeros from left and right
x &= mask;
```

# Sequence alignment

To account for aligned bit sequences, unaligned `1`-bits can be simply masked out from `x` before the LSB scan is performed. For example, to regard only bit sequences starting at *nibble* boundaries `x` can be modified with the operation `x &= 0x11111111` (it is `0x...11 = 0b...00010001`) to clear all bits not starting at an index which is a multiple of four.
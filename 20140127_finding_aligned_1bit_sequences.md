How to find the index of the least significant bit (LSB) of the first 1-bit sequence of length $n$ in an integer? Assuming $n=4$, let's consider the following example of an arbitrary 32-bit integer value. The index we're looking for is in this case 10 (indicated by the `V` character).
```
    31       24 23       16 15     V 8   7       0
MSB  0100 0111 | 1111 1101 | 1011 1100 | 0110 1001  LSB
```

Using a series of bitwise *and* (`&`) and *shift right* (`>>`) operations the index of the LSB of the first 1111-bit sequence in an integer `x` can be found with the code below. The `__builtin_ffs()` GCC intrinsic (use `_BitScanForward()` if you're on Visual C++) returns the bit position of the first set bit, starting from the LSB.

```
x &= x>>1;
x &= x>>2;
index = __builtin_ffs(x)-1; // use _BitScanForward on Visual C++
```

The first two lines shift 

Is it possible to use the construction above to find bit sequences which length is not a power of two? The answer is yes. By cleverly choosing the number of bits to shift, the index of the LSB of any sequence length can be obtained. As the order of the `and-shift` operations is irrelevant for the result, the following algorithm can be used to choose the shift amounts:

n = 9
s1 = n/2 = 4
s2 = (n-s1)/2 = 2
s3 = (n-s1-s2)/2 = 1
s4 = (n-s1-s2-s3)/2 = 1

The sum of all shift amounts must be equal to $n-1$.

Let's get back to our example above but this time $n=9$. So we're looking for the index of the LSB of the first 1-bit sequence of length 9.
# Introduction
From shader model 4 on in Direct3D 10/11 shader constants are grouped into constant buffers (cbuffers) to reduce API overhead and bandwidth required to pass shader constants from CPU to GPU. When declaring cbuffer elements certain packing rules are applied. First of all, cbuffer elements are aligned to 4-byte boundaries. Additionally, the shader compiler attempts to pack multiple cbuffer elements into `float4` variables to save size. When an element straddles the current `float4`, it is put into the next `float4`. Look at the following example:</p>

```
cbuffer // size is 28 bytes (1x float4 + 1x float3)
{
    float2 a;   // start of first float4, offset=0, size=8
    float  b;   // part of first float4, offset=8, size=4
//  float  pad; // invisible padding, offset=12, size=4
    float3 c;   // start of second float4, offset=16, size=12
                // (float3 does not fit into rest of last float4)
};
```

One exception to this rule are arrays declared inside cbuffers. For each element of such arrays a full `float4` is allocated, independent of the real array type. This saves ALU instructions for the address offset computations. However, it can lead to problems when reading from those arrays as they are not tightly packed anymore, but the data copied from CPU mostly is. Furthermore, as there is a maximum number of available cbuffer elements (currently 4096 `float4` variables) it can be necessary sometimes to avoid wasting any cbuffer space. Look at the following example.

```
cbuffer
{
    float vals[32]; // consumes 500 bytes, not 128 bytes
};
```

The size of the cbuffer above is 500 bytes, not 128 bytes as one might assume. Between every two array elements there are three padding `float` variables inserted. Note, that after the last array element no padding is done anymore. This is why the size is 500 bytes, not 512 bytes ((31×4+1)×4 bytes = 500 bytes).

# Array access difficulties
Now, if a tight block (unpadded) of 32 `float` values (128 bytes) is copied to the cbuffer, only the very first `float` at index 0 will be correctly read in the shader, because the cbuffer padding offsets every other array access after the first one. To avoid this the cbuffer from the previous example could be declared like this:</p>

```
cbuffer
{
    float4 vals[8]; // 8*4=32 floats = 128 bytes
};
```

Using that declaration the `vals` array is tightly packed, because no padding is performed. Thus, it contains exactly the data which is copied over from the CPU. However, now it is not possible anymore to address the individual `float` array elements without performing additional indexing calculations.

```
float flt = vals[i>>2][i&3];
```

To get rid of the indexing calculations the `vals` array could be "casted" (Is it really a cast? We come back to that in the next section.) into a `float` array. That way it is possible to read the tightly packed `float` data from `vals` without explicitly performing any additional indexing.

```
float fltArr[32] = (float[32])vals;
float flt = fltArr[i];
```

# Hidden inefficiencies?
Even though, it is convenient to access a block of tightly packed memory in a shader using the syntax above, there is an associated overhead when doing so. Arbitrarily accessing individual `float4` elements by an index variable, e.g. a loop counter, is a lot less efficient, because of additional indexing overhead. Let's consider the following shader compiled with -O3. It consists of four loops, each adding `count` elements of a `float` array from a cbuffer. The first loop obtains the `float` elements from a padded array, the other three loops from a tightly packed one.

```
cbuffer
{
    float  vals[512];  // for loop 1
    float4 tight[128]; // for loop 2 and 3
    uint   count;      // <= 512
};

float4 main(float4 hpos: SV_Position): SV_Target0
{
    float sum = 0.0f;
    uint i;

    for (i=0; i<count; i++) // loop 1
        sum += vals[i];

    for (i=0; i<count; i++) // loop 2
        sum += tight[i>>2][i&3];

    float valsLin[512] = (float[512])tight;
    for (i=0; i<count; i++) // loop 3
        sum += valsLin[i];

    for (i=0; i<count>>2; i++) // loop 4
        sum += tight[i].x+tight[i].y+tight[i].z+tight[i].w;

    return float4(sum, sum, sum, 1.0f);
}
```

The above shader was compiled four times. Each time everything which is not required to compile the respective loop was commented out. The generated code for each loop differs significantly concerning performance. Let's look at the generated assembly code for each loop, starting with the first one.

```
// loop 1
mov r0.xy, l(0,0,0,0)
loop
  uge r0.z, r0.x, cb0[511].y      // r0.z = r0.x >= count?
  breakc_nz r0.z                  // yes => break
  add r0.y, r0.y, cb0[r0.x + 0].x // r0.y += cb0[r0.x].x
  iadd r0.x, r0.x, l(1)           // r0.x++
endloop
```

This is the fastest way to iterate over all array elements. All instructions inside the loop except the `add` instruction are loop overhead. Though, the array elements are padded and thus, copying to the cbuffer from the CPU has to account for this. The disassembly of the second loop is shown below.

```
// loop 2
dcl_immediateConstantBuffer { { 1.000000, 0, 0, 0},
                              { 0, 1.000000, 0, 0},
                              { 0, 0, 1.000000, 0},
                              { 0, 0, 0, 1.000000} }
mov r0.xy, l(0,0,0,0)
loop
  uge r0.z, r0.x, cb0[128].x // r0.z = r0.x >= count?
  breakc_nz r0.z             // yes =>; break
  and r0.z, r0.x, l(3)       // r0.z = r0.x&3 (mod 4)
  ushr r0.w, r0.x, l(2)      // r0.w = r0.x>>2 (div 4)
  dp4 r0.z, cb0[r0.w + 0].xyzw, icb[r0.z + 0].xyzw // select
  add r0.y, r0.z, r0.y       // r0.y += r0.z
  iadd r0.x, r0.x, l(1)      // r0.x++
endloop
```

Here, the shader compiler generated code to dynamically select the `r0.z`<sup>th</sup> component of the `r0.w`<sup>th</sup> `float4`. The selection is performed based on a dot-product between the current `float4` and a `float4` from an immediate constant buffer, effectively zeroing all unwanted components. With this approach it is possible to access a non-padded array, though it requires three additional instructions inside the loop. Next, we have a look at the third loop.

```
// loop 3
dcl_indexableTemp x0[512], 4
mov x0[0].x, cb0[0].x // copy each array element into its
mov x0[1].x, cb0[0].y // own float4 register. all elements
mov x0[2].x, cb0[0].z // are copied, independent of the value
mov x0[3].x, cb0[0].w // of count
...
mov x0[508].x, cb0[127].x
mov x0[509].x, cb0[127].y
mov x0[510].x, cb0[127].z
mov x0[511].x, cb0[127].w
mov r0.xy, l(0,0,0,0)
loop
  uge r0.z, r0.x, cb0[128].x // same as loop 1
  breakc_nz r0.z
  mov r0.z, x0[r0.x + 0].x
  add r0.y, r0.z, r0.y
  iadd r0.x, r0.x, l(1)
endloop
```

It turns out that the syntactically easiest way to access the tightly packed `float` array is the least efficient. What might be interpreted as some sort of type-cast turns out to be a very costly copy operation. Every single `float` from the `vals` array gets copied into a padded temporary array `x0`. The loop it-self is identical to the first loop, accessing the temporary array `x0`. It follows the analysis of the fourth loop.

```
// loop 4
ushr r0.x, cb0[128].x, l(2) // divide count by 4
mov r0.yz, l(0,0,0,0)
loop
  uge r0.w, r0.y, r0.x                  // r0.w = r0.y >= count?
  breakc_nz r0.w                        // yes => break
  add r0.w, cb0[r0.y+0].y, cb0[r0.y+0].x // r0.w = sum of 4
  add r0.w, r0.w, cb0[r0.y + 0].z       // float4 components
  add r0.w, r0.w, cb0[r0.y + 0].w
  add r0.z, r0.w, r0.z                  // r0.z += r0.w
  iadd r0.y, r0.y, l(1)                 // r0.y++
endloop
```

This is the fastest way to calculate the sum of an unpadded `float` array. By explicitly specifying the components to sum up, one gets rid of the `dp4` instruction and the additional immediate cbuffer for masking. However, the indexing is of course not arbitrary anymore. The components are explicitly stated and thus, the compiler can generate optimal code. There are cases when that approach surely won't work.

# Summary
Accessing unpadded, multi-component variables by an index variable can result in very poorly performing code. Usually, the best way is to go for padded arrays. Though, sometimes it can be necessary to tightly pack arrays in cbuffers. Either if the amount of data to be passed is large, or the way the memory block is copied to the cbuffer cannot be changed to account for the padding. In that case the access pattern as seen in the second loop should be favoured.
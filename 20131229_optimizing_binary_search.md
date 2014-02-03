# Introduction
*Binary search* finds the index of a specified value (also called *search key*) in a pre-sorted array. Binary search is a very efficient and elegant algorithm which is used in a large number of data structures and algorithms. It has a runtime complexity of `O(log N)`, where `N` is the number of elements to be searched through, because in each step the remaining number of array elements is halved.

Even though, the binary search algorithm is generally extremely fast I recently designed a data structure where the binary search was the major bottleneck. The said data structure performs successively a vast number of binary searches on a very large, static array of integers. The array can have a size of up to 100 billion unsigned 32-bit integers which corresponds to around 372.5 GB of raw data. To search for the first occurrence of an arbitrary key in an array of 100 billion elements a binary search touches `floor(log2(N)+1)) = 30` elements, with `N = 100000000000`.

This is not an article about how to implement the binary search algorithm. More information on that can be found in plenty of resources online. The implementation of the optimization technique described in the rest of this post can be found in the accompanying [github](https://github.com/geidav/lut-binary-search) repository.

# Memory hierarchy
Considering the size of the data set the first few array accesses are guaranteed to cause *cache misses*; or even worse *page misses* depending on the amount of available main memory. On machines with little main memory a page miss requires the operating system to fetch the corresponding block containing the required array element from disk (obviously this would require some additional code logic). Accessing main memory is roughly 200x slower than accessing the first level cache. The random access time of a modern SSD is typically under 0.1 ms, which is roughly 1000x slower than accessing main memory. The random access times of spinning disks vary considerably and are usually in the order of a few milliseconds. Assuming a random access time of 5 ms, accessing a spinning disk is roughly 50000x slower than accessing main memory. Take these numbers with a grain of salt. They are by no means accurate and will be considerably different measured on concrete systems. However, the overall orders of magnitude are correct and give an intuitive feeling of how costly array accesses can be.

# Look-up tables
The previous paragraph should have cleared up why array accesses can be costly, especially when data is stored "far away" from the CPU. As a consequence it seems reasonable to try reducing the runtime costs of a binary search by reducing the number of array accesses needed for a search. This is a widely used optimization technique: an increased memory consumption is traded for increased performance.

The number of array accesses can be reduced by limiting the initial search range from `left = 0`, `right = N-1` to anything smaller which is guaranteed to still contain the search key. The simple binary structure of unsigned integers allows us to use some of the high bits as an index into a look-up table (LUT) containing the reduced search range for the corresponding key. The more bits are used the bigger is the resulting speed-up, but the more memory is used. Using 16 bits requires only a 512 KB LUT. Using 24 bits requires already a 128 MB LUT. Let's consider exemplary a 16-bit LUT where the highest 16-bit of the search key are mapped. Each LUT entry maps to a sub-range in the value array that can contain up to 65536 different values.

`0 [0x00000..0x0ffff]` -> `left` = index of first element >= 0, `right` = index of last element < 65536  
`1 [0x10000..0x1ffff]` -> `left` = index of first element >= 65536, `right` = index of last element < 131072  
`2 [0x20000..0x2ffff]` -> `left` = index of first element >= 131072, `right` = index of last element < 196608  
`...`  

Let's consider a search for the key 114910 which is in hex `0x0001c0de`. In order to obtain the LUT index for the key a right-shift by 16 bits is performed: `lutIdx = key>>16`, which results in `lutIdx = 0x0001 = 1`. The sub-range stored in the LUT at element 1 is guaranteed to contain all values in the range 65536 <= `key` < 131072. Following, the binary search, taking the search key, as well as the interval start and end as arguments, can be invoked by calling:

``` cpp
BinarySearch(key, Lut[lutIdx].left, Lut[lutIdx].right);
```

This is very little, simple and efficient code but how to populate the LUT? It turns out that populating the LUT can be done very efficiently in `O(N)` with a single, sequential pass over the input array. Even though, all elements must be touched once to populate the LUT, the sequential nature of the algorithm allows prefetching required data in advance and exploits the largely increased performance characteristics of spinning disks for sequential reads. The algorithm to populate the LUT is depicted in C++ below.

``` cpp
std::vector<std::pair<size_t, size_t>> Lut;

void InitLut(const std::vector<uint32_t> &vals, size_t lutBits)
{
    // pre-allocate memory for LUT
    const size_t lutSize = 1<<lutBits;
    const size_t shiftBits = 32-lutBits; // for 32-bit integers!
    Lut.resize(lutSize);

    // populate look-up-table
    size_t thresh = 0, last = 0;

    for (ssize_t i=0; i<(ssize_t)vals.size()-1; i++)
    {
        const size_t nextThresh = vals[i+1]>>shiftBits;
        Lut[thresh] = {last, i};

        if (nextThresh > thresh) // more than one LUT entry to fill?
        {
            last = i+1;
            for (size_t j=thresh+1; j<=nextThresh; j++)
                Lut[j] = {last, i+1};
        }

        thresh = nextThresh;
    }

    // set remaining thresholds that couldn't be found
    for (size_t i=thresh; i<Lut.size(); i++)
        Lut[i] = {last, vals.size()-1};
}
```

The LUT is implemented as an array of pairs. Each pair represents the interval the corresponding LUT entry maps to. To populate the LUT, the outer `for`-loop iterates over the pre-sorted array and calculates the number of LUT entries to fill between two successive array elements. The inner `for`-loop fills the LUT accordingly. As the start of each LUT entry can be derived from the end of the previous one, the `last` variable keeps track of the end of the last interval. The intervals are non-overlapping and it holds that the end of an interval is the beginning of the next interval minus one: `Lut[i].second = Lut[i+1].first-1`. As a consequence, the memory size of the LUT can be halved by storing only the interval starts and computing the interval ends as described. For simplicity reasons the depicted code is without this improvement, in contrast to the code in the accompanying [github](https://github.com/geidav/lut-binary-search) repository.

# Other data types
The presented LUT-based optimization only works for data types that are compatible with a simple bit-wise comparison. This means that more significant bits must have a bigger influence on the value's magnitude than less significant bits. So do signed integers and floats work as well? As long as they're positive there are no problems, because for any two positive signed integers or floats `x` and `y` it holds that if `x <= y` it follows that `UIR(x) <= UIR(y)`, where `UIR()` denotes the binary unsigned integer representation. However, negative values cause problems. Hence, when creating the LUT and when doing a range look-up the array elements and the search key must be mapped to an ordered range using a *bijective* mapping as described below.

## Signed integers
Signed integers are stored in *two's complement*, in which the most significant bit (MSB) represents the sign: if the MSB is set the value is negative, otherwise it's positive. Therefore, signed integers compare greater using bit-wise comparison than unsigned ones. It turns out, that by simply flipping the sign bit the ordering can be fixed. It's important to note that flipping the sign bit is only enough if two's complement is used. If a simple *signed magnitude* representation is used negative values are not automatically reversed. In that case not only the sign bit but also all the remaining bits have to flipped when the sign bit is set. For clarity look at the following example of two's complement integers where `FSB()` denotes the function that flips the sign bit: `FSB(-6 = 11111010) = 01111010 = 122 < FSB(-5 = 11111011) = 01111011 = 123 < FSB(2 = 00000010) = 10000010 = 130`. In contrast, if a signed magnitude representation is used flipping the sign bit is not sufficient anymore as the following example illustrates: `-6 < -5`, however `FSB(-6 = 10000110) = 00000110 = 6 > FSB(-5 = 10000101) = 00000101 = 5`.

## Floating point
Floating point values (I'm talking about IEEE 754) are a bit more nasty than signed integers. Their signed magnitude representation has exactly the same problem as signed magnitude represented integers have, outlined in the previous section. To overcome this problem all the remaining bits just have to be flipped as well if the sign bit was set. For further information read the article of Michael Herf about Radix Sort for floats on Stereopsis. One possible implementation in C++ is:

```
// taken from Michael Herf's article on Stereopsis
uint32_t MapFloat(float val)
{
    // 1) flip sign bit
    // 2) if sign bit was set flip all other bits as well
    const uint32_t cv = (uint32_t &)val;
    const uint32_t mask = (-(int32_t)(cv>>31))|0x80000000;
    return cv^mask;
}
```

#  Results
I analyzed the performance gain of the LUT optimization by searching 10 million times through an array of 1 billion 32-bit integers. The integers were uniformly distributed in the interval `[0, 0xffffffff]`. In each search a randomly determined key from the data set was used. The performance was measured on a laptop with a Core(TM) i7-3612QM CPU (*Ivy Bridge*) at 2.1 GHz with 6 MB L3 cache and 8 GB RAM. The results of the standard binary search and the LUT optimized binary search for different LUT sizes are depicted in the table below. The speed-up is related to C++ standard binary search function `std::lower_bound()`, not my own implementation used for the range reduced binary search.

Algorithm            | LUT size          | Time needed | Searches/second | Speed-up
:--------------------|:------------------|:------------|:----------------|:--------
`std::lower_bound()` | -                 | 9.373 secs  | 1.07 m          | -
LUT binary search    | 8-bit (2 KB)      | 8.592 secs  | 1.16 m          | 1.09x
LUT binary search    | 16-bit (512 KB)   | 3.873 secs  | 2.58 m          | 2.42x
LUT binary search    | 24-bit (128 MB)   | 1.99 secs   | 5.03 m          | 4.71x

The 16-bit LUT binary search is more than 2.5x, the 24-bit LUT binary search is even almost 5x faster than C++'s standard binary search implementation. The amount of required additional code is neglectable and the LUT memory size can be configured granularly by adjust the number of bits used to index the LUT. Thanks to the `O(N)` complexity of the LUT population algorithm the discussed optimization is even feasible in situations where the underlying value array changes from time to time. Especially, in situations where parts of the data set are only resident on hard disk the reduced number of array accesses saves a considerable amount of work. By using simple mappings the LUT optimization can be applied as well to signed integer and even floating point values. For a sample implementation checkout the accompanying [github](https://github.com/geidav/lut-binary-search) repository.
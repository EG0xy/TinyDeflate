# TinyDeflate

A deflate/gzip decompressor, as a C++11 template function,
that requires minimal amount of memory to work.

## Memory usage

* 160 bytes of automatic storage for length tables (320 elements, 4 bits each)
* 2160 bytes of automatic storage for huffman tree (638 elements, 27 bits each)
* 51 bytes of temporary automatic storage while a huffman tree is being generated (17 elements, 24 bits each)
* An assortment of automatic variables for various purposes (may be register variables, depending on the architecture and of the compiler wits)
* ABI mandated alignment losses

Total: 2320 bytes minimum, 2371+N bytes maximum

In addition, if you neither decompress into a raw memory area nor supply your own window function,
32768 bytes of automatic storage is allocated for the look-behind window.

## Tuning

If you can afford more RAM, there are three options in gunzip.hh that you can change:

* USE_BITARRAY_TEMPORARY_IN_HUFFMAN_CREATION : Change this to false to use additional 12 bytes of memory for a tiny boost in performance.
* USE_BITARRAY_FOR_LENGTHS : Change this to false to use additional 160 bytes of memory for a noticeable boost in performance.
* USE_BITARRAY_FOR_HUFFNODES : Change this to false to use additional 392 bytes of memory for a significant boost in performance.

All three settings at 'false' will consume 2940 bytes of automatic memory + alignment losses + callframes + spills.

## Unrequirements

* No dynamic memory is allocated under any circumstances, unless your user-supplied functors do it.
* Aside from assert() in assert.h and some template metaprogramming tools in type_traits, no standard library functions are used.
* No global variables.
* No exceptions or runtime type identification is required.
* Option to compile without constant arrays.

## Rationale

* Embedded platforms (Arduino, STM32 etc).
* ROM hacking

## Caveats

* Decompressor only. Deflate and GZIP streams are supported.
* Slower than your average inflate function. The template uses densely bitpacked arrays, which require plenty of bit-shifting operations for every access.
* The code obviously performs best on 32-bit or 64-bit platforms. Platforms where 32-bit entities must be synthesized from a number of 8-bit entities are at a disadvantage.

There are prototypes for which these conditions hold:
* Data is assumed to be valid. In case of invalid data, either an assert() will fail, or the program will write out of memory bounds.
* There is no way to prematurely abort decompression, other than by terminating the program (such as with a failing assertion).

And there are prototypes that are safe.

## Definitions

```C++
template<typename InputFunctor, typename OutputFunctor, typename WindowFunctor>
int Deflate(InputFunctor&& input, OutputFunctor&& output, WindowFunctor&& outputcopy); // (1) (2) (3)

template<typename InputFunctor, typename OutputFunctor>
int Deflate(InputFunctor&& input, OutputFunctor&& output); // (1) (2) (7)

template<typename InputFunctor, typename RandomAccessIterator>
int Deflate(InputFunctor&& input, RandomAccessIterator&& target); // (1) (7) (8)

template<typename InputFunctor, typename RandomAccessIterator>
int Deflate(InputFunctor&& input, RandomAccessIterator&& target, std::size_t target_limit); // (1) (4) (8)

template<typename InputFunctor, typename RandomAccessIterator>
int Deflate(InputFunctor&& input, RandomAccessIterator&& target_begin, RandomAccessIterator&& target_end); // (1) (5) (8)

template<typename ForwardIterator, typename OutputIterator>
int Deflate(ForwardIterator&& begin, ForwardIterator&& end, OutputIterator&& output); // (6) (7)

template<typename ForwardIterator, typename OutputFunctor>
int Deflate(ForwardIterator&& begin, ForwardIterator&& end, OutputFunctor&& output); // (2) (6) (7)

template<typename ForwardIterator, typename OutputFunctor, typename WindowFunctor>
int Deflate(ForwardIterator&& begin, ForwardIterator&& end, OutputFunctor&& output, WindowFunctor&& outputcopy); // (2) (3) (6)

template<typename InputIterator, typename OutputIterator>
void Deflate(InputIterator&& input, OutputIterator&& output); // (7) (9)

template<typename ForwardIterator, typename RandomAccessIterator>
int Deflate(ForwardIterator&& begin, ForwardIterator&& end, RandomAccessIterator&& target_begin, RandomAccessIterator&& target_end); // (5) (6) (8)
```

1) If the input functor (`input`) returns an integer type other than a `char`, `signed char`, or `unsigned char`,
and the returned value is smaller than 0 or larger than 255, the decompression aborts with return value 1.

2) If the output functor (`output`) returns a `bool`, and the returned value is `true`, the decompression aborts with return value 2.

3) If the window function returns an integer type, and the returned value is other than 0, the decompression aborts with return value 3.

4) If `target_limit` bytes have been written into `target` and the decompression is not yet complete, the decompression aborts with return value 2.

5) If `target_begin == target_end`, the decompression aborts with return value 2.

6) If `begin == end`, the decompression aborts with return value 1.

7) A separate 32768-byte sliding window will be automatically and separately allocated for the decompression.

8) The output data buffer is assumed to persist during the call and doubles as the sliding window for the decompression.

9) If the input iterator deferences into a value outside the 0 — 255 range, the decompression aborts with return value 1.

## Example use:

Decompress the standard input into the standard output (uses 32 kB automatically allocated window):

```C++
    Deflate([]()                   { return std::getchar(); },
            [](unsigned char byte) { std::putchar(byte); });
    
    // Or more simply:
    
    Deflate(std::getchar, std::putchar);
```

Decompress an array containing gzipped data into another array that must be large enough to hold the result. A window buffer will not be allocated.

```C++
    extern const char compressed_data[];
    extern unsigned char outbuffer[131072];
    
    unsigned inpos = 0;
    Deflate([&]() { return compressed_data[inpos++]; },
            outbuffer);
```

Same as above, but with range checking:

```C++
    extern const char compressed_data[];
    extern unsigned char outbuffer[131072];
    
    unsigned inpos = 0;
    int result = Deflate([&]() { return compressed_data[inpos++]; },
            outbuffer,
            sizeof(outbuffer));
    if(result != 0) std::fprintf(stderr, "Error\n");
```

Same as above, but with more range checking:

```C++
    extern const char compressed_data[];
    extern unsigned compressed_data_length;
    extern unsigned char outbuffer[131072];
    
    int result = Deflate(compressed_data, compressed_data + compressed_data_length,
                         outbuffer, outbuffer + sizeof(outbuffer));
    if(result != 0) std::fprintf(stderr, "Error\n");
```

Decompress using a custom window function (the separate 32 kB window buffer will not be allocated):

```C++
    std::vector<unsigned char> result;
    
    Deflate(std::getchar,
            [&](unsigned byte)
            {
                result.push_back(byte);
            },
            [&](unsigned length, unsigned offset)
            {
                 if(!length)
                 {
                     // offset contains the maximum look-behind distance.
                     // You could use this information to allocate a buffer of a particular size.
                     // length=0 case is invoked exactly once before any length!=0 cases are.
                 }
                 while(length-- > 0)
                 {
                     result.push_back( result[result.size()-offset] );
                 }
            });
```

# fpng
Very fast C++ .PNG writer for 24/32bpp images. fpng.cpp was written to see just how fast you can write .PNG's without sacrificing too much compression, and to determine how slow the existing .PNG writers are. (Turns out they are quite slow.) 

On average, fpng.cpp produces files roughly 1-15% smaller than [QOI](https://github.com/phoboslab/qoi) but is 1.4-1.5x slower on 24bpp images.

fpng.cpp compared to stb_image_write.h: fpng.cpp is over 10x faster with roughly 16-18% smaller files. (Surprisingly fpng.cpp is outputting smaller files on average vs. stb_image_write.h, using a substantially simpler/faster compressor.)

fpng.cpp compared to lodepng.cpp: fpng.cpp is 12-16x faster with roughly 15-18% larger files. 

Benchmark using the 184 QOI test images:

```
qoi:              4.69 secs   359.55 MB
fpng:             6.66 secs   353.50 MB
lodepng:          110.72 secs 300.14 MB
stb_image_write:  68.07 secs  425.64 MB
```

Benchmark using my 303 test images:

```
qoi:             2.25 secs  300.84 MB
fpng:            3.40 secs  253.26 MB
lodepng:         41.68 secs 220.40 MB
stb_image_write: 37.23 secs 311.41 MB
```

## Low-level description

fpng.cpp uses a custom image aware pixel-wise Deflate compressor which was optimized for simplicity over high compression ratio. It uses a simple LZRW1-style parser, all literals (except the PNG filter bytes) are output in groups of 3 or 4, all matches are multiples of 3/4 bytes, and it only utilizes a single dynamic Huffman block with one PNG IDAT block. It utilizes 64-bit registers and exploits unaligned little endian reads/writes. (On big endian CPU's it'll use 32/64bpp byteswaps.)

Passes over the input image and dynamic allocations are minimized, although it does use ```std::vector``` internally. The first scanline always uses filter #0, and the rest use filter #2 (previous scanline). It uses the fast CRC-32 code described by Brumme [here](https://create.stephan-brumme.com/crc32/). The original high-level PNG function (that code that writes the headers) was written by Alex Evans.

lodepng v20210627 fetched 12/18/2021

stb_image_write.h v1.16 fetched 12/18/2021

## Building

To build, compile from the included .SLN with Visual Studio 2019/2022 or use cmake to generate a .SLN file. For Linux/OSX, use "cmake ." then "make". Tested with MSVC 2019/gcc/clang.

I have only tested fpng.cpp on little endian systems. The code is there for big endian, and it should work, but it needs testing.

## Testing

From the "bin" directory, run "fpng_test.exe" or "./fpng_test" like this:

```fpng_test.exe <image_filename.png>```

To generate .CSV output only:

```fpng_test.exe -c <image_filename.png>```

## Using fpng 

To use fpng.cpp in other programs, copy fpng.cpp/.h and Crc32.cpp/.h into your project. No other configuration or files are needed. Computing the CRC-32 of buffers is a substantial proportion of overall compression time in fpng, so if you have a faster CRC-32 function you can modify `fpng_crc()` in fpng.cpp to call that instead. The one included in Crc32.cpp doesn't utilize any special CPU instruction sets, so it could be faster. 

`#include "fpng.h"` then call one of these C-style functions in the "fpng" namespace:

```
bool fpng_encode_image_to_memory(const void* pImage, int w, int h, int num_chans, bool flip, std::vector<uint8_t>& out_buf);
bool fpng_encode_image_to_file(const char* pFilename, const void* pImage, int w, int h, int num_chans, bool flip);
```

`num_chans` must be 3 or 4. There must be w*3*h or w*4*h bytes pointed to by pImage. The image row pitch is always w*3 or w*4 bytes. There is no automatic determination if the image actually uses an alpha channel, so if you call it with 4 you will always get a 32bpp .PNG file.

Note the adler32 function in fpng.cpp could be vectorized for higher performance.

## License

fpng.cpp/.h: Public Domain or zlib (your choice). See the end of fpng.cpp.

Crc32.cpp/.h: Copyright (c) 2011-2019 Stephan Brumme. zlib license:

https://github.com/stbrumme/crc32

# fpng
fpng is a very fast C++ .PNG reader/writer for 24/32bpp images. fpng.cpp was written to see just how fast you can write .PNG's without sacrificing too much compression.

This version of FPNG always uses PNG filter #2 and is limited to only RLE matches (i.e. LZ matches with a match distance of either 3 or 4). It's around 5% weaker than the original release, which used LZRW1 parsing. (I'll eventually add back in the original variant back in as an option.)

fpng.cpp compression compared to stb_image_write.h: 12-19x faster with roughly 5-11% avg. smaller files.
fpng.cpp decompression compared to stb_image_write.h: ~3x faster

Benchmarks using MSVC 2019, Xeon E5-2690 3.00 GHz

A real-world benchmark using my assortment of 303 24/32bpp test images we use for GPU texture compression benchmarks (mps="megapixels/second", sorted by compression rate):

```
                 comp_size  avg_comp_mps  avg_decomp_mps
fpng_1_pass:     293.10 MB  110.16 mps    162.01 mps
qoi:             300.84 MB  83.90 mps     138.18 mps
fpng_2_pass:     275.73 MB  68.32 mps     165.73 mps
lodepng:         220.40 MB  6.21 mps      27.66 mps
stb_image:       311.41 MB  5.76 mps      50.00 mps
```

A real-world benchmark using the 184 QOI test images (note 182 of the qoi test images don't have alpha channels, so this is almost entirely a 24bpp test):

```
                 comp_size  avg_comp_mps  avg_decomp_mps
fpng_1_pass:     392.45 MB	115.17 mps	  161.92 mps
fpng_2_pass:     374.76 MB  71.29 mps     164.12 mps
qoi:             359.55 MB  88.22 mps     156.24 mps
stb_image:       425.64 MB  5.71 mps      52.18 mps
lodepng:         300.14 MB  5.20 mps      29.63 mps
```

An artificial benchmark using the 184 QOI test images, but with the green channel swizzled into alpha and all images compressed as 32bpp (to create a correlated alpha channel, common in video game textures): 

```
                 comp_size  avg_comp_mps  avg_decomp_mps
qoi:             697.20 MB  154.43 mps    160.30 mps
fpng_1_pass:     540.61 MB	93.10 mps	  128.43 mps
fpng_2_pass:     487.99 MB	59.12 mps	  136.46 mps
stb_image:       486.44 MB  4.63 mps      46.25 mps
lodepng:         352.10 MB  4.25 mps      28.84 mps
```

## Low-level description

fpng.cpp uses a custom image aware pixel-wise Deflate compressor which was optimized for simplicity over high ratios. It uses a simple LZRW1-style parser, all literals (except the PNG filter bytes) are output in groups of 3 or 4, all matches are multiples of 3/4 bytes, and it only utilizes a single dynamic Huffman block with one PNG IDAT block. It utilizes 64-bit registers and exploits unaligned little endian reads/writes. (On big endian CPU's it'll use 32/64bpp byteswaps.)

Passes over the input image and dynamic allocations are minimized, although it does use ```std::vector``` internally. The first scanline always uses filter #0, and the rest use filter #2 (previous scanline). It uses the fast CRC-32 code described by Brumme [here](https://create.stephan-brumme.com/crc32/). The original high-level PNG function (that code that writes the headers) was written by [Alex Evans](https://gist.github.com/908299).

lodepng v20210627 fetched 12/18/2021

stb_image_write.h v1.16 fetched 12/18/2021

qoi.h fetched 12/18/2021

## Building

To build, compile from the included .SLN with Visual Studio 2019/2022 or use cmake to generate a .SLN file. For Linux/OSX, use "cmake ." then "make". Tested with MSVC 2019/gcc/clang.

I have only tested fpng.cpp on little endian systems. The code is there for big endian, and it should work, but it needs testing.

## Testing

From the "bin" directory, run "fpng_test.exe" or "./fpng_test" like this:

```fpng_test.exe <image_filename.png>```

To generate .CSV output only:

```fpng_test.exe -c <image_filename.png>```

There will be several output files written to the current directory: stbi.png, lodepng.png, qoi.qoi, and fpng.png. Statistics or .CSV data will be printed to stdout, and errors to stderr.

The test app decompresses fpng's output using lodepng, stb_image, and the fpng decoder to validate the compressed data. The compressed output has also been validated using [pngcheck](http://www.libpng.org/pub/png/apps/pngcheck.html).

## Using fpng 

To use fpng.cpp in other programs, copy fpng.cpp/.h and Crc32.cpp/.h into your project. No other configuration or files are needed. Computing the CRC-32 of buffers is a substantial proportion of overall compression time in fpng, so if you have a faster CRC-32 function you can modify `fpng_crc()` in fpng.cpp to call that instead. The one included in Crc32.cpp doesn't utilize any special CPU instruction sets, so it could be faster. 

`#include "fpng.h"` then call one of these C-style functions in the "fpng" namespace to encode an image:

```
bool fpng_encode_image_to_memory(const void* pImage, uint32_t w, uint32_t h, uint32_t num_chans, std::vector<uint8_t>& out_buf, uint32_t flags = 0);
bool fpng_encode_image_to_file(const char* pFilename, const void* pImage, uint32_t w, uint32_t h, uint32_t num_chans, uint32_t flags = 0);
```

`num_chans` must be 3 or 4. There must be ```w*3*h``` or ```w*4*h``` bytes pointed to by pImage. The image row pitch is always ```w*3``` or ```w*4``` bytes. There is no automatic determination if the image actually uses an alpha channel, so if you call it with 4 you will always get a 32bpp .PNG file.

The included fast decoder is designed to only decode PNG files created by fpng. However, it has a full PNG chunk parser, and when it detects PNG files not written by fpng it returns the error code "FPNG_DECODE_NOT_FPNG" so you can fall back to a general purpose PNG reader.

```
int fpng_get_info(const void* pImage, uint32_t image_size, uint32_t& width, uint32_t& height, uint32_t& channels_in_file);
int fpng_decode_memory(const void* pImage, uint32_t image_size, std::vector<uint8_t>& out, uint32_t& width, uint32_t& height, uint32_t& channels_in_file, uint32_t desired_channels);
int fpng_decode_file(const char* pFilename, std::vector<uint8_t>& out, uint32_t& width, uint32_t& height, uint32_t& channels_in_file, uint32_t desired_channels);
```

`pImage` and `image_size` point to the PNG file data.

width, height, channels_in_file will be set to the image's dimensions and number of channels, which will always be 3 or 4.

desired_channels must be 3 or 4. If the input PNG file is 32bpp and you request 24bpp, the alpha channel be discarded. If the input is 24bpp and you request 32bpp, the alpha channel will be set to 0xFF.

The return code will be fpng::FPNG_DECODE_SUCCESS on success, fpng::FPNG_DECODE_NOT_FPNG if the PNG file should be decoded with a general purpose decoder, or one of the other error values.

## Fuzzing

fpng's encoder (and decoder) has been fuzzed with random inputs and random image dimensions. The -e and -E options are used for this sort of fuzzing.

The fpng decoder's parser has been fuzzed with [zzuf](http://caca.zoy.org/wiki/zzuf) so far. For more efficient decoder fuzzing (and more coverage), set FPNG_DISABLE_DECODE_CRC32_CHECKS to 1 in fpng.cpp before fuzzing.

## License

fpng.cpp/.h: Apache 2.0 license. See the end of fpng.cpp.

Crc32.cpp/.h: Copyright (c) 2011-2019 Stephan Brumme. zlib license:

https://github.com/stbrumme/crc32

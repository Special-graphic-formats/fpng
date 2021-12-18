# fpng
Very fast C++ .PNG writer for 24/32bpp images. On average, it produces files 1-15% smaller than [QOI](https://github.com/phoboslab/qoi) but is 1.4-1.5x slower on 24bpp images.

fpng.cpp compared to stb_image_write.h: fpng.cpp is over 10x faster with roughly 16-18% smaller files.

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

lodepng v20210627 fetched 12/18/2021

stb_image_write.h v1.16 fetched 12/18/2021

## Building

To build, compile from the included .SLN with Visual Studio 2019/2022 or use cmake to generate a .SLN file. For Linux/OSX, use "cmake ." then "make". Tested with MSVC 2019/gcc/clang.

I have only tested fpng.cpp on little endian systems. The code is there for big endian, and it should work, but it needs testing.

To use fpng.cpp in other programs, copy the fpng.cpp/.h and Crc32.cpp/.h into your project. No other configuration or files are needed. Computing the CRC-32 of buffers is a substantial proportion of overall compression time in fpng, so if you have a faster CRC-32 function you can modify `fpng_crc()` in fpng.cpp to call it instead. The one included does not utilize any special CPU instruction sets, so it could be faster. 

Also, the adler32 function in fpng.cpp could be vectorized for higher performance.

## Testing

From the "bin" directory, run "fpng_test.exe" or "./fpng_test" like this:

```fpng_test.exe <image_filename.png>```

To generate .CSV output only:

```fpng_test.exe -c <image_filename.png>```

## License

fpng.cpp/.h: Public Domain or zlib (your choice). See the end of fpng.cpp.

Crc32.cpp/.h: Copyright (c) 2011-2019 Stephan Brumme. zlib license:

https://github.com/stbrumme/crc32

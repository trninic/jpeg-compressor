# jpeg-compressor

A small (~1000 lines), easy to use public domain (or Apache 2.0) C++ class in a single source file [jpge.cpp](https://github.com/orian/jpeg-compressor/blob/master/jpge.cpp) that writes baseline [JPEG](http://en.wikipedia.org/wiki/JPEG) compressed images. It supports grayscale and H1V1/H2V1/H2V2 [chroma subsampling](http://en.wikipedia.org/wiki/Chroma_subsampling) factors, [Libjpeg](http://en.wikipedia.org/wiki/Libjpeg)-compatible quality settings, and is reasonably fast with fairly low (typically less than 64KB) memory consumption. The core compression class consists of a single 890 line C++ file with a small header, along with a couple optional higher-level helper/example functions. The current release supports both single pass [Huffman coding](http://en.wikipedia.org/wiki/Huffman_coding) and more efficient (but slower) two pass coding, makes only a single dynamic memory allocation, and now accepts 32-bit source images.

The source distribution also includes an optional, completely stand-alone public domain (or Apache 2.0) JPEG decompression class with progressive image support in a single source file [jpgd.cpp](https://github.com/orian/jpeg-compressor/blob/master/jpgd.cpp). It supports both box and linear chroma upsampling, and grayscale or H1V1/H2V1/H1V2/H2V2 chroma upsampling factors. Unlike every other small JPEG decompressor I've seen, this decompressor has been fuzz tested using zzuf and afl, making it resilent against crashing, overwriting memory, or bad memory reads when given accidently or purposely corrupted inputs. Also unlike many other small JPEG decompressors, jpgd.cpp does not require loading the entire image into memory, just single MCU rows at a time (even when doing linear chroma upsampling). This property makes the decompressor useful on small 32-bit microcontrollers.

The source distribution includes a sample VS2019 Win32/x64 solution and CMakeLists.txt file for compilation with gcc/clang.

Thanks to Alex Evans for adding several features to jpge (see a [smaller jpg encoder](http://altdevblogaday.org/2011/04/06/a-smaller-jpg-encoder/)).

## Basic Usage (Compression)

Include [jpge.h](https://github.com/orian/jpeg-compressor/blob/master/jpge.h) and call one of these helper functions in the "jpge" namespace:
``` cpp
// Writes JPEG image to a file. 
// num_channels must be 1 (Y), 3 (RGB), or 4 (RGBA), image pitch must be width*num_channels.
bool compress_image_to_jpeg_file(const char *pFilename, int width, int height, int num_channels, 
                                 const uint8 *pImage_data, const params &comp_params = params());

// Writes JPEG image to memory buffer. 
// On entry, buf_size is the size of the output buffer pointed at by pBuf, which should be at least ~1024 bytes. 
// If return value is true, buf_size will be set to the size of the compressed data.
bool compress_image_to_jpeg_file_in_memory(void *pBuf, int &buf_size, int width, int height, int num_channels, 
                                           const uint8 *pImage_data, const params &comp_params = params());
```
See [tga2jpg.cpp](https://github.com/orian/jpeg-compressor/blob/master/tga2jpg.cpp) for an example usage. This example uses Sean Barrett's [stb_image.c](http://www.nothings.org/stb_image.c) module to load image files.

You can also call the `jpge::jpeg_encoder class` directly if you need more control over the image source or how/where the output stream is written.

## Basic Usage (Decompression)

Include [jpgd.h](https://github.com/orian/jpeg-compressor/blob/master/jpgd.h) and call one of these helper functions in the "jpgd" namespace:

``` cpp
// Loads a JPEG image from a memory buffer.
// req_comps can be 1 (grayscale), 3 (RGB), or 4 (RGBA).
// On return, width/height will be set to the image's dimensions, and actual_comps will be set 
// to either 1 (grayscale) or 3 (RGB).
unsigned char *decompress_jpeg_image_from_memory(const unsigned char *pSrc_data, int src_data_size, 
                                     int *width, int *height, int *actual_comps, int req_comps);

// Loads a JPEG image from a file.
unsigned char *decompress_jpeg_image_from_file(const char *pSrc_filename, int *width, int *height, 
                                     int *actual_comps, int req_comps);
```
Just like the compressor, for more control you can directly utilize the jpgd::jpeg_decompressor class, or call the decompress_jpeg_image_from_stream() function.

jpgd supports a superset of JPEG features utilized by jpge, so it can decompress any file generated by jpge along with most (if not almost all) JPEG files you're likely to encounter in the wild. It supports progressive and baseline sequential JPEG image files and the H1V1, H2V1, H1V2, and H2V2 chroma subsampling factors.

## Testing

The source distribution includes a simple example command line tool called "jpge.exe" (or jpge_x64.exe for Win64 systems) that converts images from any format that [stb_image.c](http://www.nothings.org/stb_image.c) supports (such as PNG, TGA, BMP, etc.) to baseline JPEG using the jpeg_encoder class. Its usage is:
  bin/jpge source_file.png destination_file.jpg quality_factor
Where quality_factor ranges from 1-100 (higher is better).

jpge.exe also includes a few other modes for exhaustive testing of the codec (-x option) and JPEG to TGA decompression (-d option) -- see the help text for more info.

## Revisions

 - March 25, 2020 - Upgraded jpgd.cpp to latest version (fuzzed, linear chroma upsampling), added CMakeLists.txt, tested with clang/gcc under Linux, upgraded stb_image.h/stb_image_read.h to latest versions.
 - v1.04, May 20, 2012 - Fixed double `fclose()` bug in `cfile_stream::close()` reported by Owen Kaluza. (m_pFile should have been set to NULL here. Crap.) Also put jpge.cpp and jpgd.cpp through MSVC 2008's static code analysis and fixed every warning. They where all harmless things, but it's the [right thing to do](http://www.altdevblogaday.com/2011/12/24/static-code-analysis/).
 - v1.03, Apr. 16, 2011 - Added jpgd.cpp/.h (derived from my older [http://code.google.com/p/jpgd/](http://code.google.com/p/jpgd/) project), integrated changes from Alex Evans, added support for two pass compression, added more modes/options and timer to example command line app, lots of testing.
 - v1.02, Apr. 6, 2011 - Removed 2x2 ordered dither in the H2V1 chroma subsampling method jpeg_encoder::load_block_16_8_8(). (This was a last minute addition and there was actually a typo here - the dither rounding factor was 2 when it should have been 1. It turns out that chroma subsample dithering in the H2V1 case typically lowers PSNR even with the intended rounding factor, unlike H2V2.)
 - v1.01, Dec. 18, 2010 - Initial stable release

## History

This codec was originally written in C and 16-bit x86 asm way back in 1994 for a DOS image viewer. The primary reference was Pennebaker's and Mitchell's book [JPEG: Still Image Data Compression Standard](http://www.amazon.com/JPEG-Compression-Standard-Multimedia-Standards/dp/0442012721/). The original version supported a simple form of adaptive quantization, used a quantized DCT, and two pass Huffman coding. Around 2000 I ported it to C++, but I didn't really have the time to release it until now.

Note if you're generating texture mipmaps from loaded images, you may be interested in my [imageresampler](http://code.google.com/p/imageresampler/) project.

For any questions or problems with this code please contact Rich Geldreich at <richgel99 at gmail.com>. Here's my [twitter page](http://twitter.com/#!/richgel999).

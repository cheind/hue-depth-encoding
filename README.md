# hue-encoding
This project provides efficient vectorized Python code to perform depth `<->` color encoding based on

> Sonoda, Tetsuri, and Anders Grunnet-Jepsen.
"Depth image compression by colorization for Intel RealSense depth cameras." Intel, Rev 1.0 (2021).

Here's the encoding of depth values from a moving sine wave.

https://github.com/user-attachments/assets/2814d21a-3d1b-4857-b415-dc1c5ae31460


## Properties

The encoding is designed to transform 16bit single channel images to RGB color images that can be processed by standard (lossy) image codec with minimal compression artefacts. This leads to a compression factor of up to 80x.

## Method

The compression first transforms raw depth values to the applicable encoding range [0..1530] (~10.5bit) via a normalization strategy (linear/disparity) using a user provided near/far value. Next, for each encoding value the corresponding color is computed. The color is chosen carefully to respect the aforementioned properties. Upon decoding the reverse process is computed.

## Implementation

This implementation is vectorized using numpy and can handle any image shapes `(*,H,W) <-> (*,H,W,3)`. For improved performance, we precompute encoder and decoder lookup tables to reduce encoding/decoding to a simple lookup. The lookup tables require ~32MB of memory. Use `use_lut=False` switch to rely on the pure vectorized implementation. See benchmarks below for effects.

## Usage

There is currently no PyPi package to install, however the code is contained in a single file for easy of distribution :)

```python
# Import
import huecodec as hc

# Random float depths
d = rng.random((5,240,320), dtype=np.float32)*0.9 + 0.1

# Encode
rgb = hc.depth2rgb(d, zrange=(0.1,1.0), inv_depth=False)
# (5,240,320,3), uint8

# Decode
depth = hc.rgb2depth(rgb, zrange=(0.1,1.0), inv_depth=False)
# (5,240,320), float32
```

## Evaluation

The script `python analysis.py` compares encoding and decoding characteristics for different standard codecs using hue depth encoding. Each test encoded/decoded `(100,512,512)` np.float32 depthmaps in range [0..2] containing a sinusoidal pattern plus random hard depth edges. The pattern moves horizontally over time.

|    | variant       | zrange     |        rmse |       tenc |       tdec |   nbytes |
|---:|:--------------|:-----------|------------:|-----------:|-----------:|---------:|
|  0 | hue-only      | (0.0, 2.0) | 0.000332452 | 0.00630053 | 0.0499821  | nan      |
|  1 | hue-only      | (0.0, 4.0) | 0.00066518  | 0.00578579 | 0.00461386 | nan      |
|  2 | x264-lossless | (0.0, 2.0) | 0.000333902 | 0.014336   | 0.00734525 |  39.0091 |
|  3 | x264-lossless | (0.0, 4.0) | 0.000670176 | 0.0136169  | 0.00729958 |  31.4305 |
|  4 | x264-default  | (0.0, 2.0) | 0.0629023   | 0.0163551  | 0.00709273 |  29.8135 |
|  5 | x264-default  | (0.0, 4.0) | 0.120777    | 0.0198897  | 0.00761063 |  23.3559 |

The columns/units are
 - **rmse** [m] root mean square error per depth pixel between groundtruth and transcoded depthmaps
 - **tenc** [sec/frame] encoding time per frame
 - **tdec** [sec/frame] decoding time per frame
 - **nbytes** [kb/frame] kilo-bytes per encoded frame on disk.

### Hue Benchmark

Here are benchmark results for encoding/decoding float32 depthmaps of various sizes with differnt characteristics. Note, this is pure depth -> color -> depth transcoding without any video codecs involved.

```
------------- benchmark: 8 tests ------------
Name (time in ms)              Mean          
---------------------------------------------
enc_perf[LUT-(640x480)]        2.1851 (1.0)    
dec_perf[LUT-(640x480)]        2.2124 (1.01)   
enc_perf[LUT-(1920x1080)]     17.9139 (8.20) 
dec_perf[LUT-(1920x1080)]     16.4741 (7.54)   
enc_perf[noLUT-(640x480)]     22.3938 (10.25)  
dec_perf[noLUT-(640x480)]      6.9320 (3.17)   
dec_perf[noLUT-(1920x1080)]   74.6038 (34.14)  
enc_perf[noLUT-(1920x1080)]  158.0871 (72.35)  
---------------------------------------------
```

### Depth transformations
The encoding allows for linear and disparity depth normalization. In linear mode, equal depth ratios are preserved in the encoding range [0..1530], whereas in disparity mode more emphasis is put on closer depth values than on larger ones, leading to more accurare depth resolution closeup.

![](etc/compare_encoding.svg)

### Takeaways
Here are some takeaways
 - adjust zrange as tightly as possible to your use-case
 - prefer loss-less codecs if affordable

## Notes

The original paper referenced is potentially inaccurate in its equations. This has been noted in varios posts [#10415](https://github.com/IntelRealSense/librealsense/issues/10145),[#11187](https://github.com/IntelRealSense/librealsense/issues/11187),[#10302](https://github.com/IntelRealSense/librealsense/issues/10302).

This implementation is based on the original paper and code from
https://github.com/jdtremaine/hue-codec/.
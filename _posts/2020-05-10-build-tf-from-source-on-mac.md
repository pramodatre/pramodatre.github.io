---
layout: post
title:  "Building Tensorflow from Source on MacOS"
date:   2020-05-10 18:02:41 -0400
summary: If you like to make the best of your computers' hardware for performance gains up to 300%, you need to build tenforflow from source. This post will provide some tips for building tensorflow on MacOS.
use_math: false
---
Tensorflow is a popular choice if you are working on Computer Vision projects such as image classification, object detection, pose estimation, or just any Machine Learning task. I have a MacBook Pro and while using tensorflow, I noticed a warning when importing tensorflow which stated the following.

```shell
2020-05-08 19:57:50.106998: I tensorflow/core/platform/cpu_feature_guard.cc:142] Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX2 FMA
2020-05-08 19:57:50.128346: I tensorflow/compiler/xla/service/service.cc:168] XLA service 0x7febdd364b60 initialized for platform Host (this does not guarantee that XLA will be used). Devices:
2020-05-08 19:57:50.128370: I tensorflow/compiler/xla/service/service.cc:176] StreamExecutor device (0): Host, Default Version
```

This warning is issued because the tensorflow binary installed using `pip install tensorflow` was not built specifically for my machine. Since the binary should work on a wide varity of devices, the tensorflow binary on pip repository would be built such that it works on majority of the CPUs. Using this generic binary prevents tensorflow from using hardware specific optimizations. These optimizations lead to gain in performance which is especially important when using CPU-only version tensorflow. Since I use CPU-only configuration for tensorflow on my MacBook Pro (I do not have NVIDIA GPUs), this performance gain (up to 300% as mentioned [here](https://stackoverflow.com/questions/47068709/your-cpu-supports-instructions-that-this-tensorflow-binary-was-not-compiled-to-u)) is very much welcome!

If you observe the warning closely, you see `TensorFlow binary was not compiled to use: AVX2 FMA`. AVX (Advanced Vector Extensions) are instruction set extensions and specifically, FMA (Fused Multiply Accumulate) introduced by AVX speeds up linear algebra computation. For more information, you can read the stackoverflow post [here](https://stackoverflow.com/questions/47068709/your-cpu-supports-instructions-that-this-tensorflow-binary-was-not-compiled-to-u).

Motivated by 300% gain I could get, I started following instructions for building tensorflow from source using instructions from [official tensorflow website](https://www.tensorflow.org/install/source) and [stackoverflow post](https://stackoverflow.com/questions/41293077/how-to-compile-tensorflow-with-sse4-2-and-avx-instructions?rq=1).

**Before you build the code, if your MacOS version is < 10.14, I would recommend to upgrade it to 10.14 before you proceed**. I lost lot of time since I initially started with 10.13 MacOS version and Python 3.7 version and finally could not install the *.whl file using the pip command (this is the last step which will be outlined later). Note that building tensorflow from source is time consuming and memory intensive.

I followed these instructions to build tensorflow from source: https://www.tensorflow.org/install/source

Some issues and resolutions that worked for me:
* I had to explicit select Xcode using `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer` otherwise, I got compile errors when running `bazel build //tensorflow/tools/pip_package:build_pip_package`
* Having issues as I'm unable to install using `pip install /tmp/tensorflow_pkg/tensorflow-version-tags.whl`. I'm getting the following error
```shell
(tf-build) Pramods-MacBook-Pro:tensorflow pramodanantharam$ pip install /tmp/tensorflow_pkg/tensorflow-2.2.0-cp37-cp37m-macosx_10_14_x86_64.whl
ERROR: tensorflow-2.2.0-cp37-cp37m-macosx_10_14_x86_64.whl is not a supported wheel on this platform.
```
* To fix this, I upgraded my MacOS from 10.13 to 10.14 which resolved the issue.

Conclusion
* Building tensorflow from source is a good idea if you do not have GPU on your machine -- if you have a GPU version of tensorflow, you need not bother for this marginal performance gain.
* Having MacOS 10.14.x was necessary for me to complete the process of installing the binary in my virtual environment.
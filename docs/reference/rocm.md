# ROCm

ROCM is similar to Nvidia's CUDA toolkit, but for AMD's GPUs.

* The most recent release notes are found in the [docs.amd.com](https://docs.amd.com/category/ROCm%E2%84%A2%20v5.x) site.
* A translation table between CUDA and HIP calls (with a list of what is support and in which ROCm version): [HIP Supported CUDA API Guide v5.4](https://docs.amd.com/bundle/HIP_Supported_CUDA_API_Guide_v5.4/page/CUBLAS_API_supported_by_HIP.html). Newer versions should become available with new ROCm releases.

## ROCm driver compatibility

It is possible to use newer versions of the ROCm libraries and the HIP compiler together with older AMD GPU drivers in the kernel. This is useful as you cannot update the GPU drivers yourself on a HPC system without root access. Generally speaking, the ROCm userspace software tends to work with +/- 2 release of the kernel space software. The exact tested compatibility is not stated in the ROCm release notes, but you can find the compability table in the [ROCm Deep Learning Guide](https://docs.amd.com/bundle/ROCm-Deep-Learning-Guide-v5.4.3/page/Prerequisites.html) under "Prerequisites". For example, with the ROCm 5.1.3 drivers on the compute nodes (which you may not be able to change as user), you can still use ROCm 5.4.0 userland utilities to compile and run your own programs.

## Installing ROCm in a custom location

ROCm is normally installed in `/opt/rocm`. If, for various reasons, you would like to install several versions in your home directory, some modifications are typically necessary. At least for versions up to 5.3 (possibly latter), ROCm contains many hard-coded paths to `/opt/rocm/`, which it also inserts into compiled binaries, for example for generation just-in-time code in MIOpen. This creates some problems when using your own ROCm on a HPC system.
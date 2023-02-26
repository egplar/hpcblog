# How to compile CP2K 2023.1 with AMD GPU support

The [CP2K](https://www.cp2k.org/) software package is one the first to get AMD GPU support in many parts of package. Please see the [GPU](https://www.cp2k.org/gpu) page in the documentation for an overview of which parts are accelerated. RPA calculations and linear scaling DFT were part of the delivery acceptance benchmark as part of the procurement of the LUMI and Dardel clusters.

This guide is specifically for LUMI, but it should also apply to similar supercomputers like Frontier, Adastra, Setonix, and Dardel. It is based on the procedure described to me by Alfio Lazzaro at the HPE Center of Excellence for LUMI.

## Prerequisites

You need to have at least ROCm version 5.3, version 5.2 can compile the code, but it will not work properly (at least the DBCSR library) when you run it due to bugs in ROCm. You can check the version of ROCm available provided by the Cray programming environment with

    $ module avail rocm

    -------------------- /users/peterlarsson/modules --------------------
    rocm/5.3.3

    ------------------------ HPE-Cray PE modules ------------------------
    rocm/5.0.2 (D)

    ---------------------- Non-PE HPE-Cray modules ----------------------
    rocm/5.0.2


**Currently (Jan 2023), only ROCm 5.0 is available in LUMI, which means that you have to rely on an unsupported version of ROCm installed by the LUMI support team**. You can see this extra ROCm module by activating the `CrayEnv` module:

    module load CrayEnv rocm/5.3.3

I have used it successfully to run CP2K on the LUMI-G partition.

## Compile environment

Let us use Cray's GCC and FFTW libraries to compile CP2K. Starting from the default login environment (i.e. no `LUMI` or `spack` modules loaded):

    module load CrayEnv
    module load PrgEnv-gnu/8.3.3
    module load cray-fftw/3.3.10.1
    module load craype-accel-amd-gfx90a
    module load rocm/5.3.3

In some cases, you might need to load a module with a more recent version of `cmake`, but in this case, it still works with the system default cmake (3.17.0).

## Step 1: COSMA with GPU-aware MPI support

It is necessary to compile the COSMA library separately, in order to active GPU-aware MPI support in the library, as this is currently not done automatically in the toolchain install scripts for CP2K. This speeds up RPA calculations.

    git clone --recursive https://github.com/eth-cscs/COSMA COSMA-2.6.2
    cd COSMA-2.6.2
    mkdir build
    cd build
    mkdir install
    cmake -DCMAKE_INSTALL_PREFIX=${PWD}/install -DCOSMA_BLAS=ROCM -DCOSMA_SCALAPACK=CRAY_LIBSCI -DCOSMA_WITH_TESTS=NO -DCOSMA_WITH_BENCHMARKS=NO -DCMAKE_CXX_COMPILER=CC -DCOSMA_WITH_APPS=NO -DCOSMA_WITH_PROFILING=NO -DBUILD_SHARED_LIBS=NO -DCOSMA_WITH_GPU_AWARE_MPI=ON ..    
    make
    make install

The special configuration flag here is `-DCOSMA_WITH_GPU_AWARE_MPI=ON`. The build should be fast, even in serial mode, a few minutes maximum. If you check the `install/` directory, you should see the following:

    $ ls install/lib64/
    cmake  libcosma.a  libcosma_prefixed_pxgemm.a  libcosma_pxgemm_cpp.a  libcosta.a  libcosta_prefixed_scalapack.a  libcosta_scalapack.a  libTiled-MM.a  pkgconfig

These COSMA libraries will later be used by the CP2K install script. Unfortunately, the install script expects to find the libraries in `/lib` and not `/lib64`. We can fix this by just making a symbolic link.

    $ cd install
    $ ln -s lib64 lib

We are now ready to starting installing CP2K.

## Step 2: CP2K release version 2023.1

Checkout the latest CP2K release. In January 2023, this was "2023.1", which contains AMD GPU support.

    mkdir cp2k
    cd cp2k
    git clone --recursive https://github.com/cp2k/cp2k.git 2023.1
    cd 2023.1
    git checkout v2023.1

The first step is to build some libraries used by CP2K. This can be done with the provided `install_cp2k_toolchain` script:

    cd tools/toolchain
    ./install_cp2k_toolchain.sh -j 16 --enable-cray --enable-hip=yes --gpu-ver=Mi250 --with-cosma=/[path-to-cosma-above]/COSMA-2.6.2/build/install

The scripts provides several flags which are useful here: `enable-cray` to detect the Cray programming environment, `--enable-hip` for GPU support, but with AMD GPUs (HIP is like CUDA, but for AMD GPUs), and `--gpu-ver=Mi250` to select the GPU model corresponding to what is installed in LUMI, AMD MI250x. The output should begin like this:

    WARNING: (./install_cp2k_toolchain.sh, line 330) No MPI installation detected (ignore this message in Cray Linux Environment or when MPI installation was requested).
    ------------------------------------------------------------------------
    CRAY Linux Environment (CLE) is detected
    ------------------------------------------------------------------------
    path to cc is /opt/cray/pe/craype/2.7.17/bin/cc
    path to ftn is /opt/cray/pe/craype/2.7.17/bin/ftn
    path to CC is /opt/cray/pe/craype/2.7.17/bin/CC
    Found include directory /opt/cray/pe/mpich/8.1.18/ofi/gnu/9.1/include
    Found include directory /opt/cray/pe/mpich/8.1.18/ofi/gnu/9.1/lib
    libz is found in ld search path
    libdl is found in ld search path
    Compiling with 16 processes for target native.

Building all the dependencies can take a considerable amount of time, for example, `libint` alone can take 1 hour, even using many cores. Compiling everyting with 16 cores (`-j 16` above) took ca 40 minutes. In the end of the terminal output, there will be some important information. You should see several lines like this indicating that ROCm has been found:

    ...
    libhipblas is found in ld search path
    Found lib directory /pfs/lustrep1/projappl/project_4650000xx/rocm/rocm-5.3.3/lib
    ...

And an instruction to copy the "ARCH" files with compile settings to the base directory

    Now copy:
    cp /projappl/project_4650000xx/build/cp2k/2023.1/tools/toolchain/install/arch/* to the cp2k/arch/ directory
    To use the installed tools and libraries and cp2k version
    compiled with it you will first need to execute at the prompt:
    source /projappl/project_4650000xx/build/cp2k/2023.1/tools/toolchain/install/setup
    To build CP2K you should change directory:
    cd cp2k/
    make -j 16 ARCH=local VERSION="ssmp sdbg psmp pdbg"

After having copied the files to the `arch/` directory, we can go back to the CP2K base directory and compile the program:

    cd ../..
    make -j 16 ARCH=local_hip VERSION="psmp"

Please note that the ARCH file called `local-hip` should be used, not the one named just `local` as written in the terminal output above! Compiling CP2K itself should be faster. I recorded about 7 minutes. The binaries are found in then `exe/local_hip/` folder:

    $ ls exe/local_hip/
    cp2k.popt                         dbt_tas_unittest.psmp             grid_miniapp.psmp                 nequip_unittest.psmp
    cp2k.psmp                         dbt_unittest.psmp                 grid_unittest.psmp                parallel_rng_types_unittest.psmp
    cp2k_shell.psmp                   dumpdcd.psmp                      libcp2k_unittest.psmp             xyz2dcd.psmp
    dbm_miniapp.psmp                  graph.psmp                        memory_utilities_unittest.psmp 

A basic sanity check is to see if some HIP libraries were included in the binary at link time (they should be):

    $ ldd exe/local_hip/cp2k.psmp | grep hip
        libamdhip64.so.5 => /projappl/project_4625000xx/rocm/rocm-5.3.3/lib/libamdhip64.so.5 (0x00007fe4e6c2d000)
        libhipfft.so => /projappl/project_4650000xx/rocm/rocm-5.3.3/lib/libhipfft.so (0x00007fe4e99c8000)
        libhipblas.so.0 => /projappl/project_4650000xx/rocm/rocm-5.3.3/lib/libhipblas.so.0 (0x00007fe4e69cb000)

## Step 3: Test running on the GPU nodes

Here we will run the linear-scaling DFT benchmark provided with CP2K. It is a large simulation with 20,736 atoms. The linear scaling method uses the DBCSR matrix-matrix multiplication library, which uses GPUs. The benchmark located in the `benchmarks/QS_DM_LS/` folder. The recommended way to run is to use 8 MPI ranks per compute node (1 rank per GPU die), and 8 OpenMP threads per rank. As of January 2023, the so-called "low noise" mode is enabled on the compute nodes, which means that only 63 cores are available and the first core is reserved for the operating system. This requries some special CPU binding for best performance, and it also means that practically, it is easiest to run with 7 threads per rank (`OMP_NUM_THREADS=7`).

The job script:

    #!/bin/bash
    #SBATCH -J lsdft
    #SBATCH -p standard-g
    #SBATCH -A project_465000XXX
    #SBATCH --time=00:30:00
    #SBATCH --nodes=4
    #SBATCH --gres=gpu:8
    #SBATCH --exclusive
    #SBATCH --ntasks-per-node=8
    #SBATCH --cpus-per-task=7

    export OMP_PLACES=cores
    export OMP_PROC_BIND=close
    export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK}
    
    ulimit -s unlimited
    export OMP_STACKSIZE=512M

    export MPICH_OFI_NIC_POLICY=GPU
    export MPICH_GPU_SUPPORT_ENABLED=1

    module load rocm/5.3.3

    CP2K=/path/to/cp2k/2023.1/exe/local_hip/cp2k.psmp
    srun --cpu-bind=mask_cpu:0xfe,0xfe00,0xfe0000,0xfe000000,0xfe00000000,0xfe0000000000,0xfe000000000000,0xfe00000000000000 ./select_gpu.sh ${CP2K} -i H2O-dft-ls.inp -o out-4n8r7t8g.1

The `export MPICH_GPU_SUPPORT_ENABLED=1` is not strictly necessary, since it is the default value, but I keep it there for reference.

The `select_gpu.sh` helper script is useful to get the GPU to CPU binding correct on LUMI.

    $ cat select_gpu.sh 
    #!/bin/bash

    export ROCR_VISIBLE_DEVICES=$SLURM_LOCALID
    #export ROCR_VISIBLE_DEVICES=0,1

    if [[ "$SLURM_LOCALID" == "0" ]]; then
    ROCR_VISIBLE_DEVICES=4
    fi

    if [[ "$SLURM_LOCALID" == "1" ]]; then
    ROCR_VISIBLE_DEVICES=5
    fi

    if [[ "$SLURM_LOCALID" == "2" ]]; then
    ROCR_VISIBLE_DEVICES=2
    fi

    if [[ "$SLURM_LOCALID" == "3" ]]; then
    ROCR_VISIBLE_DEVICES=3
    fi

    if [[ "$SLURM_LOCALID" == "4" ]]; then
    ROCR_VISIBLE_DEVICES=6
    fi

    if [[ "$SLURM_LOCALID" == "5" ]]; then
    ROCR_VISIBLE_DEVICES=7
    fi

    if [[ "$SLURM_LOCALID" == "6" ]]; then
    ROCR_VISIBLE_DEVICES=0
    fi

    if [[ "$SLURM_LOCALID" == "7" ]]; then
    ROCR_VISIBLE_DEVICES=1
    fi
    
    echo "Node: " $SLURM_NODEID "Local task id:" $SLURM_LOCALID "ROCR_VISIBLE_DEVICES" $ROCR_VISIBLE_DEVICES
    exec $*

This script is useful for many applications using GPU on LUMI, not only CP2K.

When I run this job, I get a runtime of 537 (+/-1) seconds on 4 LUMI-G nodes (with 32 GPUs in total). This can be compared with ca 723 (+/-1) seconds on 4 LUMI-C nodes (using all 128 cores and `OMP_NUM_THREADS=8`). The GPU speed-up is not dramatic for linear-scaling DFT, and may not be cost-effective, but it is a good start. It is possible to improve the speed a bit with 2 MB huge memory pages (436 s), but this can also slow down some other kinds of calculations, so it is not foolproof. I expect, though, that some other methods, like RPA calculations, will show much better performance on GPUs.

I show some more benchmarks for CP2K on my [CP2K benchmark page](/benchmarks/cp2k-lumi).
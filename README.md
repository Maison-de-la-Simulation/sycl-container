# SYCL Container

This container contains an environment to run and compile [SYCL 2020](https://registry.khronos.org/SYCL/specs/sycl-2020/html/sycl-2020.html) codes on NVIDIA and AMD GPUs and any CPU with two SYCL compilers:
- AdaptiveCpp [v23.10.0](https://github.com/AdaptiveCpp/AdaptiveCpp/tree/v23.10.0)
- oneAPI DPC++ (Intel LLVM commit [589824d](https://github.com/intel/llvm/tree/589824d74532c85dee50e006cdc6685269eadfef))

## How to run container

### Singularity
- Documentation for [GPU Support](https://docs.sylabs.io/guides/3.5/user-guide/gpu.html)

```sh
singularity pull docker://ghcr.io/maison-de-la-simulation/sycl-complete

#optional: Mount directory
export SINGULARITY_BIND="/path/to/local/dir:/path/to/container/dir"

#Run container in shell
singularity shell --env OMP_NUM_THREADS=64 --env OMP_PLACES=cores sycl-complete.sif
singularity shell --nv sycl-complete.sif

#Or execute command inside container
singularity exec --nv   sycl-complete.sif acpp-info
singularity exec --rocm sycl-complete.sif sycl-ls
```

### Docker
```sh
docker pull ghcr.io/maison-de-la-simulation/sycl-complete

#Run container in interactive mode
docker run --it ghcr.io/maison-de-la-simulation/sycl-complete
..
```
## Use the SYCL compilers & runtime
Compilers are installed in `/opt/sycl`. The environment is setup in the `$PATH` so just type in `sycl-ls` (DPC++) or `acpp-info` (AdaptiveCpp) to list the SYCL devices. The associated runtimes are installed so once your code is compiled you should be able to run it within the container.

Following are examples to compile a SYCL code with the two different SYCL compilers. We assume the code is mounted inside /mnt/program.

### AdaptiveCpp
- Documentation for [using acpp](https://github.com/AdaptiveCpp/AdaptiveCpp/blob/develop/doc/using-hipsycl.md).

With AdaptiveCpp, set the `ACPP_TARGETS` before using the `syclcc` compiler.

```sh
#We build for a NVIDIA A100, AMD MI250x, and GPUs
export ACPP_TARGETS="omp;cuda:sm_80;hip:gfx90a"

cd /mnt/program/build #go into build folder

#use /opt/sycl/acpp/syclcc as CXX compiler
cmake -DCMAKE_CXX_COMPILER=syclcc ..
make -j 64
```

### DPC++
- Intel DPC++ [Users manual](https://intel.github.io/llvm-docs/UsersManual.html)

With this compiler, use the `-fsycl` and `-fsycl-targets` compilation flags to use ahead of time compilation. We use the cmake `COMPIL_FLAGS` variable to pass these parameters to the `clang++` compiler.

```sh
#for NVIDIA A100
cmake \
-DCMAKE_CXX_COMPILER=clang++ \
-DCOMPIL_FLAGS="-fsycl -fsycl-targets=nvidia_gpu_sm_80" \
..


#for Intel CPU with avx512 capabilities
clang++ -fsycl -fsycl-targets=spir64_x86_64 -Xsycl-target-backend "-march=avx512"
```

<!-- ## Tested hardware
We tested with a version of docker at least ... and singularity ...

**GPUs**
| NVIDIA            |      AMD      |
|:-----------------:|:-------------:|
| A100 (`sm_80`)    |    MI250x     |
| V100              |               |

**CPUs**
|       Intel      | AMD         |
|:----------------:|:-----------:|
| Xeon Gold 6230   | EPYC  ...   |
| ...              | GENOA ...   | -->

## Details
Versions of the backend used inside the container:
- **NVIDIA GPUs:** CUDA 11.8 via [NVIDIA base container](https://hub.docker.com/layers/nvidia/cuda/11.8.0-devel-ubuntu20.04/images/sha256-6e12af425102e25d3e644ed353072eca3aa8c5f11dd79fa8e986664f9e62b37a?context=explore)
- **AMD GPUs:** [ROCm](https://docs.amd.com/en/docs-5.5.0/deploy/linux/index.html) 5.5.1
- **OpenMP CPUs** (used for AdaptiveCpp CPUs compilation and runtime): [LLVM](https://apt.llvm.org/) 16.0.0
- **OpenCL Devices** (used for DPC++ CPUs runtime): OpenCL via [oneAPI DPC++ Get Started Guide](https://intel.github.io/llvm-docs/GetStartedGuide.html#install-low-level-runtime)

We use this specific commit ([589824d](https://github.com/intel/llvm/tree/589824d74532c85dee50e006cdc6685269eadfef)) of Intel/LLVM because there is a [known issue](https://github.com/intel/llvm/issues/4381) for multi NVIDIA-GPU backend resulting in a segfault when calling `sycl-ls` due to the `get_platform` function. Because we needed the `local_accessor`, a SYCL 2020 features, we had to find a commit before the sefgault was introduced and after the implementation of the `local_accessor`.

Tools:
- CMake 3.27
- vim 8.1
- git 2.25.1
- wget, unzip, ptyhon...

## Known issues
- `clang++-16` (llvm) and `clang++` (Intel llvm, the SYCL compiler) are not the same

- On some singularity configuration: `clang++` cannot write temporary file, you need to set the `TMPDIR` pointing to a directory inside the writable container
- On some singularity configuration: mount `/sys /dev /proc` inside the container (`SINGUALRITY_BIND="/sys,/dev,/proc"`)

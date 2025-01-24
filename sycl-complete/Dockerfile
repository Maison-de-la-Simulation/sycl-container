FROM nvidia/cuda:12.1.0-devel-ubuntu22.04

LABEL org.opencontainers.image.source=https://github.com/maison-de-la-simulation/sycl-container
LABEL org.opencontainers.image.description="SYCL compilers and runtime for DPCPP and ACPP"

ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    ccache \
    build-essential \
    cppcheck \
    gdb \
    git \
    libffi-dev \
    ninja-build \
    ocl-icd-libopencl1 \
    pkg-config \
    python3 \
    python3-pip \
    python-is-python3 \
    zstd \
    vim nano \
    unzip \
    libnuma-dev \
    wget \
    lsb-release software-properties-common gnupg \
    libboost-all-dev \
&& rm -rf /var/lib/apt/lists/*

########################################################
# Install cmake 3.27
RUN cd /opt \
&& wget https://github.com/Kitware/CMake/releases/download/v3.27.7/cmake-3.27.7-linux-x86_64.sh
RUN cd /usr/local \
&& sh /opt/cmake-3.27.7-linux-x86_64.sh --skip-license \
&& rm /opt/cmake-3.27.7-linux-x86_64.sh


########################################################
# Install ROCm 6.2.1 and depndencies
RUN apt update -y && apt install -y software-properties-common \
&& add-apt-repository universe \
&& add-apt-repository multiverse \
&& apt update -y \
&& apt install -y rsync dialog

RUN wget https://repo.radeon.com/amdgpu-install/6.2.1/ubuntu/jammy/amdgpu-install_6.2.60201-1_all.deb \
&& apt install ./amdgpu-install_6.2.60201-1_all.deb \
&& apt update -y \
&& apt install -y amdgpu-dkms rocm

########################################################
# Clone and checkout intel llvm (DPCPP)
RUN mkdir -p /opt/sycl/source \
&& git clone https://github.com/intel/llvm.git \
&& cd llvm \
&& git checkout 2024-WW43 \
&& mv /llvm /opt/sycl/source

########################################################
#Install OpenCL CPU runtime to get backend available in DPCPP
RUN cd /opt \
&& wget https://github.com/intel/llvm/releases/download/2024-WW43/oclcpuexp-2024.18.10.0.08_rel.tar.gz

RUN cd /opt \
&& wget https://github.com/oneapi-src/oneTBB/releases/download/v2022.0.0/oneapi-tbb-2022.0.0-lin.tgz

RUN mkdir -p /opt/intel/oclcpuexp-2024.18.10.0.08_rel \
&& cd /opt/intel/oclcpuexp-2024.18.10.0.08_rel \
&& mv /opt/oclcpuexp-2024.18.10.0.08_rel.tar.gz /opt/intel/oclcpuexp-2024.18.10.0.08_rel \
&& tar -zxvf oclcpuexp-2024.18.10.0.08_rel.tar.gz

RUN echo /opt/intel/oclcpuexp-2024.18.10.0.08_rel/x64/libintelocl.so > /etc/OpenCL/vendors/intel_expcpu.icd

RUN mkdir -p /opt/intel \
&& cd /opt/intel && mv /opt/oneapi-tbb-2022.0.0-lin.tgz /opt/intel \
&& tar -zxvf oneapi-tbb*lin.tgz

RUN rm /opt/intel/oclcpuexp-2024.18.10.0.08_rel/oclcpuexp-2024.18.10.0.08_rel.tar.gz \
&& rm /opt/intel/oneapi-tbb-2022.0.0-lin.tgz

RUN ln -s /opt/intel/oneapi-tbb-2022.0.0/lib/intel64/gcc4.8/libtbb.so /opt/intel/oclcpuexp-2024.18.10.0.08_rel/x64 \
&& ln -s /opt/intel/oneapi-tbb-2022.0.0/lib/intel64/gcc4.8/libtbbmalloc.so /opt/intel/oclcpuexp-2024.18.10.0.08_rel/x64 \
&& ln -s /opt/intel/oneapi-tbb-2022.0.0/lib/intel64/gcc4.8/libtbb.so.12 /opt/intel/oclcpuexp-2024.18.10.0.08_rel/x64 \
&& ln -s /opt/intel/oneapi-tbb-2022.0.0/lib/intel64/gcc4.8/libtbbmalloc.so.2 /opt/intel/oclcpuexp-2024.18.10.0.08_rel/x64 

RUN echo /opt/intel/oclcpuexp-2024.18.10.0.08_rel/x64 >> /etc/ld.so.conf.d/libintelopenclexp.conf \
&& ldconfig -f /etc/ld.so.conf.d/libintelopenclexp.conf

########################################################
# Build DPC++
ENV CUDA_LIB_PATH=/usr/local/cuda/lib64/stubs
RUN mkdir -p /opt/sycl/source/llvm/build \
    && python3 /opt/sycl/source/llvm/buildbot/configure.py \
      --cuda \
      --hip \
      --cmake-gen "Ninja" \
      --cmake-opt="-DCMAKE_INSTALL_PREFIX=/opt/sycl" \
      --cmake-opt="-DCUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda" \
      -t release \
      -o /opt/sycl/source/llvm/build \
    && python3 /opt/sycl/source/llvm/buildbot/compile.py -o /opt/sycl/source/llvm/build -j 8 \
    && rm -rf /opt/sycl/source/llvm/

RUN mkdir /opt/dpcpp \
&& mv /opt/sycl/* /opt/dpcpp \
&& mv /opt/dpcpp /opt/sycl

# ########################################################
# # Install LLVM
# RUN wget --progress=bar:force https://apt.llvm.org/llvm.sh \
# && chmod +x llvm.sh \
# && ./llvm.sh 17 \
# && apt install -y libclang-17-dev clang-tools-17 libomp-17-dev \
# && rm llvm.sh

# ########################################################
# # Install acpp 
# RUN cd /opt/sycl \
# && git clone --recurse-submodules https://github.com/AdaptiveCpp/AdaptiveCpp.git -b v24.02.0 \
# && cd AdaptiveCpp \
# && mkdir build && cd build \
# && cmake -G Ninja \
#     -DCMAKE_CXX_COMPILER=/usr/bin/clang++-17 \
#     -DCLANG_EXECUTABLE_PATH=/usr/bin/clang++-17 \
#     -DLLVM_DIR=/usr/lib/llvm-17/lib/cmake/llvm \
#     -DWITH_CUDA_BACKEND=ON -DWITH_ROCM_BACKEND=ON \
#     -DCUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda \
#     -DROCM_PATH=/opt/rocm \
#     -DCMAKE_INSTALL_PREFIX=/opt/sycl/acpp \
#     -DWITH_OPENCL_BACKEND=OFF \
#     .. \
# && ninja install \
# && cd /opt \
# && rm -rf /opt/sycl/AdaptiveCpp

# ########################################################
# #Env variables
# ENV PATH=/opt/sycl/acpp/bin:/opt/sycl/dpcpp/bin:$PATH
# ENV LD_LIBRARY_PATH=/opt/sycl/dpcpp/lib:/opt/sycl/acpp/lib:$LD_LIBRARY_PATH

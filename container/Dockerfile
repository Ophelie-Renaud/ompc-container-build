FROM rockylinux:9

ARG UCX_VERSION=1.20.1
ARG MPICH_VERSION=5.0.1

RUN dnf -y update && \
    dnf -y install \
        gcc \
        gcc-c++ \
        gcc-gfortran \
        make \
        wget \
        tar \
        gzip \
        bzip2 \
        xz \
        perl \
        python3 \
        git \
        hwloc \
        hwloc-devel \
        libibverbs \
        libibverbs-devel \
        librdmacm \
        librdmacm-devel \
        rdma-core \
        numactl \
        numactl-devel \
        libxml2 \
        libxml2-devel && \
    dnf clean all

# ---------------------------------------------------------------------------
# UCX 1.20.1
# ---------------------------------------------------------------------------

WORKDIR /tmp

RUN wget -q https://github.com/openucx/ucx/releases/download/v${UCX_VERSION}/ucx-${UCX_VERSION}.tar.gz && \
    tar xf ucx-${UCX_VERSION}.tar.gz && \
    cd ucx-${UCX_VERSION} && \
    ./configure \
        --prefix=/opt/ucx \
        --with-verbs \
        --with-rdmacm \
        --without-cuda \
        --disable-logging \
        --disable-debug && \
    make -j$(nproc) && \
    make install && \
    rm -rf /tmp/ucx-${UCX_VERSION}*

ENV PATH=/opt/ucx/bin:$PATH
ENV LD_LIBRARY_PATH=/opt/ucx/lib:$LD_LIBRARY_PATH
ENV PKG_CONFIG_PATH=/opt/ucx/lib/pkgconfig:$PKG_CONFIG_PATH

# ---------------------------------------------------------------------------
# MPICH 5.0.1 -- same device as the validated Sorgan build
# ---------------------------------------------------------------------------

WORKDIR /tmp

RUN wget -q https://www.mpich.org/static/downloads/${MPICH_VERSION}/mpich-${MPICH_VERSION}.tar.gz && \
    tar xf mpich-${MPICH_VERSION}.tar.gz && \
    cd mpich-${MPICH_VERSION} && \
    ./configure \
        --prefix=/opt/mpich \
        --enable-shared \
        --with-device=ch4:ucx \
        --with-ucx=/opt/ucx \
        --with-pm=hydra \
        --without-cuda \
        --disable-fortran && \
    make -j$(nproc) && \
    make install && \
    rm -rf /tmp/mpich-${MPICH_VERSION}*

ENV PATH=/opt/mpich/bin:/opt/ucx/bin:$PATH
ENV CPATH=/opt/mpich/include
ENV LIBRARY_PATH=/opt/mpich/lib
ENV LD_LIBRARY_PATH=/opt/mpich/lib:/opt/ucx/lib:$LD_LIBRARY_PATH
ENV PKG_CONFIG_PATH=/opt/mpich/lib/pkgconfig:/opt/ucx/lib/pkgconfig
ENV CMAKE_PREFIX_PATH=/opt/mpich:/opt/ucx

# ---------------------------------------------------------------------------
# CUDA Toolkit 12.9.1
# ---------------------------------------------------------------------------

ARG CUDA_VERSION=12.9.1

RUN dnf -y install dnf-plugins-core && \
    dnf config-manager --add-repo \
        https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo && \
    dnf -y install \
        cuda-cccl-12-9 \
        cuda-cudart-12-9 \
        cuda-cuobjdump-12-9 \
        cuda-cuxxfilt-12-9 \
        cuda-nvcc-12-9 \
        cuda-nvprune-12-9 && \
    dnf clean all

ENV CUDA_HOME=/usr/local/cuda-12.9
ENV PATH=/usr/local/cuda-12.9/bin:/opt/mpich/bin:/opt/ucx/bin:$PATH
ENV LD_LIBRARY_PATH=/usr/local/cuda-12.9/lib64:/opt/mpich/lib:/opt/ucx/lib:$LD_LIBRARY_PATH

# Verify that MPI headers are visible without explicit -I flags,
# matching the environment provided by the MPICH module on Sorgan.
RUN printf '#include <mpi.h>\nint main() { return 0; }\n' | \
    g++ -x c++ -fsyntax-only -

# ---------------------------------------------------------------------------
# LLVM / OpenMP Cluster source
# ---------------------------------------------------------------------------

ARG OMPC_SOURCE_REVISION

RUN dnf config-manager --set-enabled crb && \
    dnf -y install \
        cmake \
        ninja-build \
        python3-devel \
        zlib-devel \
        libedit-devel \
        ncurses-devel && \
    dnf clean all

WORKDIR /opt/src

RUN test -n "${OMPC_SOURCE_REVISION}" && \
    git clone https://github.com/OpenMP-Cluster/llvm-project.git llvm-project && \
    cd llvm-project && \
    git checkout --detach "${OMPC_SOURCE_REVISION}" && \
    echo "=== OMPC revision ===" && \
    git log -1 --format='%H %s'

# ---------------------------------------------------------------------------
# LLVM / Clang / LLD toolchain
# ---------------------------------------------------------------------------

WORKDIR /opt/src/llvm-project

RUN cmake -G Ninja \
        -S llvm \
        -B build-toolchain \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_INSTALL_PREFIX=/opt/ompc \
        -DLLVM_ENABLE_PROJECTS="clang;lld" \
        -DLLVM_TARGETS_TO_BUILD="X86;NVPTX" \
        -DLLVM_ENABLE_ASSERTIONS=Off && \
    cmake --build build-toolchain --parallel 4 && \
    cmake -G Ninja \
        -S runtimes \
        -B build-runtimes \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_INSTALL_PREFIX=/opt/ompc \
        -DLLVM_ENABLE_RUNTIMES="openmp;offload" \
        -DLLVM_BINARY_DIR=/opt/src/llvm-project/build-toolchain \
        -DLLVM_TARGETS_TO_BUILD="X86;NVPTX" \
        -DLLVM_ENABLE_PER_TARGET_RUNTIME_DIR=ON \
        -DLLVM_ENABLE_ASSERTIONS=Off \
        -DCMAKE_C_COMPILER=/opt/src/llvm-project/build-toolchain/bin/clang \
        -DCMAKE_CXX_COMPILER=/opt/src/llvm-project/build-toolchain/bin/clang++ \
        -DOPENMP_STANDALONE_BUILD=0 \
        -DLIBOMP_OMPX_TASKGRAPH=ON \
        -DLIBOMP_ENABLE_SHARED=ON \
        -DLIBOMPTARGET_PLUGINS_TO_BUILD="mpi;cuda;host" \
        -DLIBOMPTARGET_REMOTE_PLUGINS_TO_BUILD="cuda;host" \
        -DLIBOMPTARGET_BUILD_DEVICERTL_BCLIB=On \
        -DLIBOMPTARGET_ENABLE_DEBUG=On \
        -DCUDAToolkit_ROOT=/usr/local/cuda-12.9 \
        -DMPI_C_COMPILER=/opt/mpich/bin/mpicc \
        -DMPI_CXX_COMPILER=/opt/mpich/bin/mpicxx && \
    cmake --build build-runtimes --parallel 4 && \
    cmake --build build-runtimes --target llvm-offload-mpi-proxy-device --parallel 4 && \
    echo '=== MPI Proxy executable ===' && \
    find build-toolchain build-runtimes -type f -name llvm-offload-mpi-proxy-device -print -quit | grep -q . && \
    echo "=== build sizes before cleanup ===" && \
    du -sh build-toolchain build-runtimes && \
    find build-toolchain build-runtimes -type f \
        \( -name '*.o' -o \( -name '*.a' ! -name 'libompdevice.a' \) \) \
        -delete && \
    find build-toolchain build-runtimes -type d \
        \( -name CMakeFiles -o -name Testing \) \
        -prune -exec rm -rf {} + && \
    rm -rf \
        build-toolchain/.ninja_deps \
        build-toolchain/.ninja_log \
        build-toolchain/build.ninja \
        build-toolchain/CMakeCache.txt \
        build-runtimes/.ninja_deps \
        build-runtimes/.ninja_log \
        build-runtimes/build.ninja \
        build-runtimes/CMakeCache.txt && \
    echo "=== build sizes after cleanup ===" && \
    du -sh build-toolchain build-runtimes

ENV OMPC_BUILD=/opt/src/llvm-project/build-toolchain
ENV OMPC_RUNTIMES_BUILD=/opt/src/llvm-project/build-runtimes
ENV PATH=/opt/src/llvm-project/build-toolchain/bin:/usr/local/cuda-12.9/bin:/opt/mpich/bin:/opt/ucx/bin:$PATH
ENV LIBRARY_PATH=/opt/src/llvm-project/build-toolchain/lib:/opt/src/llvm-project/build-toolchain/lib/nvptx64-nvidia-cuda:/opt/src/llvm-project/build-runtimes/openmp/runtime/src:/opt/mpich/lib
ENV LD_LIBRARY_PATH=/opt/src/llvm-project/build-toolchain/lib:/opt/src/llvm-project/build-runtimes/openmp/runtime/src:/usr/local/cuda-12.9/lib64:/opt/mpich/lib:/opt/ucx/lib:$LD_LIBRARY_PATH

# Required by the OMPC host plugin (dlopen("libffi.so")).
RUN dnf -y install libffi-devel && dnf clean all

WORKDIR /workspace

CMD ["/bin/bash"]

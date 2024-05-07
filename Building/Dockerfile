##### Splitting installation into controlled parts:
## - Base container setup (all apt-get + system + ldd extension)
## - Python pip3 preparation + First batch of python tools installs
## - Tensorflow build and install (optional)
## - FFmpeg build and install
## - OpenCV build and install (using FFmpeg)
## - Additional python tools installs
## - Magma build and install (GPU only, optional)
## - PyTorch (+ vison + audio) build and install (using FFmpeg + OpenCV, optional)
## - Final steps: /iti setup, ...
##### https://github.com/Infotrend-Inc/CTPO/tree/main

# The idea is to have a base Dockerfile that will be tailored to specific builds
# so that each build has their own final Dockerfile available for inspection

ARG CTPO_FROM=nvidia/cuda:12.3.2-devel-ubuntu22.04
FROM ${CTPO_FROM}

# Adapted from https://gitlab.com/nvidia/container-images/cuda/-/blob/master/dist/11.8.0/ubuntu2204/devel/cudnn8/Dockerfile
ENV NV_CUDNN_VERSION 9.0.0.312
ENV NV_CUDA_ADD=cuda-12
ENV NV_CUDNN_PACKAGE_NAME "libcudnn9"
ENV NV_CUDNN_PACKAGE "$NV_CUDNN_PACKAGE_NAME-+$NV_CUDA_ADD=$NV_CUDNN_VERSION-1"
ENV NV_CUDNN_PACKAGE_DEV "$NV_CUDNN_PACKAGE_NAME-dev-+$NV_CUDA_ADD=$NV_CUDNN_VERSION-1"
LABEL com.nvidia.cudnn.version="${NV_CUDNN_VERSION}"

RUN apt-get update && apt-get install -y --no-install-recommends \
    ${NV_CUDNN_PACKAGE} \
    ${NV_CUDNN_PACKAGE_DEV} \
    && apt-mark hold ${NV_CUDNN_PACKAGE} \
    && rm -rf /var/lib/apt/lists/*

ARG CTPO_NUMPROC=8

##### Base

# Install system packages
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update -y --fix-missing\
  && apt-get install -y \
    apt-utils \
    locales \
    wget \
    ca-certificates \
  && apt-get clean

# UTF-8
RUN localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
ENV LANG en_US.utf8

# Install system packages
RUN apt-get install -y \
  # Base
    build-essential \
    checkinstall \
    cmake \
    curl \
    g++ \
    gcc \
    git \
    perl \
    patchelf \
    pkg-config \
    protobuf-compiler \
    python3-dev \
    rsync \
    time \
    unzip \
    wget \
    zip \
    zlib1g \
    zlib1g-dev \
  # OpenCV
    doxygen \
    file \
    gfortran \
    gnupg \
    gstreamer1.0-plugins-good \
    imagemagick \
    libatk-adaptor \
    libatlas-base-dev \
    libboost-all-dev \
    libbz2-dev \
    libcanberra-gtk-module \
    libdc1394-dev \
    libeigen3-dev \
    libfaac-dev \
    libfreetype6-dev \
    libgflags-dev \
    libglew-dev \
    libglu1-mesa \
    libglu1-mesa-dev \
    libgoogle-glog-dev \
    libgphoto2-dev \
    libgstreamer1.0-dev \
    libgstreamer-plugins-bad1.0-0 \
    libgstreamer-plugins-base1.0-dev \
    libgtk2.0-dev \
    libgtk-3-dev \
    libhdf5-dev \
    libhdf5-serial-dev \
    libjpeg-dev \
    liblapack-dev \
    libmp3lame-dev \
    libopenblas-dev \
    libopencore-amrnb-dev \
    libopencore-amrwb-dev \
    libopenjp2-7 \
    libopenjp2-7-dev \
    libopenjp2-tools \
    libopenjpip-server \
    libpng-dev \
    libpostproc-dev \
    libprotobuf-dev \
  # Libsql
    libsqlite3-dev \
    libtbb2 \
    libtbb-dev \
    libtheora-dev \
    libtiff5-dev \
    libv4l-dev \
    libvorbis-dev \
    libwebp-dev \
    libx264-dev \
    libx265-dev \
    libxi-dev \
    libxine2-dev \
    libxmu-dev \
    libxvidcore-dev \
    libzmq3-dev \
    v4l-utils \
    x11-apps \
    x264 \
    yasm \
  # Torch
    libomp-dev \
    libsox-dev \
    libsox-fmt-all \
    libsphinxbase-dev \
    sphinxbase-utils \
  # FFMpeg (source install, do not install packages: libavcodec-dev libavformat-dev libavresample-dev libavutil-dev libswscale-dev)
    libass-dev \
    libc6 \
    libc6-dev \
    libnuma1 \
    libnuma-dev \
    libopus-dev \
    libtool \
    libvpx-dev \
  && apt-get clean


ENV CTPO_CLANG_VERSION=16
RUN apt-get install -y lsb-release software-properties-common gnupg \
  && mkdir /tmp/clang \
  && cd /tmp/clang \
  && wget https://apt.llvm.org/llvm.sh \
  && chmod +x llvm.sh \
  && ./llvm.sh ${CTPO_CLANG_VERSION} \
  && cd / \
  && rm -rf /tmp/clang
RUN which clang-${CTPO_CLANG_VERSION} \
  && which clang++-${CTPO_CLANG_VERSION} \
  && clang-${CTPO_CLANG_VERSION} --version \
  && clang++-${CTPO_CLANG_VERSION} --version

ENV CTPO_BUILD=GPU
RUN touch /tmp/.${CTPO_BUILD}_build

# - Avoid "RPC failed; curl 56 GnuTLS" issue on some pulls, while keeping the system installed git
# - Some tools expect a "python" binary
# - Prepare ldconfig
RUN git config --global http.postBuffer 1048576000 \
  && mkdir -p /usr/local/bin && ln -s $(which python3) /usr/local/bin/python \
  && mkdir -p /usr/local/lib && sh -c 'echo "/usr/local/lib" >> /etc/ld.so.conf.d/usrlocallib.conf' && ldconfig

ENV LD_LIBRARY_PATH /usr/local/lib:$LD_LIBRARY_PATH

# Misc GPU fixes
RUN cd /usr/local \
  && if [ -e cuda ]; then if [ ! -e nvidia ]; then ln -s cuda nvidia; fi; fi \
  && tmp="/usr/local/cuda/extras/CUPTI/lib64" \
  && if [ -d $tmp ]; then \
       echo $tmp >> /etc/ld.so.conf.d/nvidia-cupti.conf; \
       ldconfig; \
       echo "***** CUPTI added to LD path"; \
     fi

# Install Python3.11.5
RUN mkdir /tmp/python3115 \
  && cd /tmp/python3115 \
  && curl -O https://www.python.org/ftp/python/3.11.5/Python-3.11.5.tgz \
  && tar zxvf /tmp/python3115/Python-3.11.5.tgz \
  && cd Python-3.11.5 \
  && ./configure \
  && make \
#  && make test \
  && make install \
  && cd \
  && rm  -rf /tmp/python3115

# Setup pip
RUN wget -q -O /tmp/get-pip.py --no-check-certificate https://bootstrap.pypa.io/get-pip.py \
  && python3 /tmp/get-pip.py \
  && pip3 install -U pip \
  && rm /tmp/get-pip.py

# Install Python tools (for buiding)
RUN pip3 install -U \
  # Base
    cmake \
    future \
    mkl \
    mkl-include \
    mock \
    numpy \
    opt_einsum \
    packaging \
    pyyaml \
    requests \
    setuptools \
    six \
    wheel \
  # OpenCV
    lxml \
    Pillow \
  # Torch
    cffi \
    typing \
    ninja \
  # Extra
    scikit-image \
    scikit-learn \
    autovizwidget \
    ipython \
    jupyter \
    matplotlib \
    moviepy \
    notebook \
    pandas \
  && pip3 install -U keras_preprocessing --no-deps \
  && rm -rf /root/.cache/pip

#### Debug temp ####
#RUN apt install mc -y


### TensorFlow TensorRT ###

ENV version="10.0.0.0"
ENV arch="x86_64"
ENV cuda="cuda-12.4"

COPY nv-tensorrt-local-repo-ubuntu2204-10.0.0-cuda-12.4_1.0-1_amd64.deb TensorRT-${version}.Linux.${arch}-gnu.${cuda}.deb

RUN dpkg -i TensorRT-${version}.Linux.${arch}-gnu.${cuda}.deb \
#  && cp /var/nv-tensorrt-local-repo-ubuntu2204-10.0.0-cuda-12.4/nv-tensorrt-local*keyring.gpg /usr/share/keyrings/ \
#  && export LD_LIBRARY_PATH=<TensorRT-${version}/lib>:$LD_LIBRARY_PATH \
#  && cd TensorRT-${version}/python \
#  && python3 -m pip install tensorrt-*-cp311-none-linux_x86_64.whl \
#  && cd TensorRT-${version}/onnx_graphsurgeon \
#  && python3 -m pip install onnx_graphsurgeon-0.5.0-py2.py3-none-any.whl \
  && pip install tensorflow[and-cuda] \
  && python3 -m pip install --upgrade pip \
  && python3 -m pip install wheel \
  && python3 -m pip install --pre --upgrade tensorrt \
  && rm TensorRT-${version}.Linux.${arch}-gnu.${cuda}.deb

##### FFMPEG #####

ENV CTPO_FFMPEG_VERSION=5.1.4
ENV CTPO_FFMPEG_NVCODEC=11.1.5.2
ENV CTPO_FFMPEG_NONFREE=""
RUN mkdir -p /usr/local/src/builder \
  && cd /usr/local/src \
  && wget -q https://github.com/FFmpeg/nv-codec-headers/archive/refs/tags/n${CTPO_FFMPEG_NVCODEC}.tar.gz -O - | tar --strip-components=1 -xz -C /usr/local/src/builder \
  && cd /usr/local/src/builder \
  && make install \
  && rm -rf /usr/local/src/builder
RUN mkdir -p /usr/local/src/builder \
  && cd /usr/local/src \
  && wget -q https://ffmpeg.org/releases/ffmpeg-${CTPO_FFMPEG_VERSION}.tar.gz -O - | tar --strip-components=1 -xz -C /usr/local/src/builder \
  && cd /usr/local/src/builder \
  && time ./configure --enable-cuda --enable-cuvid --enable-nvdec --enable-nvenc ${CTPO_FFMPEG_NONFREE} --extra-cflags="-I/usr/local/cuda/include/ -fPIC" --extra-ldflags="-L/usr/local/cuda/lib64/ -Wl,-Bsymbolic" --enable-shared --disable-static --enable-gpl --enable-libv4l2 --enable-libvorbis --enable-libvpx --enable-libwebp --enable-libxvid --enable-libopus --enable-pic --enable-libass --enable-libx264 --enable-libx265 | tee /tmp/ffmpeg_config.txt \
  && make -j${CTPO_NUMPROC} install \
  && ldconfig \
  && rm -rf /usr/local/src/builder
# From https://docs.nvidia.com/video-technologies/video-codec-sdk/ffmpeg-with-nvidia-gpu/#basic-testing
# GPU Testing: ffmpeg -y -vsync 0 -hwaccel cuda -hwaccel_output_format cuda -i input.mp4 -c:a copy -c:v h264_nvenc -b:v 5M output.mp4
# GPU Testing: ffmpeg -y -vsync 0 -hwaccel cuvid -c:v h264_cuvid -i in.mp4 -c:v hevc_nvenc out.mkv

RUN echo "${CTPO_FFMPEG_VERSION}" > /tmp/.ffmpeg_built


#### OpenCV ####
RUN pip3 install opencv-python

##### Magma (for PyTorch GPU only) #####

ENV CTPO_MAGMA=2.7.2
ENV CTPO_MAGMA_ARCH="Pascal Volta Turing Ampere Hopper"
RUN mkdir -p /usr/local/src/builder/build; \
    cd /usr/local/src; \
    wget -q http://icl.utk.edu/projectsfiles/magma/downloads/magma-${CTPO_MAGMA}.tar.gz -O - | tar --strip-components=1 -xz -C /usr/local/src/builder; \
    cd /usr/local/src/builder/build; \
    time cmake -DGPU_TARGET="${CTPO_MAGMA_ARCH}" ..; \
    time make -j${CTPO_NUMPROC} install; \
    rm -rf /usr/local/src/magma


#### Torch ####
RUN pip3 install torch torchvision torchaudio


##### Final steps #####

# Add lab & dataframe display widget
RUN apt-get install -y python3-venv \
  && pip3 install -U \
      jupyterlab
# https://jupyterlab.readthedocs.io/en/stable/getting_started/changelog.html
# with v4 extensions are installed using the extension manager, see https://jupyterlab.readthedocs.io/en/stable/user/extensions.html#id11

RUN touch /.within_container
COPY tools/withincontainer_checker.sh /tmp/withincontainer_checker.sh
RUN chmod +x /tmp/withincontainer_checker.sh \
  && /tmp/withincontainer_checker.sh

# Setting up working directory
RUN mkdir /iti
WORKDIR /iti

# Uncomment as needed
#ENV NVIDIA_VISIBLE_DEVICES all
#ENV NVIDIA_DRIVER_CAPABILITIES video,compute,utility

CMD /bin/bash

# Copyright (c) 2022 PaddlePaddle Authors. All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.


FROM nvidia/cuda:11.2.2-cudnn8-devel-ubuntu20.04

ARG http_proxy
ARG https_proxy

#Install the build dependencies
RUN apt-get update && DEBIAN_FRONTEND=noninteractive TZ=Asia/Shanghai apt-get install -y --no-install-recommends curl wget vim git patchelf python3-dev python3-pip openssl \
    python3-setuptools build-essential libgl1-mesa-glx libglib2.0-dev ca-certificates libb64-dev datacenter-gpu-manager \
    libssl-dev zlib1g-dev rapidjson-dev libboost-dev libre2-dev librdmacm-dev libnuma-dev libarchive-dev unzip && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

RUN python3 -m pip install --upgrade pip && python3 -m pip install redis

# install cmake
WORKDIR /home
RUN wget -q https://github.com/Kitware/CMake/releases/download/v3.18.6/cmake-3.18.6-Linux-x86_64.tar.gz && tar -zxvf cmake-3.18.6-Linux-x86_64.tar.gz
ENV PATH=/home/cmake-3.18.6-Linux-x86_64/bin:$PATH


#install triton
ENV TAG=r21.10
RUN git clone https://github.com/triton-inference-server/server.git -b $TAG && \
   cd server && \
   mkdir -p build/tritonserver/install && \
   python3 build.py \
     --build-dir `pwd`/build \
     --no-container-build \
     --backend=ensemble \
     --enable-gpu \
     --endpoint=grpc \
     --endpoint=http \
     --enable-stats \
     --enable-tracing \
     --enable-logging \
     --enable-metrics \
     --enable-gpu-metrics \
     --enable-cpu-metrics \
     --enable-nvtx \
     --cmake-dir `pwd` \
     --repo-tag=common:$TAG \
     --repo-tag=core:$TAG \
     --repo-tag=backend:$TAG \
     --repo-tag=thirdparty:$TAG \
     --backend=python:$TAG

COPY python/dist/*.whl /opt/fastdeploy/
RUN python3 -m pip install  /opt/fastdeploy/*.whl \
    && rm -rf /opt/fastdeploy/*.whl


# compile triton-inference-server/server，copy tritonserver and python backend into image
# triton server
RUN mkdir -p /opt/tritonserver && cp -r /home/server/build/tritonserver/install/* /opt/tritonserver
# python backend
RUN mkdir -p /opt/tritonserver/backends/python && cp -r /home/server/build/python/install/backends/python /opt/tritonserver/backends/

# copy compiled fastdeploy backend into image
COPY serving/build/libtriton_fastdeploy.so /opt/tritonserver/backends/fastdeploy/

# rename tritonserver to fastdeployserver
RUN mv /opt/tritonserver/bin/tritonserver /opt/tritonserver/bin/fastdeployserver

# copy compiled fastdeploy_install into image
COPY build/fastdeploy_install/* /opt/fastdeploy/

# Set environment variable
ENV LD_LIBRARY_PATH="/opt/fastdeploy/lib:/opt/fastdeploy/third_libs/install/onnxruntime/lib:/opt/fastdeploy/third_libs/install/paddle2onnx/lib:/opt/fastdeploy/third_libs/install/paddle_inference/paddle/lib:/opt/fastdeploy/third_libs/install/openvino/runtime/lib/:/opt/fastdeploy/third_libs/install/tensorrt/lib/:/opt/fastdeploy/third_libs/install/opencv/lib64/:$LD_LIBRARY_PATH"
ENV PATH="/opt/tritonserver/bin:$PATH"
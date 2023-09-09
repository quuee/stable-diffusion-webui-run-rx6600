## 安装rocm(ubuntu)

### 查看linux版本

[Installation Prerequisites (Linux) — ROCm 5.6.1 Documentation Home](https://rocm.docs.amd.com/en/latest/deploy/linux/prerequisites.html)

查看linux发行版本

```shell
uname -m && cat /etc/*release
```

内核信息

```shell
uname -srmv
```
查看groups
```shell
groups
```
将自己添加到render video
```shell
sudo usermod -a -G render,video $LOGNAME
```
若要默认将所有未来用户添加到 render video
```shell
echo 'ADD_EXTRA_GROUPS=1' | sudo tee -a /etc/adduser.conf
echo 'EXTRA_GROUPS=video' | sudo tee -a /etc/adduser.conf
echo 'EXTRA_GROUPS=render' | sudo tee -a /etc/adduser.conf
```

### 脚本安装 amdgpu
22.04
```shell
sudo apt update
wget https://repo.radeon.com/amdgpu-install/5.6.1/ubuntu/jammy/amdgpu-install_5.6.50601-1_all.deb
sudo apt install ./amdgpu-install_5.6.50601-1_all.deb
```
### use case
```shell
sudo amdgpu-install --list-usecase
```
example
``` shell
If --usecase option is not present, the default selection is "graphics,opencl,hip"

Available use cases:
rocm(for users and developers requiring full ROCm stack)
- OpenCL (ROCr/KFD based) runtime
- HIP runtimes
- Machine learning framework
- All ROCm libraries and applications
- ROCm Compiler and device libraries
- ROCr runtime and thunk
lrt(for users of applications requiring ROCm runtime)
- ROCm Compiler and device libraries
- ROCr runtime and thunk
opencl(for users of applications requiring OpenCL on Vega or
later products)
- ROCr based OpenCL
- ROCm Language runtime

openclsdk (for application developers requiring ROCr based OpenCL)
- ROCr based OpenCL
- ROCm Language runtime
- development and SDK files for ROCr based OpenCL

hip(for users of HIP runtime on AMD products)
- HIP runtimes
hiplibsdk (for application developers requiring HIP on AMD products)
- HIP runtimes
- ROCm math libraries
- HIP development libraries
```

```shell
sudo amdgpu-install --usecase=rocm
```
或者
```shell
sudo amdgpu-install --usecase=hiplibsdk,rocm
```

### 多版本rocm安装

### 查看版本rocm
```shell
apt show rocm-libs -a
rocminfo  
lsmod | grep amdgpu
apt list --installed | grep amdgpu-dkms
```

## Running inside Docker
[amd rocm doc](https://rocm.docs.amd.com/en/latest/deploy/linux/quick_start.html)
官网说docker运行只需安装amdgpu-dkms

```
/dev/kfd : 所有GPU共享主机计算接口
```
```
/dev/dri/renderD<node> : 每个设备的直接渲染接口 （DRI） 设备 显卡。<node> 是系统中每张卡的数字，从 128 开始
```
```dockerfile
FROM ubuntu:20.04

# Register the ROCM package repository, and install rocm-dev package
ARG ROCM_VERSION=5.5.3
ARG AMDGPU_VERSION=5.5.3

RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends ca-certificates curl libnuma-dev gnupg \
  && curl -sL https://repo.radeon.com/rocm/rocm.gpg.key | apt-key add - \
  && printf "deb [arch=amd64] https://repo.radeon.com/rocm/apt/$ROCM_VERSION/ ubuntu main" | tee /etc/apt/sources.list.d/rocm.list \
  && printf "deb [arch=amd64] https://repo.radeon.com/amdgpu/$AMDGPU_VERSION/ubuntu focal main" | tee /etc/apt/sources.list.d/amdgpu.list \
  && apt-get update \
  && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
  sudo \
  libelf1 \
  kmod \
  file \
  python3 \
  python3-pip \
  rocm-dev \
  rocm-libs \
  git \
  build-essential && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/*

ARG user=x

COPY sudo-nopasswd /etc/sudoers.d/sudo-nopasswd

RUN useradd --create-home -G sudo,video,render --shell /bin/bash ${user}

USER ${user}
WORKDIR /home/${user}
ENV PATH "${PATH}:/opt/rocm/bin"

RUN pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/rocm5.4.2

RUN git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui
RUN cd stable-diffusion-webui


# Default to a login shell
CMD ["bash", "-l"]
```
### 打包镜像
build
```shell
docker build -f Dockerfile-ubuntu2004 -t rocm/pytorch:v1.0 .
```
### 启动容器
```shell
docker run -it --device=/dev/kfd --device=/dev/dri --security-opt seccomp=unconfined --group-add video --network=host --ipc=host --cap-add=SYS_PTRACE --shm-size 8G rocm/pytorch:v1.0
```

### 进入容器
```shell
# 验证
python3 -c 'import torch; print(torch.__version__);print(torch.cuda.is_available())'
False

# 查看是否安装正确
/opt/rocm/bin/rocminfo

ROCk module is loaded
Unable to open /dev/kfd read-write: Permission denied
Failed to get user name to check for video group membership
```
验证都失败了。



## 换官方镜像rocm/pytoch
官方镜像看似选择很多

```shell

docker pull rocm/pytorch:rocm5.6_ubuntu20.04_py3.8_pytorch_2.0.1

docker run -it --network=host --device=/dev/kfd --device=/dev/dri --group-add=video --ipc=host --cap-add=SYS_PTRACE --security-opt seccomp=unconfined -v $HOME/docker-container/stable-diffusion:/dockerx rocm/pytorch:rocm5.6_ubuntu20.04_py3.8_pytorch_2.0.1
```
检验torch rocminfo成功

始终启动不了stable-diffusion，python版本切换不了，始终保持3.8.16。  
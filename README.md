## 安装rocm(ubuntu)
[docker环境](./ubunt-rx6600-rocm.md)
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

## 本地安装stable-diffusion
[本地环境](./stable-diffusion-localenv-guide.md)
rocm环境前面已装
Xformers仅适用N卡，有的项目说不稳定

### 本地环境需要安装python一些库
根据stable-difussion-webui提示
```shell
sudo apt install python3 python3-venv libgl1 libglib2.0-0
```
### 创建激活python虚拟环境 
```shell
python3 -m venv stable-diffusion-venv # 因为已在~/Documents/GithubProjects/目录下创建，或者在stable-diffusion-webui项目里创建
source ~/Documents/GithubProjects/stable-diffusion-venv/bin/activate # 激活虚拟环境
git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git
cd stable-diffusion-webui
bash webui.sh
```

### 虚拟路径删除
1、直接删除虚拟路径
2、在虚拟环境中运行 deactivate 退出 返回默认的python环境

### 下载模型
https://civitai.com 科学上网
https://huggingface.co 科学上网
https://rentry.co/sdmodels 科学上网
https://arthub.ai 科学上网
https://www.liblibai.com/ 科学上网


#### Checkpoint/大模型/底模型/主模型
Checkpoint模型是SD能够绘图的基础模型，因此被称为大模型、底模型或者主模型，WebUI上就叫它Stable Diffusion模型。安装完SD软件后，必须搭配主模型才能使用。不同的主模型，其画风和擅长的领域会有侧重。

checkpoint模型包含生成图像所需的一切，不需要额外的文件。但是它们体积很大，通常为2G-7G。

常见文件模式:尾缀ckpt、safetensors（如果都有提供的话建议下载safetensors，下同）

存放路径： xxx/xxx/stable-diffusion-webui/models  
如：/home/x/Documents/GithubProjects/stable-diffusion-webui/models/Stable-diffusion

#### 模型存放位置
stable-diffusion-webui/models目录下有不同类型的模型目录，根据模型存放

### 启动后报错和办法
```python
# 错误信息
OSError: Can't load tokenizer for './models/clip-vit-large-patch14'. If you were trying to load it from 'https://huggingface.co/models', make sure you don't have a local directory with the same name. Otherwise, make sure './models/clip-vit-large-patch14' is the correct path to a directory containing all relevant files for a CLIPTokenizer tokenizer
```
#### github issues 给出的办法(没用)
```
https://github.com/AUTOMATIC1111/stable-diffusion-webui/pull/2945#issuecomment-1285574017

从这下就行 https://huggingface.co/openai/clip-vit-large-patch14 ，下载的都放到./models/clip-vit-large-patch14 
```

参考stable-diffusion/environment.yaml 和stable-diffusion教程，应该是放在项目根目录下，所以放在stable-diffusion-webui/下，启动成功。

### 试跑
```shell
# 查看amd显卡状态
sudo apt install radeontop
radeontop
```

```python
# 溢出了
torch.cuda.OutOfMemoryError: HIP out of memory. Tried to allocate 1.12 GiB (GPU 0; 7.98 GiB total capacity; 6.84 GiB already allocated; 794.00 MiB free; 7.12 GiB reserved in total by PyTorch) If reserved memory is >> allocated memory try setting max_split_size_mb to avoid fragmentation.
```

### 尝试小显存试跑1 (黑屏了)
```shell
# 终端手动启动  
PYTORCH_HIP_ALLOC_CONF=garbage_collection_threshold:0.9,max_split_size_mb:512 python3 launch.py --precision full --no-half --opt-sub-quad-attention
```
### 尝试在webui-user.sh文件里里添加参数
```shell
export COMMANDLINE_ARGS="--medvram --opt-split-attention"
```
```shell
source ~/Documents/GithubProjects/stable-diffusion-venv/bin/activate
bash webui.sh
````
success,done!

[启动时优化参数解释](https://github.com/AUTOMATIC1111/stable-diffusion-webui/wiki/Optimizations)
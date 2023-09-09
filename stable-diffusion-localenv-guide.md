## 本地安装stable-diffusion
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
https://www.liblibai.com/ 

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
bash webui.sh
````
success,done!

[启动时优化参数解释](https://github.com/AUTOMATIC1111/stable-diffusion-webui/wiki/Optimizations)
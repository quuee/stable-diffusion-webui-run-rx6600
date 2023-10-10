# vits-fast-fine-tuning
## 环境安装
(确保您已安装 、CMake & C/C++ 编译器、ffmpeg，按自己需要根据LOCAL.md安装)
sudo apt install ffmpeg #简易安装
ffmpeg -v
conda create -n vits-py38 python=3.8
conda activate vits-py38

(5.6 5.7都可以)
pip3 install --pre torch torchvision torchaudio --index-url https://download.pytorch.org/whl/nightly/rocm5.7

删除requirements.txt里面的 numpy torch torchvision torchaudio
或替换如下
```txt
Cython==0.29.21
librosa==0.9.2
matplotlib==3.3.1
scikit-learn==1.0.2
scipy
tensorboard
unidecode
pyopenjtalk
jamo
pypinyin
jieba
protobuf
cn2an
inflect
eng_to_ipa
ko_pron
indic_transliteration==2.3.37
num_thai==0.0.5
opencc==1.1.1
demucs
openai-whisper
gradio
```
pip install -r requirements.txt -i https://pypi.mirrors.ustc.edu.cn/simple/

## 构建单调对齐
cd monotonic_align
mkdir monotonic_align
python setup.py build_ext --inplace

## 下载辅助数据
wget https://huggingface.co/datasets/Plachta/sampled_audio4ft/resolve/main/sampled_audio4ft_v2.zip
unzip sampled_audio4ft_v2.zip
## 下载预训练模型
wget https://huggingface.co/spaces/Plachta/VITS-Umamusume-voice-synthesizer/resolve/main/pretrained_models/D_trilingual.pth -O ./pretrained_models/D_0.pth
wget https://huggingface.co/spaces/Plachta/VITS-Umamusume-voice-synthesizer/resolve/main/pretrained_models/G_trilingual.pth -O ./pretrained_models/G_0.pth
wget https://huggingface.co/spaces/Plachta/VITS-Umamusume-voice-synthesizer/resolve/main/configs/uma_trilingual.json -O ./configs/finetune_speaker.json

## 创建文件夹custom_character_voice

### 处理音频 音频标注（短音频）
export HSA_OVERRIDE_GFX_VERSION=10.3.0
python3 scripts/short_audio_transcribe.py --languages "CJE" --whisper_size medium
得到short_character_anno.txt  

### 添加辅助数据
//没有辅助数据
python preprocess_v2.py --languages "CJE"
//有辅助数据
python preprocess_v2.py --add_auxiliary_data True --languages "CJE"

## 模型训练 100步 200步 300步
python finetune_speaker_v2.py -m ./OUTPUT_MODEL --max_epochs "100" --drop_speaker_embed True  

## 使用模型
复制./OUTPUT_MODEL/config.json文件至项目根目录./VITS-fast-fine-tuning下，然后将其更名为finetune_speaker.json
python VC_inference.py --model_dir ./OUTPUT_MODEL/G_latest.pth --share True
浏览器自动弹出http://127.0.0.1:7860/ 模型推理页面
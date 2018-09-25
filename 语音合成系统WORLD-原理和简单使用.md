---
title: 语音合成系统WORLD-原理和简单使用
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---

>最近在做语音合成相关的一个东西，其中后期需要做一个声音转换系统，但是真正的声音转换系统还挺复杂，因为我们的目的是希望能够将一个声音完全地变为另一个已知的

WORLD通过获取三个语音信号相关的参数信息来合成原始语音，这三个参数信息分别是：基频F0、频谱包络、非周期信号参数（英文分别为：Fundamental Frequency、spectral envelope、aperiodic parameter）

**参数解释：**
* 基频F0：对于一个由振动而发出的声音信号，这组信号可以看做是由许多组频率不同的正弦波组成，其中频率最低的正弦波即为基频，其他的为泛音。
	* 首先，通过一个低通滤波器对原始信号进行滤波
	* 之后，对滤波后的信号进行评估。由于滤波后的信号应该恰好是一个正弦波，所以，如下图所示，1、2、3、4四个波段的长度应该恰好都是一个周期长度，所以通过计算这四个周期的标准差，就可以评估此正弦波的正确与否。（周期的倒数即频率）
	* 选取标准差最小的作为最终的基频
* 频谱包络：将不同频率的振幅最高点通过平滑的曲线连接起来就是频谱包络。如下图所示。此方法有多种，在求解梅尔倒谱系数时，用的是倒谱法；此工具箱中使用的是CheapTrick算法
![enter description here](http://osiy4s0ad.bkt.clouddn.com/blog/1537845024155.png)
* 非周期信号参数：一般的语音都是有周期信号和非周期信号组成，所以，除了以上获取周期信号的参数，我们还需要得到其中的非周期信号参数，才能完美的合成原始信号。


----------


**WORLD简单使用：**

``` python
此项目主要是用matlab和c，这里的python使用的是调用c接口
"""

import pyworld as pw
from scipy.io import wavfile
import matplotlib.pyplot as plt
import numpy as np
import os
import soundfile as sf

base_dir = "E:\Python_Workspace\\tensorflow-exercises"
# 读取文件
WAV_FILE = os.path.join(base_dir,'files\LJ001-0001.wav')

# 提取语音特征
x, fs = sf.read(WAV_FILE)

# f0 : ndarray
#     F0 contour. 基频等高线
# sp : ndarray
#     Spectral envelope. 频谱包络
# ap : ndarray
#     Aperiodicity. 非周期性
f0, sp, ap = pw.wav2world(x, fs)    # use default options


#分布提取参数
#使用DIO算法计算音频的基频F0

_f0, t = pw.dio( x, fs, f0_floor= 50.0, f0_ceil= 600.0, channels_in_octave= 2, frame_period=pw.default_frame_period)

#使用CheapTrick算法计算音频的频谱包络

_sp = pw.cheaptrick( x, _f0, t, fs)

#计算aperiodic参数

_ap = pw.d4c( x, _f0, t, fs)

#基于以上参数合成音频

_y = pw.synthesize(_f0, _sp, _ap, fs, pw.default_frame_period)

#写入音频文件

sf. write( 'test/y_without_f0_refinement.wav', _y, fs)


plt.plot(f0)
plt.show()
plt.imshow(np.log(sp), cmap='gray')
plt.show()
plt.imshow(ap, cmap='gray')

# 合成原始语音
synthesized = pw.synthesize(f0, sp, ap, fs, pw.default_frame_period)

# 1.输出原始语音
sf.write('test/synthesized.wav', synthesized, fs)

# 2.变高频-更类似女性
high_freq = pw.synthesize(f0*2.0, sp, ap, fs)  # 周波数を2倍にすると1オクターブ上がる

sf.write('test/high_freq.wav', high_freq, fs)

# 3.直接修改基频，变为机器人发声
robot_like_f0 = np.ones_like(f0)*100  # 100は適当な数字
robot_like = pw.synthesize(robot_like_f0, sp, ap, fs)

sf.write('test/robot_like.wav', robot_like, fs)

# 4.提高基频，同时频谱包络后移？更温柔的女性？
female_like_sp = np.zeros_like(sp)
for f in range(female_like_sp.shape[1]):
    female_like_sp[:, f] = sp[:, int(f/1.2)]
female_like = pw.synthesize(f0*2, female_like_sp, ap, fs)

sf.write('test/female_like.wav', female_like, fs)

# 5.转换基频（不能直接转换）
x2, fs2 = sf.read('utterance/A2_0.wav')
f02, sp2, ap2 = pw.wav2world(x2, fs2)
f02 = f02[:len(f0)]
print(len(f0),len(f02))
other_like = pw.synthesize(f02, sp, ap, fs)

sf.write('test/other_like.wav', other_like, fs)
```

区分男女说话主要是通过音高（基频）和音色（频谱包络-频谱最大幅度的连接线）
音高：http://ibillxia.github.io/blog/2013/05/16/audio-signal-processing-time-domain-pitch-python-realization/
音色：http://ibillxia.github.io/blog/2013/05/18/audio-signal-processing-time-domain-timbre-python-realization/

文档：直接参考C语言文档或者查看github源码及其一个demo
https://qiita.com/ohtaman/items/84426cee09c2ba4abc22

原理参考：http://www.elecfans.com/d/625428.html
---
title: 梅尔倒谱系数实现-MFCC
tags: MFCC,FFT
grammar_cjkRuby: true
---


``` stylus
""" 
@author: zoutai
@file: mymfcc.py 
@time: 2018/03/26 
@description:
"""
from matplotlib.colors import BoundaryNorm
import librosa
import librosa.display
import numpy
import scipy.io.wavfile
from scipy.fftpack import dct
import matplotlib.pyplot as plt
import numpy as np


# 第一步-读取音频，画出时域图（采样率-幅度）
sample_rate, signal = scipy.io.wavfile.read('OSR_us_000_0010_8k.wav')  # File assumed to be in the same directory
signal = signal[0:int(3.5 * sample_rate)]
# plot the wave
time = np.arange(0,len(signal))*(1.0 / sample_rate)
# plt.plot(time,signal)
plt.xlabel("Time(s)")
plt.ylabel("Amplitude")
plt.title("Signal in the Time Domain ")
plt.grid('on')#标尺，on：有，off:无。




# 第二部-预加重
# 消除高频信号。因为高频信号往往都是相似的，
# 通过前后时间相减，就可以近乎抹去高频信号，留下低频信号。
# 原理：y(t)=x(t)−αx(t−1)

pre_emphasis = 0.97
emphasized_signal = numpy.append(signal[0], signal[1:] - pre_emphasis * signal[:-1])



time = np.arange(0,len(emphasized_signal))*(1.0 / sample_rate)
# plt.plot(time,emphasized_signal)
# plt.xlabel("Time(s)")
# plt.ylabel("Amplitude")
# plt.title("Signal in the Time Domain after Pre-Emphasis")
# plt.grid('on')#标尺，on：有，off:无。





# 第三步、取帧，用帧表示
frame_size = 0.025 # 帧长
frame_stride = 0.01 # 步长

# frame_length-一帧对应的采样数, frame_step-一个步长对应的采样数
frame_length, frame_step = frame_size * sample_rate, frame_stride * sample_rate  # Convert from seconds to samples
signal_length = len(emphasized_signal) # 总的采样数

frame_length = int(round(frame_length))
frame_step = int(round(frame_step))

# 总帧数
num_frames = int(numpy.ceil(float(numpy.abs(signal_length - frame_length)) / frame_step))  # Make sure that we have at least 1 frame

pad_signal_length = num_frames * frame_step + frame_length
z = numpy.zeros((pad_signal_length - signal_length))
pad_signal = numpy.append(emphasized_signal, z) # Pad Signal to make sure that all frames have equal number of samples without truncating any samples from the original signal

# Construct an array by repeating A（200） the number of times given by reps（348）.
# 这个写法太妙了。目的：用矩阵来表示帧的次数，348*200，348-总的帧数，200-每一帧的采样数
# 第一帧采样为0、1、2...200;第二帧为80、81、81...280..依次类推
indices = numpy.tile(numpy.arange(0, frame_length), (num_frames, 1)) + numpy.tile(numpy.arange(0, num_frames * frame_step, frame_step), (frame_length, 1)).T
frames = pad_signal[indices.astype(numpy.int32, copy=False)] # Copy of the array indices
# frame：348*200，横坐标348为帧数，即时间；纵坐标200为一帧的200毫秒时间，内部数值代表信号幅度

# plt.matshow(frames, cmap='hot')
# plt.colorbar()
# plt.figure()
# plt.pcolormesh(frames)



# 第四步、加汉明窗
# 傅里叶变换默认操作的时间段内前后端点是连续的，即整个时间段刚好是一个周期，
# 但是，显示却不是这样的。所以，当这种情况出现时，仍然采用FFT操作时，
# 就会将单一频率周期信号认作成多个不同的频率信号的叠加，而不是原始频率，这样就差生了频谱泄漏问题

frames *= numpy.hamming(frame_length) # 相乘，和卷积类似
# # frames *= 0.54 - 0.46 * numpy.cos((2 * numpy.pi * n) / (frame_length - 1))  # Explicit Implementation **

# plt.pcolormesh(frames)



# 第五步-傅里叶变换频谱和能量谱

# _raw_fft扫窗重叠，将348*200，扩展成348*512
NFFT = 512
mag_frames = numpy.absolute(numpy.fft.rfft(frames, NFFT))  # Magnitude of the FFT
pow_frames = ((1.0 / NFFT) * ((mag_frames) ** 2))  # Power Spectrum


# plt.pcolormesh(mag_frames)
#
# plt.pcolormesh(pow_frames)





# 第六步，Filter Banks滤波器组
# 公式：m=2595*log10(1+f/700)；f=700(10^(m/2595)−1)
nfilt = 40 #窗的数目
low_freq_mel = 0
high_freq_mel = (2595 * numpy.log10(1 + (sample_rate / 2) / 700))  # Convert Hz to Mel
mel_points = numpy.linspace(low_freq_mel, high_freq_mel, nfilt + 2)  # Equally spaced in Mel scale
hz_points = (700 * (10**(mel_points / 2595) - 1))  # Convert Mel to Hz
bin = numpy.floor((NFFT + 1) * hz_points / sample_rate)

fbank = numpy.zeros((nfilt, int(numpy.floor(NFFT / 2 + 1))))
for m in range(1, nfilt + 1):
    f_m_minus = int(bin[m - 1])   # left
    f_m = int(bin[m])             # center
    f_m_plus = int(bin[m + 1])    # right

    for k in range(f_m_minus, f_m):
        fbank[m - 1, k] = (k - bin[m - 1]) / (bin[m] - bin[m - 1])
    for k in range(f_m, f_m_plus):
        fbank[m - 1, k] = (bin[m + 1] - k) / (bin[m + 1] - bin[m])
filter_banks = numpy.dot(pow_frames, fbank.T)
filter_banks = numpy.where(filter_banks == 0, numpy.finfo(float).eps, filter_banks)  # Numerical Stability
filter_banks = 20 * numpy.log10(filter_banks)  # dB;348*26

# plt.subplot(111)
# plt.pcolormesh(filter_banks.T)
# plt.grid('on')
# plt.ylabel('Frequency [Hz]')
# plt.xlabel('Time [sec]')
# plt.show()


#
# 第七步，梅尔频谱倒谱系数-MFCCs
num_ceps = 12 #取12个系数
cep_lifter=22 #倒谱的升个数？？
mfcc = dct(filter_banks, type=2, axis=1, norm='ortho')[:, 1 : (num_ceps + 1)] # Keep 2-13
(nframes, ncoeff) = mfcc.shape
n = numpy.arange(ncoeff)
lift = 1 + (cep_lifter / 2) * numpy.sin(numpy.pi * n / cep_lifter)
mfcc *= lift  #*

# plt.pcolormesh(mfcc.T)
# plt.ylabel('Frequency [Hz]')
# plt.xlabel('Time [sec]')


# 第八步，均值化优化
# to balance the spectrum and improve the Signal-to-Noise (SNR), we can simply subtract the mean of each coefficient from all frames.

filter_banks -= (numpy.mean(filter_banks, axis=0) + 1e-8)
mfcc -= (numpy.mean(mfcc, axis=0) + 1e-8)

# plt.subplot(111)
# plt.pcolormesh(mfcc.T)
# plt.ylabel('Frequency [Hz]')
# plt.xlabel('Time [sec]')
# plt.show()



# 直接频谱分析
# plot the wave
# plt.specgram(signal,Fs = sample_rate, scale_by_freq = True, sides = 'default')
# plt.ylabel('Frequency(Hz)')
# plt.xlabel('Time(s)')
# plt.show()


# 经过FFT计算后输出的点数是257(N/2+1)，其含义表示的是从0(Hz)到采样率/2(Hz)的N/2+1点频率的成分
plt.figure(figsize=(10, 4))
mfccs = librosa.feature.melspectrogram(signal,sr=8000,n_fft=512,n_mels=40)
librosa.display.specshow(mfccs, x_axis='time')
plt.colorbar()
plt.title('MFCC')
plt.tight_layout()
plt.show()
```



[更加完美的讲解-博客][1]
[librosa demo][2]
[博客参考][3]
[什么是窗函数][4]
[什么是频谱泄露][5]
[傅里叶变换浅显易懂][6]
[语音处理相关方法][7]


  [1]: https://zhuanlan.zhihu.com/p/27416870
  [2]: http://nbviewer.jupyter.org/github/librosa/librosa/blob/master/examples/LibROSA%20demo.ipynb
  [3]: http://haythamfayek.com/2016/04/21/speech-processing-for-machine-learning.html
  [4]: https://zhuanlan.zhihu.com/p/24318554
  [5]: https://mp.weixin.qq.com/s?__biz=MzI5NTM0MTQwNA==&mid=2247484164&idx=1&sn=fdaf2164306a9ca4166c2aa8713cacc5&scene=21#wechat_redirect
  [6]: http://blog.jobbole.com/70549/
  [7]: https://www.cnblogs.com/xingshansi/p/6799994.html

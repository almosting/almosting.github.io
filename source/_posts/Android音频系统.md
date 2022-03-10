---
title: Android 音频系统
date: 2022-03-10 10:37:51
categories: Android
tags: 系统
---

- App
音频应用软件

- Framework
MediaPlayer 和 MediaRecorder，以及 AudioTrack、AudioRecorder、AudioMannager、AudioService 以及 AudioSystem 。

- Libraries
frameorks/av/media/libmedia、libaudioflinger、libmediaplayerservice

- HAL
AudioFlinger、AudioPolicyService。

{% asset_img 音频系统框架全图.png 音频系统框架全图 %}
- AudioPolicyService：APS 是音频框架的服务，main_audioserver 启动，会创建 AudioCommandThread 和 AudioPolicyClient、AudioPolicyManager。它主要由 AudioSystem 通过 binder 调用，也可以由 AudioPolicyClient，AudioPolicyManager 直接调用。它的大部分操作都交给 AudioPolicyManager 来做
- AudioPolicyClient：APC 是 AudioPolicyService 的内部类。它用于打开关闭输入输出，设置流音量，传递参数给 hal 层（如 audio_hw.cpp）等；它主要是通过 binder 跨进程调用 AudioFlinger 去完成真正的操作。可以由 AudioManager 通过 mpClientInterface 去调用它。
- AudioPolicyManager：APM 是 AudioPolicyService 的主要工作类，AudioPolicyService 的大部分操作都由他来执行
- AudioFlinger：AF 主要承担音频混合输出，是 Audio 系统的核心，从 AudioTrack 来的数据最终都会在这里处理，并被写入到 Audio 的 HAL。
- DevicesFactoryHalLocal：根据名字加载对应的 hal module。比如传进去 a2dp 相关的名字，会加载到 audio.a2dp.default.so
- DevicesFactoryHalHidl：跨进行加载 hidl hal module。
- DevicesFactoryHalInterface：用于创建子类 DevicesFactoryHalHybrid。
- DevicesFactoryHalHybrid：选择创建 DevicesFactoryHalLocal 或者 DevicesFactoryHalHidl，我这里只创建 DevicesFactoryHalLocal。
- DeviceHalLocal：通过私有成员 audio_hw_device_t *mDev，直接调用 hal 代码，用来设置和获取底层参数，打开和关闭 stream。
- StreamOutHalLocal：通过私有成员 audio_stream_out_t *mStream 直接调用 hal 代码，用于操作流，比如 start、stop、flush、puse 操作；还有调用 write 函数写音频数据到 hal 层。
- AudioStreamOutSink：它其实是 StreamOutHalLocal 的一个 wrapper，它也有 write 函数，不过是通过 StreamOutHalLocal 来操作的。

  附上一张重要的类图：

  {% asset_img audio_architecture.jpg audio_architecture %}

## Audio 服务的启动

{% asset_img audio_service.png audio_service %}

1. 创建 AudioFlinger 和 AudioPolicyService。
2. 解析 Audio Config 文件（audio_policy_configuration.xml），获取支持的音频外设列表及各输入输出通路详细参数。
3. 根据解析得到的外设列表，加载所有的 Audio HAL 库。
4. 为所有 output 设备打开 outputStream 并创建 PlaybackThread 线程。
5. 为所有 input 设备打开 inputStream 并创建 RecordThread 线程。

## AudioTrack

{% asset_img audiotrack.png audiotrack %}

Android 声音播放都是通过 AudioTrack 进行，包括 MediaPlayer 最终也是创建 AudioTrack 来播放的。通过 AudioTrack 播放声音主要包括下面几步：
1. 创建 AudioTrack。
2. 调用 AudioTrack 的 play() 方法。
3. 调用 AudioTrack 的 write() 方法写入音频数据。

创建 AudioTrack 时重点是通过 AudioPolicyManager 分配了音频路由通路，同时通知服务端 AudioFlinger 创建对应的 Track，用于接收音频数据。
- 调用 play() 方法主要是将创建的 Track 加到 mActiveTracks 并激活沉睡的 PlaybackThread 线程。
- 调用 write() 方法通过共享内存将数据写入服务端 AudioFlinger，PlaybackThread 收到数据激活线程，将数据进行混音等处理再写入对应的 Audio HAL，Audio HAL 再将数据写入驱动或其它外设。

## 音频策略

音频调试参考 [Android Audio](https://source.android.com/devices/audio/debugging?hl=zh-cn)。

首先要搞清楚 stream_type，device，strategy 三者之间的关系：
- AudioSystem::stream_type：音频流的类型
- AudioSystem::audio_devices：音频输入输出设备，每一个 bit 代表一种设备。
- AudioPolicyManagerBase::routing_strategy：音频路由策略

AudioPolicyManagerBase.getStrategy 根据 stream type，返回对应的 routing strategy 值，AudioPolicyManagerBase.getDeviceForStrategy() 则是根据 routing strategy，返回可用的 device。
首先需要的是加载音频设备，音频设备的配置在 `system/etc/audio_policy.conf` 和 `vendor/etc/aduio_policy`, 配置文件中表示了各种 audio interface，通过 AudioFlinger 加载音频设备。
按照一定的优先级选择符合要求的 Device，然后为 Device 选择合适的 Output 通道。
可以通过重载 getStrategy 自己划分 Strategy。

```c++
routing_strategy strategy = (routing_strategy) getStrategyForAttr(&attributes);
audio_devices_t device = getDeviceForStrategy(strategy, false /*fromCache*/);
```

## 截取音频数据

有三个地方可以获取到 PCM 数据：
- 第一个地方 `frameworks/av/media/libmedia/AudioTrack.cpp`

```c++
nsecs_t AudioTrack::processAudioBuffer(){  
{
//在 releaseBuffer 之前 dump 
releaseBuffer(&audioBuffer);  
}  
```
- 第二个地方 `frameworks/av/services/audioflinger/Tracks.cpp`

```c++
status_t AudioFlinger::PlaybackThread::Track::getNextBuffer(
        AudioBufferProvider::Buffer* buffer)
{
    // add dump method
    ServerProxy::Buffer buf;
    size_t desiredFrames = buffer->frameCount;
    buf.mFrameCount = desiredFrames;
    status_t status = mServerProxy->obtainBuffer(&buf);
    buffer->frameCount = buf.mFrameCount;
    buffer->raw = buf.mRaw;
    if (buf.mFrameCount == 0 && !isStopping() && !isStopped() && !isPaused()) {
        ALOGV("underrun,  framesReady(%zu) < framesDesired(%zd), state: %d",
                buf.mFrameCount, desiredFrames, mState);
        mAudioTrackServerProxy->tallyUnderrunFrames(desiredFrames);
    } else {
        mAudioTrackServerProxy->tallyUnderrunFrames(0);
    }

    return status;
}
```
- 第三个地方`hardware/xxx/audio/tinyalsa_hal/audio_hw.c`

```
static ssize_t out_write(struct audio_stream_out *stream, const void* buffer, size_t bytes)
{  
	//add dump method	 
}  
```

## 音频数据流向

Android 系统 audio 框架中主要有三种播放模式：low latency playback、deep buffer playback 和 compressed offload playback。
- low latency / deep buffer 模式下的音频数据流向

{% asset_img ap-audio.png ap_audio %}

- compressed offload 模式下的音频数据流向

{% asset_img offload.png offload %}

- 音频录制
{% asset_img recorder.png recorder %}

- 打电话
{% asset_img phone.png phone %}

## 配置解析

module 下面有 mixPorts、devicePorts 和 routes 子段，它们下面又分别包含多个 mixPort、devicePort 和 route 的字段，这些字段内标识为 source 和 sink 两种角色：
devicePorts(source)：为实际的硬件输入设备；
devicePorts(sink)：为实际的硬件输出设备；
mixPorts(source)：为经过 AudioFlinger 之后的流类型，也称“输出流设备”，是个逻辑设备而非物理设备，对应 AudioFlinger 里面的一个 PlayerThread；
mixPorts(sink)：为进入 AudioFlinger 之前的流类型，也称“输入流设备”，是个逻辑设备而非物理设备，对应 AudioFlinger 里面的一个 RecordThread；
routes：定义 devicePort 和 mixPorts 的路由策略。

{% asset_img module.jpg module %}

profile 参数包含音频流一些信息，比如位数、采样率、通道数，它将被构建为 AudioProfile 对象，保存到 mixPort，然后在存储到 module。对于 xml 里面的 devicePort，一般没有 profile 参数，则会创建一个默认的 profile。当把 mixPort 加入到 Moudle 时，会进行分类：

即 source 角色保存到 OutputProfileCollection mOutputProfiles，sink 角色保存到 InputProfileCollection mInputProfiles。
而 devicePort 则调用 HwModule::setDeclaredDevices() 保存到 module 的 mDeclaredDevices。

{% asset_img hwmodule.jpg hwmodule %}

## HAL

{% asset_img hal.png hal %}

Audio HAL 大致的类图，hal 采用工厂模式，分为 Local 和 HIDL 模式，最后都会调用到 audio_stream_out 或者 audio_stream_in 中，对应调用到 audio_hw.c（由各个厂商实现）中。

## 蓝牙连接例子

{% asset_img bluetooth.png bluetooth %}

AudioService 的 handleDeviceConnection 调用 AudioPolicyManager 的 setDeviceConnectionStateInt。

checkOutputsForDevice 会检测所有的 profile（output），查找每个 profile 是否都存在对应的线程，如果没有则进行创建checkOutputForAllStrategies 切换 AudioTrack 的写入数据源。
---
layout:     post
title:      AudioFlinger分析
date:       2013-06-21 21:21:29
summary:    对Android源码Audio部分AudioFlinger的分析
categories: android audio
---

###一 目的

在AT（AudioTrack）中，我们涉及到的都是流程方面的事务，而不是系统Audio策略上的内容。WHY？因为AT是AF的客户端，而AF是Android系统中Audio管理的中枢。AT我们分析的是按流程方法，那么以AT为切入点的话，AF的分析也应该是流程分析了。
对于分析AT来说，只要能把它的调用顺序（也就是流程说清楚就可以了），但是对于AF的话，简单的分析调用流程 我自己感觉是不够的。因为我发现手机上的声音交互和管理是一件比较复杂的事情。举个简单例子，当听music的时候来电话了，声音处理会怎样？
虽然在Android中，还有一个叫AudioPolicyService的（APS）东西，但是它最终都会调用到AF中去，因为AF实际创建并管理了硬件设备。所以，针对Android声音策略上的分析，我会单独在以后来分析。
###二 从AT切入到AF

直接从头看代码是没法掌握AF的主干的，必须要有一个切入点，也就是用一个正常的调用流程来分析AF的处理流程。先看看AF的产生吧，这个C/S架构的服务者是如何产生的呢？
#####2.1 AudioFlinger的诞生

AF是一个服务，这个就不用我多说了吧？代码在
framework/base/media/mediaserver/Main_mediaServer.cpp中。
<pre><code>int main(int argc, char** argv)
{
    sp<ProcessState> proc(ProcessState::self());
	sp<IServiceManager> sm = defaultServiceManager();
	....
    AudioFlinger::instantiate();--->AF的实例化
	AudioPolicyService::instantiate();--->APS的实例化
	....
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
</code></pre>
看来这个程序的负担很重啊。没想到。为何AF，APS要和MediaService和CameraService都放到一个篮子里？
看看AF的实例化静态函数，在framework/base/libs/audioFlinger/audioFlinger.cpp中
<pre><code>void AudioFlinger::instantiate() {
    defaultServiceManager()->addService( //把AF实例加入系统服务
            String16("media.audio_flinger"), new AudioFlinger());
}
</code></pre>
再来看看它的构造函数是什么做的。
<pre><code>AudioFlinger::AudioFlinger()
    : BnAudioFlinger(),//初始化基类
      mAudioHardware(0), //audio硬件的HAL对象
	mMasterVolume(1.0f), mMasterMute(false), mNextThreadId(0)
{
mHardwareStatus = AUDIO_HW_IDLE;
//创建代表Audio硬件的HAL对象
    mAudioHardware = AudioHardwareInterface::create();
    mHardwareStatus = AUDIO_HW_INIT;
    if (mAudioHardware->initCheck() == NO_ERROR) {
        setMode(AudioSystem::MODE_NORMAL);
//设置系统的声音模式等，其实就是设置硬件的模式
        setMasterVolume(1.0f);
        setMasterMute(false);
    }
}
</code></pre>
AF中经常有setXXX的函数，到底是干什么的呢？我们看看setMode函数。
<pre><code>status_t AudioFlinger::setMode(int mode)
{
     mHardwareStatus = AUDIO_HW_SET_MODE;
    status_t ret = mAudioHardware->setMode(mode);//设置硬件的模式
    mHardwareStatus = AUDIO_HW_IDLE;
    return ret;
}
</code></pre>
当然，setXXX还有些别的东西，但基本上都会涉及到硬件对象。我们暂且不管它。等分析到Audio策略再说。

好了，Android系统启动的时候，看来AF也准备好硬件了。不过，创建硬件对象就代表我们可以播放了吗？
#####2.2 AT调用AF的流程

我这里简单的把AT调用AF的流程列一下，待会按这个顺序分析AF的工作方式。
--参加AudioTrack分析的4.1节

1. 创建
<pre><code>AudioTrack* lpTrack = new AudioTrack();
lpTrack->set(...);
这个就进入到C++的AT了。下面是AT的set函数
audio_io_handle_t output =
AudioSystem::getOutput((AudioSystem::stream_type)streamType,
            sampleRate, format, channels, (AudioSystem::output_flags)flags);
    status_t status = createTrack(streamType, sampleRate, format, channelCount,
                                  frameCount, flags, sharedBuffer, output);
----->creatTrack会和AF打交道。我们看看createTrack重要语句
const sp<IAudioFlinger>& audioFlinger = AudioSystem::get_audio_flinger();
   //下面很重要，调用AF的createTrack获得一个IAudioTrack对象
    sp<IAudioTrack> track = audioFlinger->createTrack();
    sp<IMemory> cblk = track->getCblk();//获取共享内存的管理结构
</code></pre>
总结一下创建的流程，AT调用AF的createTrack获得一个IAudioTrack对象，然后从这个对象中获得共享内存的对象。
2. start和write
看看AT的start，估计就是调用IAudioTrack的start吧？
<pre><code>void AudioTrack::start()
{
	...
   status_t status = mAudioTrack->start();
}
</code></pre>
那write呢?我们之前讲了，AT就是从共享buffer中:

1. Lock缓存
2. 写缓存
3.  Unlock缓存
注意，这里的Lock和Unlock是有问题的，什么问题呢?

按这种方式的话，那么AF一定是有一个线程在那也是：

1. Lock，
2. 读缓存，写硬件
3.  Unlock
总之，我们知道了AT的调用AF的流程了。下面一个一个看。

#####2.3 AF流程

1 createTrack
<pre><code>sp<IAudioTrack> AudioFlinger::createTrack(
        pid_t pid,//AT的pid号
        int streamType,//MUSIC，流类型
        uint32_t sampleRate,//8000 采样率
        int format,//PCM_16类型
        int channelCount,//2，双声道
        int frameCount,//需要创建的buffer可包含的帧数
        uint32_t flags,
        const sp<IMemory>& sharedBuffer,//AT传入的共享buffer，这里为空
        int output,//这个是从AuidoSystem获得的对应MUSIC流类型的索引
        status_t *status)
{
    sp<PlaybackThread::Track> track;
    sp<TrackHandle> trackHandle;
    sp<Client> client;
    wp<Client> wclient;
    status_t lStatus;
       {
        Mutex::Autolock _l(mLock);
//根据output句柄，获得线程？
        PlaybackThread *thread = checkPlaybackThread_l(output);
//看看这个进程是不是已经是AF的客户了
//这里说明一下，由于是C/S架构，那么作为服务端的AF肯定有地方保存作为C的AT的信息
//那么，AF是根据pid作为客户端的唯一标示的
//mClients是一个类似map的数据组织结构
         wclient = mClients.valueFor(pid);
        if (wclient != NULL) {
       } else {
         //如果还没有这个客户信息，就创建一个，并加入到map中去
            client = new Client(this, pid);
            mClients.add(pid, client);
        }
//从刚才找到的那个线程对象中创建一个track
        track = thread->createTrack_l(client, streamType, sampleRate, format,
                channelCount, frameCount, sharedBuffer, &lStatus);
    }
//还有一个trackHandle，而且返回到AF端的是这个trackHandle对象
     trackHandle = new TrackHandle(track);
   return trackHandle;
}
</code></pre>
这个AF函数中，突然冒出来了很多新类型的数据结构。说实话，我刚开始接触的时候，大脑因为常接触到这些眼生的东西而死机！大家先不要拘泥于这些东西，我会一一分析到的。
先进入到checkPlaybackThread_l看看。
<pre><code>
AudioFlinger::PlaybackThread *AudioFlinger::checkPlaybackThread_l(int output) const
{
PlaybackThread *thread = NULL;
//看到这种indexOfKey的东西，应该立即能想到：
//喔，这可能是一个map之类的东西，根据key能找到实际的value
    if (mPlaybackThreads.indexOfKey(output) >= 0) {
        thread = (PlaybackThread *)mPlaybackThreads.valueFor(output).get();
}
//这个函数的意思是根据output值，从一堆线程中找到对应的那个线程
    return thread;
}
</code></pre>

看到这里很疑惑啊：

* AF的构造函数中没有创建线程，只创建了一个audio的HAL对象
* 如果AT是AF的第一个客户的话，我们刚才的调用流程里边，也没看到哪有创建线程的地方呀。
* output是个什么玩意儿？为什么会根据它作为key来找线程呢？
看来，我们得去Output的来源那看看了。
我们知道，output的来源是由AT的set函数得到的：如下：
<pre><code>
audio_io_handle_t output = AudioSystem::getOutput(
(AudioSystem::stream_type)streamType, //MUSIC类型
            sampleRate, //8000
			format, //PCM_16
			channels, //2两个声道
			(AudioSystem::output_flags)flags//0
);
</code></pre>
上面这几个参数后续不再提示了，大家知道这些值都是由AT做为切入点传进去的

然后它在调用AT自己的createTrack，最终把这个output值传递到AF了。其中audio_io_handle_t类型就是一个int类型。

//叫handle啊？好像linux下这种叫法的很少，难道又是受MS的影响吗？

我们进到AudioSystem::getOutput看看。注意，大家想想这是系统的第一次调用,而且发生在AudioTrack那个进程里边。AudioSystem的位置在framework/base/media/libmedia/AudioSystem.cpp中
<pre><code>
audio_io_handle_t AudioSystem::getOutput(stream_type stream,
                                    uint32_t samplingRate,
                                    uint32_t format,
                                    uint32_t channels,
                                    output_flags flags)
{
    audio_io_handle_t output = 0;
    if ((flags & AudioSystem::OUTPUT_FLAG_DIRECT) == 0 &&
        ((stream != AudioSystem::VOICE_CALL && stream != 			AudioSystem::BLUETOOTH_SCO)
         || channels != AudioSystem::CHANNEL_OUT_MONO 
         || (samplingRate != 8000 && samplingRate != 16000))) {
        Mutex::Autolock _l(gLock);

//根据我们的参数，我们会走到这个里边来
//喔，又是从map中找到stream=music的output。可惜啊，我们是第一次进来
//output一定是0
        output = AudioSystem::gStreamOutputMap.valueFor(stream);
       }
	if (output == 0) {

//我晕，又到AudioPolicyService(APS)
//由它去getOutput
        const sp<IAudioPolicyService>& aps = AudioSystem::get_audio_policy_service();
        output = aps->getOutput(stream, samplingRate, format, channels, flags);
        if ((flags & AudioSystem::OUTPUT_FLAG_DIRECT) == 0) {
            Mutex::Autolock _l(gLock);
//如果取到output了，再把output加入到AudioSystem维护的这个map中去
//说白了，就是保存一些信息吗。免得下次又这么麻烦去骚扰APS！
            AudioSystem::gStreamOutputMap.add(stream, output);
        }
    }
    return output;
}
</code></pre>
怎么办？需要到APS中才能找到output的信息？
没办法，硬着头皮进去吧。那先得看看APS是如何创建的。不过这个刚才已经说了，是和AF一块在那个Main_mediaService.cpp中实例化的。
位置在framework/base/lib/libaudioflinger/ AudioPolicyService.cpp中

<pre><code>AudioPolicyService::AudioPolicyService()
    : BnAudioPolicyService() , mpPolicyManager(NULL)
{
    // 下面两个线程以后再说
mTonePlaybackThread = new AudioCommandThread(String8(""));
mAudioCommandThread = new AudioCommandThread(String8("ApmCommandThread"));

if (defined GENERIC_AUDIO) || (defined AUDIO_POLICY_TEST)
//喔，使用普适的AudioPolicyManager，把自己this做为参数
//我们这里先使用普适的看看吧
mpPolicyManager = new AudioPolicyManagerBase(this);
//使用硬件厂商提供的特殊的AudioPolicyManager
    //mpPolicyManager = createAudioPolicyManager(this);
    }
}
</code></pre>
我们看看AudioManagerBase的构造函数吧，在framework/base/lib/audioFlinger/
AudioPolicyManagerBase.cpp中。

<pre><code>AudioPolicyManagerBase::AudioPolicyManagerBase(AudioPolicyClientInterface *clientInterface)
    : mPhoneState(AudioSystem::MODE_NORMAL), mRingerMode(0), mMusicStopTime(0), mLimitRingtoneVolume(false)
{
mpClientInterface = clientInterface;这个client就是APS，刚才通过this传进来了
AudioOutputDescriptor *outputDesc = new AudioOutputDescriptor();
outputDesc->mDevice = (uint32_t)AudioSystem::DEVICE_OUT_SPEAKER;
    mHardwareOutput = mpClientInterface->openOutput(&outputDesc->mDevice,
                                    &outputDesc->mSamplingRate,
                                    &outputDesc->mFormat,
                                    &outputDesc->mChannels,
                                    &outputDesc->mLatency,
                                    outputDesc->mFlags);
  openOutput又交给APS的openOutput来完成了，真绕....
}
</code></pre>
唉，看来我们还是得回到APS，
<pre><code>audio_io_handle_t AudioPolicyService::openOutput(uint32_t *pDevices,
                                uint32_t *pSamplingRate,
                                uint32_t *pFormat,
                                uint32_t *pChannels,
                                uint32_t *pLatencyMs,
                                AudioSystem::output_flags flags)
{
    sp<IAudioFlinger> af = AudioSystem::get_audio_flinger();
//FT,FT,FT,FT,FT,FT,FT
//绕了这么一个大圈子，竟然回到AudioFlinger中了啊？？
return af->openOutput(pDevices, pSamplingRate, (uint32_t *)pFormat, pChannels,
 pLatencyMs, flags);
}
</code></pre>
在我们再次被绕晕之后，我们回眸看看足迹吧：

1. 在AudioTrack中，调用set函数
2. 这个函数会通过AudioSystem::getOutput来得到一个output的句柄
3. AS的getOutput会调用AudioPolicyService的getOutput
4. 然后我们就没继续讲APS的getOutPut了，而是去看看APS创建的东西
5. 发现APS创建的时候会创建一个AudioManagerBase，这个AMB的创建又会调用APS的openOutput。
6. APS的openOutput又会调用AudioFlinger的openOutput
有一个疑问，AT中set参数会和APS构造时候最终传入到AF的openOutput一样吗？如果不一样，那么构造时候openOutput的又是什么参数呢？
先放下这个悬念，我们继续从APS的getOutPut看看。
<pre><code>
audio_io_handle_t AudioPolicyService::getOutput(AudioSystem::stream_type stream,
                                    uint32_t samplingRate,
                                    uint32_t format,
                                    uint32_t channels,
                                    AudioSystem::output_flags flags)
{
     Mutex::Autolock _l(mLock);
//自己又不干活，由AudioManagerBase干活
    return mpPolicyManager->getOutput(stream, samplingRate, format, channels, flags);
}
进去看看吧
audio_io_handle_t AudioPolicyManagerBase::getOutput(AudioSystem::stream_type stream,
                                    uint32_t samplingRate,
                                    uint32_t format,
                                    uint32_t channels,
                                    AudioSystem::output_flags flags)
{
    audio_io_handle_t output = 0;
    uint32_t latency = 0;
    // open a non direct output
    output = mHardwareOutput; //这个是在哪里创建的？在AMB构造的时候..
    return output;
}
</code></pre>
具体AMB的分析待以后Audio系统策略的时候我们再说吧。反正，到这里，我们知道了，在APS构造的时候会open一个Output，而这个Output又会调用AF的openOutput。
<pre><code>
int AudioFlinger::openOutput(uint32_t *pDevices,

                                uint32_t *pSamplingRate,

                                uint32_t *pFormat,

                                uint32_t *pChannels,

                                uint32_t *pLatencyMs,

                                uint32_t flags)

{

    status_t status;

    PlaybackThread *thread = NULL;

    mHardwareStatus = AUDIO_HW_OUTPUT_OPEN;

    uint32_t samplingRate = pSamplingRate ? *pSamplingRate : 0;

    uint32_t format = pFormat ? *pFormat : 0;

    uint32_t channels = pChannels ? *pChannels : 0;

    uint32_t latency = pLatencyMs ? *pLatencyMs : 0;

 

     Mutex::Autolock _l(mLock);

   //由Audio硬件HAL对象创建一个AudioStreamOut对象

    AudioStreamOut *output = mAudioHardware->openOutputStream(*pDevices,

                                                             (int *)&format,

                                                             &channels,

                                                             &samplingRate,

                                                             &status);

   mHardwareStatus = AUDIO_HW_IDLE;

if (output != 0) {

//创建一个Mixer线程

        thread = new MixerThread(this, output, ++mNextThreadId);

        }

//终于找到了，把这个线程加入线程管理组织中

        mPlaybackThreads.add(mNextThreadId, thread);

       return mNextThreadId;

    }

}
</code></pre>
明白了，看来AT在调用AF的createTrack的之前，AF已经在某个时候把线程创建好了，而且是一个Mixer类型的线程，看来和混音有关系呀。这个似乎和我们开始设想的AF工作有点联系喔。Lock，读缓存，写Audio硬件，Unlock。可能都是在这个线程里边做的。

2. 继续createTrack
<pre><code>
AudioFlinger::createTrack(

        pid_t pid,

        int streamType,

        uint32_t sampleRate,

        int format,

        int channelCount,

        int frameCount,

        uint32_t flags,

        const sp<IMemory>& sharedBuffer,

        int output,

        status_t *status)

{

    sp<PlaybackThread::Track> track;

    sp<TrackHandle> trackHandle;

    sp<Client> client;

    wp<Client> wclient;

    status_t lStatus;

    {

//假设我们找到了对应的线程

        Mutex::Autolock _l(mLock);

        PlaybackThread *thread = checkPlaybackThread_l(output);

       //晕，调用这个线程对象的createTrack_l

track = thread->createTrack_l(client, streamType, sampleRate, format,

                channelCount, frameCount, sharedBuffer, &lStatus);

    }

        trackHandle = new TrackHandle(track);

return trackHandle；----》注意，这个对象是最终返回到AT进程中的。
</code></pre>
  实在是....太绕了。再进去看看thread->createTrack_l吧。_l的意思是这个函数进入之前已经获得同步锁了。
跟着sourceinsight ctrl+鼠标左键就进入到下面这个函数。
下面这个函数的签名好长啊。这是为何？

原来Android的C++类中大量定义了内部类。说实话，我之前几年的C++的经验中基本没接触过这么频繁使用内部类的东东。--->当然，你可以说STL也大量使用了呀。

我们就把C++的内部类当做普通的类一样看待吧，其实我感觉也没什么特殊的含义，和外部类是一样的，包括函数调用，public/private之类的东西。这个和JAVA的内部类是大不一样的。
<pre><code>
sp<AudioFlinger::PlaybackThread::Track>  AudioFlinger::PlaybackThread::createTrack_l(

        const sp<AudioFlinger::Client>& client,

        int streamType,

        uint32_t sampleRate,

        int format,

        int channelCount,

        int frameCount,

        const sp<IMemory>& sharedBuffer,

        status_t *status)

{

    sp<Track> track;

    status_t lStatus;

    { // scope for mLock

        Mutex::Autolock _l(mLock);

//new 一个track对象

//我有点愤怒了，Android真是层层封装啊，名字取得也非常相似。

//看看这个参数吧，注意sharedBuffer这个，此时的值应是0

        track = new Track(this, client, streamType, sampleRate, format,

                channelCount, frameCount, sharedBuffer);

       mTracks.add(track); //把这个track加入到数组中，是为了管理用的。

}

lStatus = NO_ERROR;

   return track;

}
</code></pre>
看到这个数组的存在，我们应该能想到什么吗？这时已经有：
 一个MixerThread，内部有一个数组保存track的
看来，不管有多少个AudioTrack，最终在AF端都有一个track对象对应，而且这些所有的track对象都会由一个线程对象来处理。----难怪是Mixer啊
再去看看new Track，我们一直还没找到共享内存在哪里创建的！！！
<pre><code> 
AudioFlinger::PlaybackThread::Track::Track(

            const wp<ThreadBase>& thread,

            const sp<Client>& client,

            int streamType,

            uint32_t sampleRate,

            int format,

            int channelCount,

            int frameCount,

            const sp<IMemory>& sharedBuffer)

    :   TrackBase(thread, client, sampleRate, format, channelCount, frameCount, 0, sharedBuffer),

    mMute(false), mSharedBuffer(sharedBuffer), mName(-1)

{

// mCblk !=NULL?什么时候创建的？？

//只能看基类TrackBase，还是很愤怒，太多继承了。

    if (mCblk != NULL) {

       mVolume[0] = 1.0f;

        mVolume[1] = 1.0f;

        mStreamType = streamType;

         mCblk->frameSize = AudioSystem::isLinearPCM(format) ? channelCount *

 sizeof(int16_t) : sizeof(int8_t);

    }

}

看看基类TrackBase干嘛了

AudioFlinger::ThreadBase::TrackBase::TrackBase(

            const wp<ThreadBase>& thread,

            const sp<Client>& client,

            uint32_t sampleRate,

            int format,

            int channelCount,

            int frameCount,

            uint32_t flags,

            const sp<IMemory>& sharedBuffer)

    :   RefBase(),

        mThread(thread),

        mClient(client),

        mCblk(0),

        mFrameCount(0),

        mState(IDLE),

        mClientTid(-1),

        mFormat(format),

        mFlags(flags & ~SYSTEM_FLAGS_MASK)

{

    size_t size = sizeof(audio_track_cblk_t);

   size_t bufferSize = frameCount*channelCount*sizeof(int16_t);

   if (sharedBuffer == 0) {

       size += bufferSize;

   }

//调用client的allocate函数。这个client是什么？就是我们在CreateTrack中创建的

那个Client，我不想再说了。反正这里会创建一块共享内存

    mCblkMemory = client->heap()->allocate(size);

  有了共享内存，但是还没有里边有同步锁的那个对象audio_track_cblk_t

     mCblk = static_cast<audio_track_cblk_t *>(mCblkMemory->pointer());

     下面这个语法好怪啊。什么意思？？？

new(mCblk) audio_track_cblk_t();

  //各位，这就是C++语法中的placement new。干啥用的啊?new后面的括号中是一块buffer，再

后面是一个类的构造函数。对了，这个placement new的意思就是在这块buffer中构造一个对象。

我们之前的普通new是没法让一个对象在某块指定的内存中创建的。而placement new却可以。

这样不就达到我们的目的了吗？搞一块共享内存，再在这块内存上创建一个对象。这样，这个对象不也就能在两个内存中共享了吗？太牛牛牛牛牛了。怎么想到的？

       // clear all buffers

       mCblk->frameCount = frameCount;

       mCblk->sampleRate = sampleRate;

       mCblk->channels = (uint8_t)channelCount;

}
</code></pre>
好了，解决一个重大疑惑，跨进程数据共享的重要数据结构audio_track_cblk_t是通过placement new在一块共享内存上来创建的。
回到AF的CreateTrack，有这么一句话：
<pre><code>
trackHandle = new TrackHandle(track);

return trackHandle；----》注意，这个对象是最终返回到AT进程中的。

trackHandle的构造使用了thread->createTrack_l的返回值。
</code></pre>

#####2.4 到底有少种对象

读到这里的人，一定会被异常多的class类型，内部类，继承关系搞疯掉。说实话，这里废点心血整个或者paste一个大的UML图未尝不可。但是我是不太习惯用图说话，因为图我实在是记不住。那好吧。我们就用最简单的话语争取把目前出现的对象说清楚。

1. AudioFlinger
<pre><code>class AudioFlinger : public BnAudioFlinger, public IBinder::DeathRecipient</code></pre>

AudioFlinger类是代表整个AudioFlinger服务的类，其余所有的工作类都是通过内部类的方式在其中定义的。你把它当做一个壳子也行吧。

2. Client
Client是描述C/S结构的C端的代表，也就算是一个AT在AF端的对等物吧。不过可不是Binder机制中的BpXXX喔。因为AF是用不到AT的功能的。

<pre><code>class Client : public RefBase {

    public:

        sp<AudioFlinger>    mAudioFlinger;//代表S端的AudioFlinger

        sp<MemoryDealer>    mMemoryDealer;//每个C端使用的共享内存，通过它分配

        pid_t               mPid;//C端的进程id

    };</code></pre>

3. TrackHandle
Trackhandle是AT端调用AF的CreateTrack得到的一个基于Binder机制的Track。
这个TrackHandle实际上是对真正干活的PlaybackThread::Track的一个跨进程支持的封装。
什么意思？本来PlaybackThread::Track是真正在AF中干活的东西，不过为了支持跨进程的话，我们用TrackHandle对其进行了一下包转。这样在AudioTrack调用TrackHandle的功能，实际都由TrackHandle调用PlaybackThread::Track来完成了。可以认为是一种Proxy模式吧。
这个就是AudioFlinger异常复杂的一个原因！！！
<pre><code>class TrackHandle : public android::BnAudioTrack {

    public:

                            TrackHandle(const sp<PlaybackThread::Track>& track);

        virtual             ~TrackHandle();

        virtual status_t    start();

        virtual void        stop();

        virtual void        flush();

        virtual void        mute(bool);

        virtual void        pause();

        virtual void        setVolume(float left, float right);

        virtual sp<IMemory> getCblk() const;

        sp<PlaybackThread::Track> mTrack;

};</code></pre>

 
4. 线程类
AF中有好几种不同类型的线程，分别有对应的线程类型：

* RecordThread：
<pre><code>RecordThread : public ThreadBase, public AudioBufferProvider</code></pre>

用于录音的线程。

* PlaybackThread:
<pre><code>class PlaybackThread : public ThreadBase</code></pre>

用于播放的线程

*  MixerThread
<pre><code>MixerThread : public PlaybackThread</code></pre>

用于混音的线程，注意他是从PlaybackThread派生下来的。

* DirectoutputThread
<pre><code>DirectOutputThread : public PlaybackThread</code></pre>

直接输出线程，我们之前在代码里老看到DIRECT_OUTPUT之类的判断，看来最终和这个线程有关。

* DuplicatingThread：
<pre><code>DuplicatingThread : public MixerThread</code></pre>

复制线程？而且从混音线程中派生？暂时不知道有什么用
这么多线程，都有一个共同的父类ThreadBase，这个是AF对Audio系统单独定义的一个以Thread为基类的类。------》FT，真的很麻烦。
ThreadBase我们不说了，反正里边封装了一些有用的函数。
我们看看PlayingThread吧，里边由定义了内部类：
 
5.  PlayingThread的内部类Track
我们知道，TrackHandle构造用的那个Track是PlayingThread的createTrack_l得到的。
<pre><code>class Track : public TrackBase</code></pre>

晕喔，又来一个TrackBase。
TrackBase是ThreadBase定义的内部类
<pre><code>class TrackBase : public AudioBufferProvider, public RefBase</code></pre>

基类AudioBufferProvider是一个对Buffer的封装，以后在AF读共享缓冲，写数据到硬件HAL中用得到。
个人感觉：上面这些东西，其实完完全全可以独立到不同的文件中，然后加一些注释说明。
写这样的代码，要是我是BOSS的话，一定会很不爽。有什么意义吗？有什么好处吗？
#####2.5 AF流程继续

好了，这里终于在AF中的createTrack返回了TrackHandle。这个时候系统处于什么状态？
AF中的几个Thread我们之前说了，在AF启动的某个时间就已经起来了。我们就假设AT调用AF服务前，这个线程就已经启动了。
这个可以看代码就知道了：
<pre><code>void AudioFlinger::PlaybackThread::onFirstRef()

{

    const size_t SIZE = 256;

    char buffer[SIZE];

 

    snprintf(buffer, SIZE, "Playback Thread %p", this);

//onFirstRef，实际是RefBase的一个方法，在构造sp的时候就会被调用

//下面的run就真正创建了线程并开始执行threadLoop了

    run(buffer, ANDROID_PRIORITY_URGENT_AUDIO);

}</code></pre>

到底执行哪个线程的threadLoop？我记得我们是根据output句柄来查找线程的。
看看openOutput的实行，真正的线程对象创建是在那儿。
<pre><code>int AudioFlinger::openOutput(uint32_t *pDevices,

                                uint32_t *pSamplingRate,

                                uint32_t *pFormat,

                                uint32_t *pChannels,

                                uint32_t *pLatencyMs,

                                uint32_t flags)

{

        if ((flags & AudioSystem::OUTPUT_FLAG_DIRECT) ||

            (format != AudioSystem::PCM_16_BIT) ||

            (channels != AudioSystem::CHANNEL_OUT_STEREO)) {

            thread = new DirectOutputThread(this, output, ++mNextThreadId);

//如果flags没有设置直接输出标准，或者format不是16bit，或者声道数不是2立体声

//则创建DirectOutputThread。       

} else {

    //可惜啊，我们创建的是最复杂的MixerThread  

 thread = new MixerThread(this, output, ++mNextThreadId);   
</code></pre>

1. MixerThread
非常重要的工作线程，我们看看它的构造函数。
<pre><code>AudioFlinger::MixerThread::MixerThread(const sp<AudioFlinger>& audioFlinger, AudioStreamOut* output, int id)

    :   PlaybackThread(audioFlinger, output, id),

        mAudioMixer(0)

{

mType = PlaybackThread::MIXER;

//混音器对象，传进去的两个参数时基类ThreadBase的，都为0

//这个对象巨复杂，最终混音的数据都由它生成，以后再说...

    mAudioMixer = new AudioMixer(mFrameCount, mSampleRate);

   }
</code></pre>
2. AT调用start
此时，AT得到IAudioTrack对象后，调用start函数。
<pre><code>status_t AudioFlinger::TrackHandle::start() {

    return mTrack->start();

} //果然，自己又不干活，交给mTrack了，这个是PlayintThread createTrack_l得到的Track对象

status_t AudioFlinger::PlaybackThread::Track::start()

{

    status_t status = NO_ERROR;

sp<ThreadBase> thread = mThread.promote();

//这个Thread就是调用createTrack_l的那个thread对象，这里是MixerThread

    if (thread != 0) {

        Mutex::Autolock _l(thread->mLock);

        int state = mState;

         if (mState == PAUSED) {

            mState = TrackBase::RESUMING;

           } else {

            mState = TrackBase::ACTIVE;

        }

  //把自己由加到addTrack_l了

//奇怪，我们之前在看createTrack_l的时候，不是已经有个map保存创建的track了

//这里怎么又出现了一个类似的操作？

        PlaybackThread *playbackThread = (PlaybackThread *)thread.get();

        playbackThread->addTrack_l(this);

    return status;

}

看看这个addTrack_l函数

status_t AudioFlinger::PlaybackThread::addTrack_l(const sp<Track>& track)

{

    status_t status = ALREADY_EXISTS;

 

    // set retry count for buffer fill

    track->mRetryCount = kMaxTrackStartupRetries;

    if (mActiveTracks.indexOf(track) < 0) {

        mActiveTracks.add(track);//啊，原来是加入到活跃Track的数组啊

        status = NO_ERROR;

}

//我靠，有戏啊！看到这个broadcast，一定要想到：恩，在不远处有那么一个线程正

//等着这个CV呢。

    mWaitWorkCV.broadcast();

   return status;

}
</code></pre>
让我们想想吧。start是把某个track加入到PlayingThread的活跃Track队列，然后触发一个信号事件。由于这个事件是PlayingThread的内部成员变量，而PlayingThread又创建了一个线程，那么难道是那个线程在等待这个事件吗？这时候有一个活跃track，那个线程应该可以干活了吧？
这个线程是MixerThread。我们去看看它的线程函数threadLoop吧。
<pre><code>bool AudioFlinger::MixerThread::threadLoop()

{

    int16_t* curBuf = mMixBuffer;

    Vector< sp<Track> > tracksToRemove;

 while (!exitPending())

    {

        processConfigEvents();

//Mixer进到这个循环中来

        mixerStatus = MIXER_IDLE;

        { // scope for mLock

           Mutex::Autolock _l(mLock);

            const SortedVector< wp<Track> >& activeTracks = mActiveTracks;

//每次都取当前最新的活跃Track数组

//下面是预备操作，返回状态看看是否有数据需要获取

mixerStatus = prepareTracks_l(activeTracks, &tracksToRemove);

       }

//LIKELY，是GCC的一个东西，可以优化编译后的代码

//就当做是TRUE吧

if (LIKELY(mixerStatus == MIXER_TRACKS_READY)) {

            // mix buffers...

//调用混音器，把buf传进去，估计得到了混音后的数据了

//curBuf是mMixBuffer，PlayingThread的内部buffer，在某个地方已经创建好了，

//缓存足够大

            mAudioMixer->process(curBuf);

            sleepTime = 0;

            standbyTime = systemTime() + kStandbyTimeInNsecs;

        }

有数据要写到硬件中，肯定不能sleep了呀

if (sleepTime == 0) {

           //把缓存的数据写到outPut中。这个mOutput是AudioStreamOut

//由Audio HAL的那个对象创建得到。等我们以后分析再说

           int bytesWritten = (int)mOutput->write(curBuf, mixBufferSize);

            mStandby = false;

        } else {

            usleep(sleepTime);//如果没有数据，那就休息吧..

        }
</code></pre>

3. MixerThread核心
到这里，大家是不是有种焕然一新的感觉？恩，对了，AF的工作就是如此的精密，每个部分都配合得丝丝入扣。不过对于我们看代码的人来说，实在搞不懂这么做的好处----哈哈  有点扯远了。
MixerThread的线程循环中，最重要的两个函数：
prepare_l和mAudioMixer->process，我们一一来看看。
<pre><code>uint32_t AudioFlinger::MixerThread::prepareTracks_l(const SortedVector< wp<Track> >& activeTracks, Vector< sp<Track> > *tracksToRemove)

{

 

    uint32_t mixerStatus = MIXER_IDLE;

    //得到活跃track个数，这里假设就是我们创建的那个AT吧，那么count=1

    size_t count = activeTracks.size();

 

    float masterVolume = mMasterVolume;

    bool  masterMute = mMasterMute;

   for (size_t i=0 ; i<count ; i++) {

        sp<Track> t = activeTracks[i].promote();

      Track* const track = t.get()；

   //得到placement new分配的那个跨进程共享的对象

        audio_track_cblk_t* cblk = track->cblk();

//设置混音器，当前活跃的track。

        mAudioMixer->setActiveTrack(track->name());

        if (cblk->framesReady() && (track->isReady() || track->isStopped()) &&

                !track->isPaused() && !track->isTerminated())

        {

            // compute volume for this track

//AT已经write数据了。所以肯定会进到这来。

            int16_t left, right;

            if (track->isMuted() || masterMute || track->isPausing() ||

                mStreamTypes[track->type()].mute) {

                left = right = 0;

                if (track->isPausing()) {

                    track->setPaused();

                }

//AT设置的音量假设不为零，我们需要聆听声音！

//所以走else流程

            } else {

                // read original volumes with volume control

                float typeVolume = mStreamTypes[track->type()].volume;

                float v = masterVolume * typeVolume;

                float v_clamped = v * cblk->volume[0];

                if (v_clamped > MAX_GAIN) v_clamped = MAX_GAIN;

                left = int16_t(v_clamped);

                v_clamped = v * cblk->volume[1];

                if (v_clamped > MAX_GAIN) v_clamped = MAX_GAIN;

                right = int16_t(v_clamped);

//计算音量

            }

//注意，这里对混音器设置了数据提供来源，是一个track，还记得我们前面说的吗？Track从

AudioBufferProvider派生

          mAudioMixer->setBufferProvider(track);

            mAudioMixer->enable(AudioMixer::MIXING);

 

            int param = AudioMixer::VOLUME;

           //为这个track设置左右音量等

          mAudioMixer->setParameter(param, AudioMixer::VOLUME0, left);

            mAudioMixer->setParameter(param, AudioMixer::VOLUME1, right);

            mAudioMixer->setParameter(

                AudioMixer::TRACK,

                AudioMixer::FORMAT, track->format());

            mAudioMixer->setParameter(

                AudioMixer::TRACK,

                AudioMixer::CHANNEL_COUNT, track->channelCount());

            mAudioMixer->setParameter(

                AudioMixer::RESAMPLE,

                AudioMixer::SAMPLE_RATE,

                int(cblk->sampleRate));

        } else {

           if (track->isStopped()) {

                track->reset();

            }

  //如果这个track已经停止了，那么把它加到需要移除的track队列tracksToRemove中去

//同时停止它在AudioMixer中的混音

            if (track->isTerminated() || track->isStopped() || track->isPaused()) {

                tracksToRemove->add(track);

                mAudioMixer->disable(AudioMixer::MIXING);

            } else {

                mAudioMixer->disable(AudioMixer::MIXING);

            }

        }

    }

 

    // remove all the tracks that need to be...

    count = tracksToRemove->size();

    return mixerStatus;

}</code></pre>

看明白了吗？prepare_l的功能是什么？根据当前活跃的track队列，来为混音器设置信息。可想而知，一个track必然在混音器中有一个对应的东西。我们待会分析AudioMixer的时候再详述。
为混音器准备好后，下面调用它的process函数
<pre><code>void AudioMixer::process(void* output)

{

    mState.hook(&mState, output);//hook？难道是钩子函数？

}

晕乎，就这么简单的函数？？？
CTRL+左键，hook是一个函数指针啊，在哪里赋值的？具体实现函数又是哪个？
没办法了，只能分析AudioMixer类了。
4. AudioMixer
AudioMixer实现在framework/base/libs/audioflinger/AudioMixer.cpp中
AudioMixer::AudioMixer(size_t frameCount, uint32_t sampleRate)

    :   mActiveTrack(0), mTrackNames(0), mSampleRate(sampleRate)

{

    mState.enabledTracks= 0;

    mState.needsChanged = 0;

    mState.frameCount   = frameCount;

    mState.outputTemp   = 0;

    mState.resampleTemp = 0;

    mState.hook         = process__nop;//process__nop，是该类的静态函数

track_t* t = mState.tracks;

//支持32路混音。牛死了

    for (int i=0 ; i<32 ; i++) {

        t->needs = 0;

        t->volume[0] = UNITY_GAIN;

        t->volume[1] = UNITY_GAIN;

        t->volumeInc[0] = 0;

        t->volumeInc[1] = 0;

        t->channelCount = 2;

        t->enabled = 0;

        t->format = 16;

        t->buffer.raw = 0;

        t->bufferProvider = 0;

        t->hook = 0;

        t->resampler = 0;

        t->sampleRate = mSampleRate;

        t->in = 0;

        t++;

    }

}

//其中，mState是在AudioMixer.h中定义的一个数据结构

//注意，source insight没办法解析这个mState，因为....见下面的注释。

struct state_t {

        uint32_t        enabledTracks;

        uint32_t        needsChanged;

        size_t          frameCount;

        mix_t           hook;

        int32_t         *outputTemp;

        int32_t         *resampleTemp;

        int32_t         reserved[2];

        track_t         tracks[32];// __attribute__((aligned(32)));《--把这里注释掉

//否则source insight会解析不了这个state_t类型

    };

    int             mActiveTrack;

    uint32_t        mTrackNames;//names？搞得像字符串，实际是一个int

    const uint32_t  mSampleRate;

	state_t         mState
</code></pre>
好了，没什么吗。hook对应的可选函数实现有：
<pre><code>process__validate

process__nop

process__genericNoResampling

process__genericResampling

process__OneTrack16BitsStereoNoResampling

process__TwoTracks16BitsStereoNoResampling
</code></pre>
AudioMixer构造的时候，hook是process__nop，有几个地方会改变这个函数指针的指向。
这部分涉及到数字音频技术，我就无力讲解了。我们看看最接近的函数
process__OneTrack16BitsStereoNoResampling

<pre><code>void AudioMixer::process__OneTrack16BitsStereoNoResampling(state_t* state, void* output)

{

单track，16bit双声道，不需要重采样,大部分是这种情况了

    const int i = 31 - __builtin_clz(state->enabledTracks);

    const track_t& t = state->tracks[i];

 

    AudioBufferProvider::Buffer& b(t.buffer);

  

    int32_t* out = static_cast<int32_t*>(output);

    size_t numFrames = state->frameCount;

 

    const int16_t vl = t.volume[0];

    const int16_t vr = t.volume[1];

    const uint32_t vrl = t.volumeRL;

    while (numFrames) {

        b.frameCount = numFrames;

//获得buffer

        t.bufferProvider->getNextBuffer(&b);

        int16_t const *in = b.i16;

 

       size_t outFrames = b.frameCount;

       if  UNLIKELY--->不走这.

        else {

            do {

          //计算音量等数据，和数字音频技术有关。这里不说了

                uint32_t rl = *reinterpret_cast<uint32_t const *>(in);

                in += 2;

                int32_t l = mulRL(1, rl, vrl) >> 12;

                int32_t r = mulRL(0, rl, vrl) >> 12;

                *out++ = (r<<16) | (l & 0xFFFF);

            } while (--outFrames);

        }

        numFrames -= b.frameCount;

//释放buffer。

        t.bufferProvider->releaseBuffer(&b);

    }

}</code></pre>

好像挺简单的啊，不就是把数据处理下嘛。这里注意下buffer。到现在，我们还没看到取共享内存里AT端write的数据呐。
那只能到bufferProvider去看了。
注意，这里用的是AudioBufferProvider基类，实际的对象是Track。它从AudioBufferProvider派生。
我们用得是PlaybackThread的这个Track
<pre><code>status_t AudioFlinger::PlaybackThread::Track::getNextBuffer(AudioBufferProvider::Buffer* buffer)

{

//一阵暗喜吧。千呼万唤始出来，终于见到cblk了

     audio_track_cblk_t* cblk = this->cblk();

     uint32_t framesReady;

     uint32_t framesReq = buffer->frameCount;

 //哈哈，看看数据准备好了没，

      framesReady = cblk->framesReady();

 

     if (LIKELY(framesReady)) {

        uint32_t s = cblk->server;

        uint32_t bufferEnd = cblk->serverBase + cblk->frameCount;

        bufferEnd = (cblk->loopEnd < bufferEnd) ? cblk->loopEnd : bufferEnd;

        if (framesReq > framesReady) {

            framesReq = framesReady;

        }

        if (s + framesReq > bufferEnd) {

            framesReq = bufferEnd - s;

        }

获得真实的数据地址

         buffer->raw = getBuffer(s, framesReq);

         if (buffer->raw == 0) goto getNextBuffer_exit;

 

         buffer->frameCount = framesReq;

        return NO_ERROR;

     }

getNextBuffer_exit:

     buffer->raw = 0;

     buffer->frameCount = 0;

    return NOT_ENOUGH_DATA;

}

再看看释放缓冲的地方：releaseBuffer，这个直接在ThreadBase中实现了
void AudioFlinger::ThreadBase::TrackBase::releaseBuffer(AudioBufferProvider::Buffer* buffer)

{

    buffer->raw = 0;

    mFrameCount = buffer->frameCount;

    step();

    buffer->frameCount = 0;

}

看看step吧。mFrameCount表示我已经用完了这么多帧。

bool AudioFlinger::ThreadBase::TrackBase::step() {

    bool result;

    audio_track_cblk_t* cblk = this->cblk();

result = cblk->stepServer(mFrameCount);//哼哼，调用cblk的stepServer，更新

服务端的使用位置

    return result;

}
</code></pre>
到这里，大伙应该都明白了吧。原来AudioTrack中write的数据，最终是这么被使用的呀！！！
恩，看一个process__OneTrack16BitsStereoNoResampling不过瘾，再看看
process__TwoTracks16BitsStereoNoResampling。
<pre><code>void AudioMixer::process__TwoTracks16BitsStereoNoResampling(state_t* state, void*

output)

int i;

    uint32_t en = state->enabledTracks;

 

    i = 31 - __builtin_clz(en);

    const track_t& t0 = state->tracks[i];

    AudioBufferProvider::Buffer& b0(t0.buffer);

 

    en &= ~(1<<i);

    i = 31 - __builtin_clz(en);

    const track_t& t1 = state->tracks[i];

    AudioBufferProvider::Buffer& b1(t1.buffer);

  

    int16_t const *in0;

    const int16_t vl0 = t0.volume[0];

    const int16_t vr0 = t0.volume[1];

    size_t frameCount0 = 0;

 

    int16_t const *in1;

    const int16_t vl1 = t1.volume[0];

    const int16_t vr1 = t1.volume[1];

    size_t frameCount1 = 0;

  

    int32_t* out = static_cast<int32_t*>(output);

    size_t numFrames = state->frameCount;

    int16_t const *buff = NULL;

 

 

    while (numFrames) {

  

        if (frameCount0 == 0) {

            b0.frameCount = numFrames;

            t0.bufferProvider->getNextBuffer(&b0);

            if (b0.i16 == NULL) {

                if (buff == NULL) {

                    buff = new int16_t[MAX_NUM_CHANNELS * state->frameCount];

                }

                in0 = buff;

                b0.frameCount = numFrames;

            } else {

                in0 = b0.i16;

            }

            frameCount0 = b0.frameCount;

        }

        if (frameCount1 == 0) {

            b1.frameCount = numFrames;

            t1.bufferProvider->getNextBuffer(&b1);

            if (b1.i16 == NULL) {

                if (buff == NULL) {

                    buff = new int16_t[MAX_NUM_CHANNELS * state->frameCount];

                }

                in1 = buff;

                b1.frameCount = numFrames;

               } else {

                in1 = b1.i16;

            }

            frameCount1 = b1.frameCount;

        }

      

        size_t outFrames = frameCount0 < frameCount1?frameCount0:frameCount1;

 

        numFrames -= outFrames;

        frameCount0 -= outFrames;

        frameCount1 -= outFrames;

       

        do {

            int32_t l0 = *in0++;

            int32_t r0 = *in0++;

            l0 = mul(l0, vl0);

            r0 = mul(r0, vr0);

            int32_t l = *in1++;

            int32_t r = *in1++;

            l = mulAdd(l, vl1, l0) >> 12;

            r = mulAdd(r, vr1, r0) >> 12;

            // clamping...

            l = clamp16(l);

            r = clamp16(r);

            *out++ = (r<<16) | (l & 0xFFFF);

        } while (--outFrames);

      

        if (frameCount0 == 0) {

            t0.bufferProvider->releaseBuffer(&b0);

        }

        if (frameCount1 == 0) {

            t1.bufferProvider->releaseBuffer(&b1);

        }

    }  

      

    if (buff != NULL) {

        delete [] buff;      

    }

}
</code></pre>
看不懂了吧？？哈哈，知道有这回事就行了，专门搞数字音频的需要好好研究下了！
###三 再论共享audio_track_cblk_t

为什么要再论这个？因为我在网上找了下，有人说audio_track_cblk_t是一个环形buffer，环形buffer是什么意思？自己查查！
这个吗，和我之前的工作经历有关系，某BOSS费尽心机想搞一个牛掰掰的环形buffer，搞得我累死了。现在audio_track_cblk_t是环形buffer？我倒是想看看它是怎么实现的。

顺便我们要解释下，audio_track_cblk_t的使用和我之前说的Lock,读/写，Unlock不太一样。为何？

第一因为我们没在AF代码中看到有缓冲buffer方面的wait，MixThread只有当没有数据的时候会usleep一下。
第二，如果有多个track，多个audio_track_cblk_t的话，假如又是采用wait信号的办法，那么由于pthread库缺乏WaitForMultiObjects的机制，那么到底该等哪一个？这个问题是我们之前在做跨平台同步库的一个重要难题。

1. 写者的使用
我们集中到audio_track_cblk_t这个类，来看看写者是如何使用的。写者就是AudioTrack端，在这个类中，叫user
	* framesAvailable，看看是否有空余空间
	* buffer，获得写空间起始地址
	* stepUser，更新user的位置。

2. 读者的使用
读者是AF端，在这个类中加server。
	* framesReady，获得可读的位置
	* stepServer，更新读者的位置
看看这个类的定义：
<pre><code>
struct audio_track_cblk_t

{

               Mutex       lock; //同步锁

                Condition   cv;//CV

volatile    uint32_t    user;//写者

    volatile    uint32_t    server;//读者

                uint32_t    userBase;//写者起始位置

                uint32_t    serverBase;//读者起始位置

    void*       buffers;

    uint32_t    frameCount;

    // Cache line boundary

    uint32_t    loopStart; //循环起始

    uint32_t    loopEnd; //循环结束

    int         loopCount;

uint8_t     out;   //如果是Track的话，out就是1，表示输出。

}
</code></pre>
注意这是volatile，跨进程的对象，看来这个volatile也是可以跨进程的嘛。
唉，又要发挥下了。volatile只是告诉编译器，这个单元的地址不要cache到CPU的缓冲中。也就是每次取值的时候都要到实际内存中去读，而且可能读内存的时候先要锁一下总线。防止其他CPU核执行的时候同时去修改。由于是跨进程共享的内存，这块内存在两个进程都是能见到的，又锁总线了，又是同一块内存，volatile当然保证了同步一致性。
loopStart和loopEnd这两个值是表示循环播放的起点和终点的，下面还有一个loopCount吗，表示循环播放次数的
那就分析下吧。
先看写者的那几个函数

4. 写者分析
先用frameavail看看当前剩余多少空间，我们可以假设是第一次进来嘛。读者还在那sleep呢。
<pre><code>uint32_t audio_track_cblk_t::framesAvailable()

{

    Mutex::Autolock _l(lock);

    return framesAvailable_l();

}

int32_t audio_track_cblk_t::framesAvailable_l()

{

    uint32_t u = this->user; 当前写者位置，此时也为0

    uint32_t s = this->server; //当前读者位置，此时为0

    if (out) { out为1

        uint32_t limit = (s < loopStart) ? s : loopStart;

我们不设循环播放时间吗。所以loopStart是初始值INT_MAX，所以limit=0

        return limit + frameCount - u;

//返回0+frameCount-0，也就是全缓冲最大的空间。假设frameCount=1024帧

    }

}
</code></pre>
然后调用buffer获得其实位置，buffer就是得到一个地址位置。
<pre><code>void* audio_track_cblk_t::buffer(uint32_t offset) const

{

    return (int8_t *)this->buffers + (offset - userBase) * this->frameSize;

}
</code></pre>
完了，我们更新写者，调用stepUser
<pre><code>uint32_t audio_track_cblk_t::stepUser(uint32_t frameCount)

{

//framecount，表示我写了多少，假设这一次写了512帧

    uint32_t u = this->user;//user位置还没更新呢，此时u=0；

 

    u += frameCount;//u更新了，u=512

    // Ensure that user is never ahead of server for AudioRecord

    if (out) {

       //没甚，计算下等待时间

}

//userBase还是初始值为0，可惜啊，我们只写了1024的一半

//所以userBase加不了

   if (u >= userBase + this->frameCount) {

        userBase += this->frameCount;

//但是这句话很重要，userBase也更新了。根据buffer函数的实现来看，似乎把这个

//环形缓冲铺直了....连绵不绝。

    }

    this->user = u;//喔，user位置也更新为512了，但是useBase还是0

    return u;

}
</code></pre>
好了，假设写者这个时候sleep了，而读者起来了。

5. 读者分析
<pre><code> 
uint32_t audio_track_cblk_t::framesReady()

{

    uint32_t u = this->user; //u为512

    uint32_t s = this->server;//还没读呢，s为零

 

    if (out) {

        if (u < loopEnd) {

            return u - s;//loopEnd也是INT_MAX，所以这里返回512，表示有512帧可读了

        } else {

            Mutex::Autolock _l(lock);

            if (loopCount >= 0) {

                return (loopEnd - loopStart)*loopCount + u - s;

            } else {

                return UINT_MAX;

            }

        }

    } else {

        return s - u;

    }

}

使用完了，然后stepServer
bool audio_track_cblk_t::stepServer(uint32_t frameCount)

{

    status_t err;

   err = lock.tryLock();

    uint32_t s = this->server;

 

    s += frameCount; //读了512帧了，所以s=512

    if (out) {

       

    }

   没有设置循环播放嘛，所以不走这个

    if (s >= loopEnd) {

       s = loopStart;

        if (--loopCount == 0) {

            loopEnd = UINT_MAX;

            loopStart = UINT_MAX;

        }

}

//一样啊，把环形缓冲铺直了

    if (s >= serverBase + this->frameCount) {

        serverBase += this->frameCount;

    }

    this->server = s; //server为512了

    cv.signal(); //读者读完了。触发下写者吧。

    lock.unlock();

    return true;

}</code></pre>

6. 真的是环形缓冲吗？
环形缓冲是这样一个场景，现在buffer共1024帧。
假设：
	* 写者先写到1024帧
	* 读者读到512帧
	* 那么，写者还可以从头写512帧。
所以，我们得回头看看frameavail是不是把这512帧算进来了。
<pre><code>uint32_t audio_track_cblk_t::framesAvailable_l()

{

    uint32_t u = this->user;  //1024

    uint32_t s = this->server;//512

 

    if (out) {

        uint32_t limit = (s < loopStart) ? s : loopStart;

        return limit + frameCount - u;返回512，用上了！

    }

}
</code></pre>
再看看stepUser这句话
<pre><code>if (u >= userBase + this->frameCount) {u为1024，userBase为0，frameCount为1024

        userBase += this->frameCount;//好，userBase也为1024了

}
</code></pre>
看看buffer
<pre><code>return (int8_t *)this->buffers + (offset - userBase) * this->frameSize;
</code></pre>
//offset是外界传入的基于user的一个偏移量。offset-userBase，得到的正式从头开始的那段数据空间。
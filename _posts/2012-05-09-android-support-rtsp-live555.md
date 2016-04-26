---
layout: post
title: 让android支持RTSP及live555分析
---

##如何让Android支持C++异常机制

Android不支持C++异常机制,如果需要用到的话,则需要在编译的时候加入比较完整的C++库.

Android支持的C++库可以在Android NDK中找到(解压后找到libsupc++.a放到代码环境中即可):
[Android NDK](http://www.crystax.net/en/android/ndk/7)

编译时加上参数:

    -fexceptions -lstdc++

还需要将libsupc++.a链接上


##移植live555到Android的例子

[Android live555](https://github.com/boltonli/ohbee/tree/master/android/streamer/jni)


##RTSP协议

参考: rfc2326, rfc3550, rfc3984

RTP Header结构[#0]

    0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |V=2|P|X|  CC   |M|     PT      |       sequence number         |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                           timestamp                           |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |           synchronization source (SSRC) identifier            |
    +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
    |            contributing source (CSRC) identifiers             |
    |                             ....                              |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


##H.264视频格式

参考: rfc3984, 『H.264中的NAL技术』, 『H.264 NAL层解析』


##ACC音频格式

参考: ISO_IEC_13818-7.pdf


##live555架构分析

###0  总述

0.1  这里主要以H264+ACC为基础作介绍

0.2  live555中的demo说明，RTSP服务端为live555MediaServer，openRTSP为调试用客户端。

0.3  可以在live555中实现一个trace_bin的函数跟踪流媒体数据的处理过程。

    void trace_bin(const unsigned char *bytes_ptr, int bytes_num)
    {
        #define LOG_LINE_BYTES 16
        int i, j;

        for (i = 0; i <= bytes_num / LOG_LINE_BYTES; i++) {
            for (j = 0;
                 j < ( (i < (bytes_num / LOG_LINE_BYTES))
                       ? LOG_LINE_BYTES
                       : (bytes_num % LOG_LINE_BYTES) );
                 j++)
            {
                if (0 == j) printf("%04d   ", i * LOG_LINE_BYTES);
                if (LOG_LINE_BYTES/2 == j) printf("   ");
                printf(" %02x", bytes_ptr[i * LOG_LINE_BYTES + j]);
            }
            printf("\n");
        }
    }


###1  宏观流程

1.1  对每个播放请求建立一个session，并对应音视频建立subsession,subsession则是具体处理流媒体的单位。

    ------------------------------------------------------[#1]--
    session     <--->  client requst
       |
    subsession  <--->  audio/video
    ------------------------------------------------------------

1.2  数据处理流程:

    ------------------------------------------------------[#2]--
    source --> filter(source) ... --> sink
    |                           |      |
    +-------+-------------------+      |
            |                          v
            v                    subsession.createNewRTPSink()
    subsession.createNewStreamSource()
    ------------------------------------------------------------

1.3  BasicTaskScheduler::SingleStep()

BasicTaskScheduler是live555的任务处理器，他的主要工作都是在SingleStep()中完成的.

在SingleStep()中主要完成下面三种工作:

    void BasicTaskScheduler::SingleStep(unsigned maxDelayTime) {
        //1. 处理io任务
        ...
        int selectResult = select(fMaxNumSockets, &readSet, &writeSet,
                                  &exceptionSet, &tv_timeToDelay);
        ...
        while ((handler = iter.next()) != NULL) {
            ...
            (*handler->handlerProc)(handler->clientData, resultConditionSet);
            break;
        }
        ...

        //2. handle any newly-triggered event
        ...

        //3. handle any delayed event
        fDelayQueue.handleAlarm();
    }

RTSP请求、链接建立、开始播放处理主要是在1中完成的，而视频播放主要是在3中完成。

以ACC播放为例：

    void ADTSAudioFileSource::doGetNextFrame() {
        // 读取数据并做一些简单处理
        ...
        int numBytesRead = fread(fTo, 1, numBytesToRead, fFid);
        ...

        // 将FramedSource::afterGetting加入fDelayQueue中
        // FramedSource::afterGetting会处理读取到数据，并又会调用
        // ADTSAudioFileSource::doGetNextFrame()，这样实现循环读取文件。
        nextTask() = envir().taskScheduler().scheduleDelayedTask(0,
                    (TaskFunc*)FramedSource::afterGetting, this);
    }

1.4  DelayQueue
fDelayQueue是一个需要处理的任务的队列，每次SingleStep()只会执行第一个任务head()，这里的任务对应DelayQueue的元素，DelayQueue的各个元素都会有自己的DelayTime，用来表示延时多久后执行。而队列中的元素便是按照DelayTime有小到大排列的，元素中fDeltaTimeRemaining记录的是该元素相对于它之前元素的延时。参照函数DelayQueue::addEntry()便可看出是如何入队列的。

例如([]中的数字便是相对延时(fDeltaTimeRemaining))：

    [0]->[1]->[3]->[2]->...->[1]->NULL
     ^
     |
    head()

在处理DelayQueue时往往都要先做一次计时同步操作synchronize()，因为DelayQueue中元素的延时都是相对的，所以一般只要处理首元素即可，不过如果同步之后延时有小于0的，便都会改为DELAY_ZERO(即表示需要立即执行的)。

执行任务：

    void DelayQueue::handleAlarm() {
        ...
        toRemove->handleTimeout();
    }

    void AlarmHandler::handleTimeout() {
        (*fProc)(fClientData);
        DelayQueueEntry::handleTimeout();
    }

任务在处理完成后便会被删除。


###2  类关系

*live555的流程分析主要就放在这个章节中，如果有需要参考函数关系或者对象关系的请参考3, 4两个章节。*

2.1  涉及到的主要类的关系图:

    ------------------------------------------------------[#3]--
    Medium
      +ServerMediaSubsession
      |  +OnDemandServerMediaSubsession
      |     +FileServerMediaSubsession
      |        +H264VideoFileServerMediaSubsession      //h264
      |        +ADTSAudioFileServerMediaSubsession      //aac
      |
      +MediaSink
      |  +RTPSink
      |     +MultiFramedRTPSink
      |        +VideoRTPSink
      |        |  +H264VideoRTPSink                     //h264
      |        +MPEG4GenericRTPSink                     //aac
      |
      +MediaSource
         +FramedSource      //+doGetNextFrame(); +fAfterGettingFunc;
            +FramedFilter
            |  +H264FUAFragmenter                       //h264
            |  +MPEGVideoStreamFramer
            |     +H264VideoStreamFramer                //h264
            +FramedFileSource
               +ByteStreamFileSource                    //h264
               +ADTSAudioFileSource                     //acc

    StreamParser
      +MPEGVideoStreamParser
         +H264VideoStreamParser                         //h264
    ------------------------------------------------------------

我们看下FramedFilter和FramedFileSource相对于FramedSource增加了哪些成员:

    FramedFilter {
        FramedSource* fInputSource;
    }

    FramedFileSource {
        FILE* fFid;
    }

从两者的命名和增加的成员可以看出各自的作用。FramedFilter便是对应着[#2]中的filter，而FramedFileSource则是以本地文件为输入的source。

2.2  如何实现带有filter流程:

这便用到了FramedFilter中的fInputSource成员，以H264为例，

    H264VideoStreamFramer.fInputSource = ByteStreamFileSource;
    H264FUAFragmenter.fInputSource = H264VideoStreamFramer;

将上游source赋值到下游filter的fInputSource即可，对于H264便可以得到下面的一个处理流程:

    ByteStreamFileSource -> H264VideoStreamFramer -> H264FUAFragmenter -> H264VideoRTPSink

在H264VideoStreamFramer的父类MPEGVideoStreamFramer中也有新增成员，

    MPEGVideoStreamFramer {
        MPEGVideoStreamParser* fParser;
    }

    MPEGVideoStreamFramer.fParser = H264VideoStreamParser;

H264VideoStreamParser是用来filter过程中处理视频数据的。

在MultiFramedRTPSink::buildAndSendPacket()中添加RTP头[#0]。


###3  函数关系

3.1  H264函数调用关系

    ------------------------------------------------------[#4]--
    RTSPServer::RTSPClientSession::handleCmd_SETUP()
    OnDemandServerMediaSubsession::getStreamParameters(
        streamToken: new StreamState(
            fMediaSource: H264VideoFileServerMediaSubsession::createNewStreamSource() )
    )

    **********
    RTSPServer::RTSPClientSession::handleCmd_DESCRIBE()
    ServerMediaSession::generateSDPDescription()
    OnDemandServerMediaSubsession::sdpLines()
    H264VideoFileServerMediaSubsession::createNewStreamSource()
    H264VideoStreamFramer::createNew( fInputSource: ByteStreamFileSource::createNew(),
                                      fParser: new H264VideoStreamParser(
                                          fInputSource: H264VideoStreamFramer.fInputSource) )

    **********
    RTSPServer::RTSPClientSession::handleCmd_PLAY()
    H264VideoFileServerMediaSubsession::startStream() [OnDemandServerMediaSubsession::startStream()]
    StreamState::startPlaying()
    H264VideoRTPSink::startPlaying() [MediaSink::startPlaying(fMediaSource)]//got in handleCmd_SETUP()
    H264VideoRTPSink::continuePlaying()
        fSource, fOurFragmenter: H264FUAFragmenter(fInputSource: fMediaSource)
    MultiFramedRTPSink::continuePlaying()
    MultiFramedRTPSink::buildAndSendPacket()
    MultiFramedRTPSink::packFrame()
    H264FUAFragmenter::getNextFrame() [FramedSource::getNextFrame()]
    H264FUAFragmenter::doGetNextFrame() {1}
    1)=No NALU=
      H264VideoStreamFramer::getNextFrame() [FramedSource::getNextFrame()]
      MPEGVideoStreamFramer::doGetNextFrame()
      H264VideoStreamParser::registerReadInterest()
      MPEGVideoStreamFramer::continueReadProcessing()
      H264VideoStreamParser::parse()
      H264VideoStreamFramer::afterGetting() [FramedSource::afterGetting()]
      H264FUAFragmenter::afterGettingFrame()
      H264FUAFragmenter::afterGettingFrame1()
      goto {1}  //Now we have got NALU
    2)=Has NALU=
      FramedSource::afterGetting()
      MultiFramedRTPSink::afterGettingFrame()
      MultiFramedRTPSink::afterGettingFrame1()
      MultiFramedRTPSink::sendPacketIfNecessary()
    ------------------------------------------------------------

###4  对象关系

4.1  H264对象关系图

    ------------------------------------------------------[#5]--
    ServerMediaSession{#1}.fSubsessionsTail = H264VideoFileServerMediaSubsession{2}.fParentSession = {1}
    fStreamStates[] {
      .subsession  = {2}
      .streamToken = StreamState {
                       .fMaster = {2}
                       .fRTPSink = H264VideoRTPSink{5}.fSource/fOurFragmenter
                                 = H264FUAFragmenter{4} {
                                     .fInputSource = H264VideoStreamFramer{3}
                                     .fAfterGettingFunc = MultiFramedRTPSink::afterGettingFrame()
                                     .fAfterGettingClientData = {5}
                                     .fOnCloseFunc = MultiFramedRTPSink::ourHandleClosure()
                                     .fOnCloseClientData = {5}
                                   }
                       .fMediaSource = {3} {
                                         .fParser = H264VideoStreamParser {
                                                      .fInputSource = ByteStreamFileSource{6}
                                                      .fTo = [{5}.]fOutBuf->curPtr()
                                                    }
                                         .fInputSource = {6}
                                         .fAfterGettingFunc = H264FUAFragmenter::afterGettingFrame()
                                         .fAfterGettingClientData = {4}
                                         .fOnCloseFunc = FramedSource::handleClosure()
                                         .fOnCloseClientData = {4}
                                       }
                     }
    }
    ------------------------------------------------------------

4.2  AAC对象关系图

    ------------------------------------------------------[#6]--
    ServerMediaSession{1}.fSubsessionsTail = ADTSAudioFileServerMediaSubsession{2}.fParentSession = {1}
    fStreamStates[] {
      .subsession  = {2}
      .streamToken = StreamState {
                       .fMaster = {2}
                       .fRTPSink = MPEG4GenericRTPSink {
                                     .fOutBuf = OutPacketBuffer
                                     .fSource = ADTSAudioFileSource {3}
                                     .fRTPInterface = RTPInterface.fGS = Groupsock
                                   }
                       .fMediaSource = {3}
                     }
    }
    ------------------------------------------------------------

###5  RTSP

5.1  RTSP命令和处理函数的对应关系:

    RTSP命令               live555中处理函数
    ---------------------------------------------
    OPTIONS        <--->  handleCmd_OPTIONS
    DESCRIBE       <--->  handleCmd_DESCRIBE
    SETUP          <--->  handleCmd_SETUP
    PLAY           <--->  handleCmd_PLAY
    PAUSE          <--->  handleCmd_PAUSE
    TEARDOWN       <--->  handleCmd_TEARDOWN
    GET_PARAMETER  <--->  handleCmd_GET_PARAMETER
    SET_PARAMETER  <--->  handleCmd_SET_PARAMETER

5.2  RTSP播放交互示例(openRTSP)

    --------------------------------------------------------------------------------
    ubuntu$ ./openRTSP rtsp://192.168.43.1/grandma.264
    Opening connection to 192.168.43.1, port 554...
    ...remote connection opened
    Sending request: OPTIONS rtsp://192.168.43.1/grandma.264 RTSP/1.0
    CSeq: 2
    User-Agent: ./openRTSP (LIVE555 Streaming Media v2012.02.29)


    Received 152 new bytes of response data.
    Received a complete OPTIONS response:
    RTSP/1.0 200 OK
    CSeq: 2
    Date: Tue, Jan 25 2011 21:02:53 GMT
    Public: OPTIONS, DESCRIBE, SETUP, TEARDOWN, PLAY, PAUSE, GET_PARAMETER, SET_PARAMETER

    --------------------------------------------------------------------------------
    Sending request: DESCRIBE rtsp://192.168.43.1/grandma.264 RTSP/1.0
    CSeq: 3
    User-Agent: ./openRTSP (LIVE555 Streaming Media v2012.02.29)
    Accept: application/sdp


    Received 682 new bytes of response data.
    Received a complete DESCRIBE response:
    RTSP/1.0 200 OK
    CSeq: 3
    Date: Tue, Jan 25 2011 21:02:53 GMT
    Content-Base: rtsp://192.168.43.1/grandma.264/
    Content-Type: application/sdp
    Content-Length: 517

    v=0
    o=- 1295989373493698 1 IN IP4 0.0.0.0
    s=H.264 Video, streamed by the LIVE555 Media Server
    i=grandma.264
    t=0 0
    a=tool:LIVE555 Streaming Media v2012.02.04
    a=type:broadcast
    a=control:*
    a=range:npt=0-
    a=x-qt-text-nam:H.264 Video, streamed by the LIVE555 Media Server
    a=x-qt-text-inf:grandma.264
    m=video 0 RTP/AVP 96
    c=IN IP4 0.0.0.0
    b=AS:500
    a=rtpmap:96 H264/90000
    a=fmtp:96 packetization-mode=1;profile-level-id=4D4033;
    sprop-parameter-sets=Z01AM5p0FidCAAADAAIAAAMAZR4wZUA=,aO48gA==
    a=control:track1

    Opened URL "rtsp://192.168.43.1/grandma.264", returning a SDP description:
    v=0
    o=- 1295989373493698 1 IN IP4 0.0.0.0
    s=H.264 Video, streamed by the LIVE555 Media Server
    i=grandma.264
    t=0 0
    a=tool:LIVE555 Streaming Media v2012.02.04
    a=type:broadcast
    a=control:*
    a=range:npt=0-
    a=x-qt-text-nam:H.264 Video, streamed by the LIVE555 Media Server
    a=x-qt-text-inf:grandma.264
    m=video 0 RTP/AVP 96
    c=IN IP4 0.0.0.0
    b=AS:500
    a=rtpmap:96 H264/90000
    a=fmtp:96 packetization-mode=1;profile-level-id=4D4033;
    sprop-parameter-sets=Z01AM5p0FidCAAADAAIAAAMAZR4wZUA=,aO48gA==
    a=control:track1

    Created receiver for "video/H264" subsession (client ports 56488-56489)

    --------------------------------------------------------------------------------
    Sending request: SETUP rtsp://192.168.43.1/grandma.264/track1 RTSP/1.0
    CSeq: 4
    User-Agent: ./openRTSP (LIVE555 Streaming Media v2012.02.29)
    Transport: RTP/AVP;unicast;client_port=56488-56489


    Received 205 new bytes of response data.
    Received a complete SETUP response:
    RTSP/1.0 200 OK
    CSeq: 4
    Date: Tue, Jan 25 2011 21:02:53 GMT
    Transport: RTP/AVP;unicast;destination=192.168.43.244;source=192.168.43.1;
    client_port=56488-56489;server_port=6970-6971
    Session: 7626020D


    Setup "video/H264" subsession (client ports 56488-56489)
    Created output file: "video-H264-1"

    --------------------------------------------------------------------------------
    Sending request: PLAY rtsp://192.168.43.1/grandma.264/ RTSP/1.0
    CSeq: 5
    User-Agent: ./openRTSP (LIVE555 Streaming Media v2012.02.29)
    Session: 7626020D
    Range: npt=0.000-


    Received 186 new bytes of response data.
    Received a complete PLAY response:
    RTSP/1.0 200 OK
    CSeq: 5
    Date: Tue, Jan 25 2011 21:02:53 GMT
    Range: npt=0.000-
    Session: 7626020D
    RTP-Info: url=rtsp://192.168.43.1/grandma.264/track1;seq=26490;rtptime=1809652062


    Started playing session
    Receiving streamed data (signal with "kill -HUP 6297" or "kill -USR1 6297" to terminate)...
    Received RTCP "BYE" on "video/H264" subsession (after 35 seconds)

    --------------------------------------------------------------------------------
    Sending request: TEARDOWN rtsp://192.168.43.1/grandma.264/ RTSP/1.0
    CSeq: 6
    User-Agent: ./openRTSP (LIVE555 Streaming Media v2012.02.29)
    Session: 7626020D


    Received 65 new bytes of response data.
    Received a complete TEARDOWN response:
    RTSP/1.0 200 OK
    CSeq: 6
    Date: Tue, Jan 25 2011 21:03:28 GMT
    --------------------------------------------------------------------------------

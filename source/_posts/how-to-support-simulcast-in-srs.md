---
title: How to support Simulcast in SRS
date: 2021-07-22 15:53:03
tags: simulcast
---

# SRS 如何支持 Simulcast

## 1. Simulcast 介绍


WebRTC 中 Simulcast 指同播, 即在一个视频媒体行(MediaLine)中存在多个视频流(RTP Stream), 这些流来自相同的视频采集源, 其差别主要体现在视频编码,分辨率,码率等方面.

目前 WebRTC 源码为 Simulcast 提供了两种接口的API(参考1)

- SDP munging 风格
- RID based 风格

sdp 示例
```
// SDP munging 风格
a=ssrc-group:SIM 1390104252 1390104253 1390104254

// RID based 风格
a=rid:high send
a=rid:mid send
a=rid:low send
a=simulcast:send high;mid;low
```

本文重点讨论如何在 SRS 中支持 SDP munging 风格的 Simulcast, 对于 RID based 风格支持, 笔者会在下一篇介绍

## 2. SDP munging 风格特点

### 2.1 特点

SDP munging 风格的 Simulcast 接口体现在sdp协商时,其视频媒体行会出现 `a=ssrc-group:SIM` 字样,其格式为

`a=ssrc-group:SIM layer0 layer1 layer2...`

其中 SSRC 序列 {layer0 layer1 layer2...} 即为 Simulcast 的多个层级, 序列长度通常不超过3.

它们按照分辨率大小从小到大依次排列; 假定 layer0 的分辨率为 w0xh0, 其他类推, 则:

分辨率满足: 

    layer0(w0xh0) < layer1(w1xh1) <  layer2(w2xh2) ...

长宽比率满足:

    w0 : w1 : w2 = 1: k: k^2
    h0 : h1 : h2 = 1: k: k^2
    (通常k = 2, 如果是源码开发,可自行修改)


### 2.2 示例(完整版见`8. 附录`)

一份offer的某个视频媒体的部分片段

```
a=ssrc-group:FID 1390104252 2798384649
a=ssrc:1390104252 cname:rsPDymoMnGBuVXTx
a=ssrc:1390104252 msid:ez0lPBqQZbbAN0ImN4ZrrUCbshr2khvkB6ob 99528680-68d8-418b-afc6-205372e9cb6f
a=ssrc:2798384649 cname:rsPDymoMnGBuVXTx
a=ssrc:2798384649 msid:ez0lPBqQZbbAN0ImN4ZrrUCbshr2khvkB6ob 99528680-68d8-418b-afc6-205372e9cb6f
a=ssrc-group:FID 1390104253 2798384650
a=ssrc:1390104253 cname:rsPDymoMnGBuVXTx
a=ssrc:1390104253 msid:ez0lPBqQZbbAN0ImN4ZrrUCbshr2khvkB6ob 99528680-68d8-418b-afc6-205372e9cb6f
a=ssrc:2798384650 cname:rsPDymoMnGBuVXTx
a=ssrc:2798384650 msid:ez0lPBqQZbbAN0ImN4ZrrUCbshr2khvkB6ob 99528680-68d8-418b-afc6-205372e9cb6f
a=ssrc-group:FID 1390104254 2798384651
a=ssrc:1390104254 cname:rsPDymoMnGBuVXTx
a=ssrc:1390104254 msid:ez0lPBqQZbbAN0ImN4ZrrUCbshr2khvkB6ob 99528680-68d8-418b-afc6-205372e9cb6f
a=ssrc:2798384651 cname:rsPDymoMnGBuVXTx
a=ssrc:2798384651 msid:ez0lPBqQZbbAN0ImN4ZrrUCbshr2khvkB6ob 99528680-68d8-418b-afc6-205372e9cb6f
a=ssrc-group:SIM 1390104252 1390104253 1390104254
```


### 2.3 解释 

根据这个片段可以推测, 这个sdp里面视频媒体行是一个同播3个码流的 Simulcast

```
low: 1390104252 
mid: 1390104253 
high: 1390104254
```

## 3. SRS 处理 RTC 基本流程回顾与分析

### 3.1 普通场景(single stream)
推流端

以 SRS 的测试网页为例, 打开 http://127.0.0.1:8080/players/rtc_publisher.html 点击 `开始推流`

浏览器在处理 offer 后, 会将 offer 发给 SRS, SRS 处理 offer 后返回 answer给浏览器并就绪等待收流, 浏览器在获得answer,经过后续步骤后最终推流给 SRS

播放端

播放端同样在 SRS 收到浏览器的offer后,回复answer,并准备把推流端的流转发给播放端

### 3.2 Simulcast 场景 (simulcast stream)

对于支持 Simulcast 场景, 大部份流程与上述分析基本一致, 除了以下几点

- 推流端如何生成带有 `a=ssrc-group:SIM`的sdp, 以及确定 `SSRC 序列 {layer0 layer1 layer2...}` 的长度
- SRS 如何识别推流端的 `a=ssrc-group:SIM` 
- 播放端如何拉取指定layer的流

## 4. SRS 开发支持

经过 `3. SRS 处理 RTC 基本流程回顾与分析` 后, 得出 Simulcast 场景(simulcast stream)与普通场景(single stream)的几点主要的不同之处, 这个几点不同,实际对应着几个开发需求. 解决它们就解决问题

额外的, 为了应对Simulcast的细节,我们需要约定一些概念

- numberOfSimulcastLayers: 标示启用Simulcast层级个数, 也就是 `SSRC 序列 {layer0 layer1 layer2...}` 的长度, 用于推流端, 举例 `webrtc://127.0.0.1/live/livestream?numberOfSimulcastLayers=3`
- layer: 指定播放的哪一层, 用于播放端, 举例 `webrtc://127.0.0.1/live/livestream?layer=0`

### 4.1 补充 offer 对 `a=ssrc-group:SIM`的支持

补充 offer 对 `a=ssrc-group:SIM`的支持 主要由 `function UpdateNativeCreateOffer(numberOfSimulcastLayers)` 完成
UpdateNativeCreateOffer 参考 simulcast-playground 项目, 详见 参考2, 其参数 numberOfSimulcastLayers 即为 Simulcast 开的层数

numberOfSimulcastLayers 取值为 1,2,3; 当 numberOfSimulcastLayers=1 时, Simulcast 场景 (simulcast stream) 向下退化为 普通场景(single stream)

UpdateNativeCreateOffer 解析offer的视频媒体行的 SSRC 和 RTX SSRC, 将其作为Simulcast的其中一层, 并根据 numberOfSimulcastLayers 取值, 补充剩下的层级, 并将各层的 SSRC 聚合为

`SSRC 序列 {layer0 layer1 layer2...}` 添加形如 `2.1 特点` 描述的 `a=ssrc-group:SIM layer0 layer1 layer2...`

最后组合为新的offer并返回


### 4.2 补充 SRS 对 `a=ssrc-group:SIM` 的解析

SRS 解析 `a=ssrc-group:SIM` 保存至 SrsSSRCGroup 
在获得  `a=ssrc-group:SIM` 及相应的 `SSRC 序列` 后, SRS 为 publisher 建立与 `SSRC 序列` 与之相对应的 video_track_desc, 此后 publisher启动, 处理推流端的各layer的流

### 4.3 建立 `SSRC 序列` 转发映射, SRS 生成并回复正确的 answer

首先,我们需要知道播放端需要播放哪个层级的视频, 即layer是多少,这个可以在播放端发给offer的消息体中, 扩展一个layer字段, 如

```
var data = {
    api: conf.apiUrl, 
    tid: conf.tid, 
    streamurl: 'webrtc://127.0.0.1/live/livestream?layer=0',
    clientip: null, 
    sdp: offer.sdp
};
```

然后, SRS 解析 streamurl 取得layer后, 以其为索引, 筛选`SSRC 序列`, 然后查询 pulisher 对应的 video_track_desc, 以此创建 player 所需的 video_track_desc 和 SSRC, 返回给播放端, 并处理后续的player逻辑


## 5. 操作演示

### 5.1 启动 SRS

以 osx 为例

```
git clone https://github.com/ossrs/srs/tree/feature/simulcast
cd srs/track 
git checkout feature/simulcast 
git pull
./configure --osx && make -j8
ulimit -HSn 1107
CANDIDATE=$(ifconfig en0 inet| grep inet|awk '{print $2}')
./objs/srs -c conf/rtc.conf
```

### 5.2 操作

打开推流端 http://127.0.0.1:8080/players/rtc_publisher.html

输入 webrtc://127.0.0.1/live/livestream?numberOfSimulcastLayers=3

打开播放端 http://127.0.0.1:8080/players/rtc_player.html

测试 low layer, 输入 webrtc://127.0.0.1/live/livestream?layer=0

测试 mid layer, 输入  webrtc://127.0.0.1/live/livestream?layer=1

测试 high layer, 输入  webrtc://127.0.0.1/live/livestream?layer=2

示例

https://github.com/ossrs/srs/pull/2420#issuecomment-864359462

https://github.com/ossrs/srs/pull/2420#issuecomment-865427561


## 6. 补充

前面 `1. Simulcast 介绍` 提到, Simulcast 主要存在两种API接口, 本文介绍的 SDP munging 为其一, 另一种 RID based 开发已基本完成, 代码提交和介绍文档随后给出

体验详见 https://github.com/johzzy/srs/tree/johnny/experimental

## 7. 参考

1.  
https://www.meetecho.com/blog/simulcast-janus-ssrc/
https://webrtchacks.com/not-a-guide-to-sdp-munging/
2. 
https://github.com/fippo/simulcast-playground
https://fippo.github.io/simulcast-playground/
https://webrtchacks.com/a-playground-for-simulcast-without-an-sfu/


## 8. 附录

一份 SDP munging 风格 Simulcast 的 sdp

https://gist.github.com/johzzy/067dd46a1222bf211f79aef5bcf4cd3a
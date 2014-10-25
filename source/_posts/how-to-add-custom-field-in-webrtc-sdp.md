---
title: How to add custom field in WebRTC SDP
date: 2021-07-23 13:34:08
tags:
---

# WebRTC SDP 如何添加自定义字段

因业务需要, 对于不同场景下的 PeerConnection 需要施以不同的配置, 由此想到解决办法是:

    在 WebRTC SDP 添加自定义字段, 并在源码里增加对应字段的解析器和配置集, 分派至需要配置的地方.

## 1. SDP 修改

举例 

向一份Offer中全局范围添加 `a=johzzy-config hello=1,world=2`, 完整版本详见 附录

```
v=0
o=- 8320953459633665386 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=msid-semantic: WMS fr2ZdH5Z5douqinTOprWuS2lAhjkqyNZJqE4
a=johzzy-config hello=1,world=2
m=audio 9 UDP/TLS/RTP/SAVPF 111 103 104 9 0 8 106 105 13 110 112 113 126
...
```

然后设置到 `PeerConnection.setLocalDescription`

## 2. PeerConnection 解析 SDP 的过程分析

根据WebRTC源码, 以Android为例, 得到相关调用栈如下:
```
PeerConnection.setLocalDescription
...generated_peerconnection_jni...
JNI_PeerConnection_SetLocalDescription
JavaToNativeSessionDescription
CreateSessionDescription
SdpDeserialize


////////////////////////////////
PeerConnection::SetLocalDescription
SdpOfferAnswerHandler::SetLocalDescription
SdpOfferAnswerHandler::DoSetLocalDescription
SdpOfferAnswerHandler::ApplyLocalDescription
...
```

重点看一下 SdpDeserialize
```
// @see pc/webrtc_sdp.cc

bool SdpDeserialize(const std::string& message,
                    JsepSessionDescription* jdesc,
                    SdpParseError* error) {
  std::string session_id;
  std::string session_version;
  TransportDescription session_td("", "");
  RtpHeaderExtensions session_extmaps;
  rtc::SocketAddress session_connection_addr;
  auto desc = std::make_unique<cricket::SessionDescription>();
  size_t current_pos = 0;

  // Session Description
  if (!ParseSessionDescription(message, &current_pos, &session_id,
                               &session_version, &session_td, &session_extmaps,
                               &session_connection_addr, desc.get(), error)) {
    return false;
  }

  // Media Description
  std::vector<std::unique_ptr<JsepIceCandidate>> candidates;
  if (!ParseMediaDescription(message, session_td, session_extmaps, &current_pos,
                             session_connection_addr, desc.get(), &candidates,
                             error)) {
    return false;
  }

  jdesc->Initialize(std::move(desc), session_id, session_version);

  for (const auto& candidate : candidates) {
    jdesc->AddCandidate(candidate.get());
  }
  return true;
}

```

可以看到两个函数 ParseSessionDescription/ParseMediaDescription

前文添加`a=johzzy-config hello=1,world=2`的位置是Session行, 由此可以继续分析  ParseSessionDescription 代码

ParseSessionDescription 函数代码片段
```
  // RFC 4566
  // a=* (zero or more session attribute lines)
  while (GetLineWithType(message, pos, &line, kLineTypeAttributes)) {
    if (HasAttribute(line, kAttributeGroup)) {
      if (!ParseGroupAttribute(line, desc, error)) {
        return false;
      }
    } else if (HasAttribute(line, kAttributeIceUfrag)) {
      if (!GetValue(line, kAttributeIceUfrag, &(session_td->ice_ufrag),
                    error)) {
        return false;
      }
    } else if (HasAttribute(line, kAttributeIcePwd)) {
      if (!GetValue(line, kAttributeIcePwd, &(session_td->ice_pwd), error)) {
        return false;
      }
    } else if (HasAttribute(line, kAttributeIceLite)) {
      session_td->ice_mode = cricket::ICEMODE_LITE;
    } else if (HasAttribute(line, kAttributeIceOption)) {
      if (!ParseIceOptions(line, &(session_td->transport_options), error)) {
        return false;
      }
    } else if (HasAttribute(line, kAttributeFingerprint)) {
      if (session_td->identity_fingerprint.get()) {
        return ParseFailed(
            line,
            "Can't have multiple fingerprint attributes at the same level.",
            error);
      }
      std::unique_ptr<rtc::SSLFingerprint> fingerprint;
      if (!ParseFingerprintAttribute(line, &fingerprint, error)) {
        return false;
      }
      session_td->identity_fingerprint = std::move(fingerprint);
    } else if (HasAttribute(line, kAttributeSetup)) {
      if (!ParseDtlsSetup(line, &(session_td->connection_role), error)) {
        return false;
      }
    } else if (HasAttribute(line, kAttributeMsidSemantics)) {
      std::string semantics;
      if (!GetValue(line, kAttributeMsidSemantics, &semantics, error)) {
        return false;
      }
      desc->set_msid_supported(
          CaseInsensitiveFind(semantics, kMediaStreamSemantic));
    } else if (HasAttribute(line, kAttributeExtmapAllowMixed)) {
      desc->set_extmap_allow_mixed(true);
    } else if (HasAttribute(line, kAttributeExtmap)) {
      RtpExtension extmap;
      if (!ParseExtmap(line, &extmap, error)) {
        return false;
      }
      session_extmaps->push_back(extmap);
    }
  }

```

从这份代码片段可以看到 WebRTC在解析Session级别的`a=*` 字段时, 

先使用 GetLineWithType(message, pos, &line, kLineTypeAttributes) 取一行

然后使用 HasAttribute 判断类型

最后根据具体类型,施以相应的解析流程

对应我们新增的 `a=johzzy-config hello=1,world=2`

就需要新增一下内容

- 类型 `a=johzzy-config`
- 对 `a=johzzy-config` 的解析流程

## 3. PeerConnection 对 SDP 的解析支持

根据上述分析, 可新增如下代码

```
  while (GetLineWithType(message, pos, &line, kLineTypeAttributes)) {
    if ...
    } else if (HasAttribute(line, kAttributeExtmap)) {
      RtpExtension extmap;
      if (!ParseExtmap(line, &extmap, error)) {
        return false;
      }
      session_extmaps->push_back(extmap);
    } else if (HasAttribute(line, kAttributeCustomConfig)) { // 此为新增
      session_td->custom_config.Parse(line);
    }
  }
```

其中相关代码定义如下:
```
static const char kAttributeCustomConfig[] = "johzzy-config";
struct AttributeCustomConfig {
    int Parse(const std::string& line);
};

struct TransportDescription {
    ...
    AttributeCustomConfig custom_config; // 此为新增
}

```

## 4. AttributeCustomConfig 分派

要想解决 AttributeCustomConfig 分派的问题, 需要搞清楚 PeerConnection 的结构

...

待续

## 附录

```
v=0
o=- 8320953459633665386 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=msid-semantic: WMS fr2ZdH5Z5douqinTOprWuS2lAhjkqyNZJqE4
a=johzzy-config hello=1,world=2
m=audio 9 UDP/TLS/RTP/SAVPF 111 103 104 9 0 8 106 105 13 110 112 113 126
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:W5kQ
a=ice-pwd:WYhsnmnjesYbmRu3nfAniUCb
a=ice-options:trickle
a=fingerprint:sha-256 29:77:A9:4F:1C:75:1F:FF:80:BF:41:B0:8A:93:87:43:58:5D:DE:2F:D1:2A:3C:56:0A:FE:F0:26:1A:6C:8E:DE
a=setup:actpass
a=mid:0
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:3 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:5 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:6 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=sendrecv
a=msid:fr2ZdH5Z5douqinTOprWuS2lAhjkqyNZJqE4 d75d31dd-2882-4d4a-8c2f-86520c2351e2
a=rtcp-mux
a=rtpmap:111 opus/48000/2
a=rtcp-fb:111 transport-cc
a=fmtp:111 minptime=10;useinbandfec=1
a=rtpmap:103 ISAC/16000
a=rtpmap:104 ISAC/32000
a=rtpmap:9 G722/8000
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:106 CN/32000
a=rtpmap:105 CN/16000
a=rtpmap:13 CN/8000
a=rtpmap:110 telephone-event/48000
a=rtpmap:112 telephone-event/32000
a=rtpmap:113 telephone-event/16000
a=rtpmap:126 telephone-event/8000
a=ssrc:1576372018 cname:k4SO79jR7YMb0Z0n
a=ssrc:1576372018 msid:fr2ZdH5Z5douqinTOprWuS2lAhjkqyNZJqE4 d75d31dd-2882-4d4a-8c2f-86520c2351e2
a=ssrc:1576372018 mslabel:fr2ZdH5Z5douqinTOprWuS2lAhjkqyNZJqE4
a=ssrc:1576372018 label:d75d31dd-2882-4d4a-8c2f-86520c2351e2
m=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 100 101 102 121 127 120 125 107 108 109 35 36 124 119 123 118 114 115 116
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:W5kQ
a=ice-pwd:WYhsnmnjesYbmRu3nfAniUCb
a=ice-options:trickle
a=fingerprint:sha-256 29:77:A9:4F:1C:75:1F:FF:80:BF:41:B0:8A:93:87:43:58:5D:DE:2F:D1:2A:3C:56:0A:FE:F0:26:1A:6C:8E:DE
a=setup:actpass
a=mid:1
a=extmap:14 urn:ietf:params:rtp-hdrext:toffset
a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:13 urn:3gpp:video-orientation
a=extmap:3 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:12 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay
a=extmap:11 http://www.webrtc.org/experiments/rtp-hdrext/video-content-type
a=extmap:7 http://www.webrtc.org/experiments/rtp-hdrext/video-timing
a=extmap:8 http://www.webrtc.org/experiments/rtp-hdrext/color-space
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:5 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:6 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=sendrecv
a=msid:fr2ZdH5Z5douqinTOprWuS2lAhjkqyNZJqE4 344f6e6d-1bec-478f-bdf1-1265e757dbb8
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 goog-remb
a=rtcp-fb:96 transport-cc
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96
a=rtpmap:98 VP9/90000
a=rtcp-fb:98 goog-remb
a=rtcp-fb:98 transport-cc
a=rtcp-fb:98 ccm fir
a=rtcp-fb:98 nack
a=rtcp-fb:98 nack pli
a=fmtp:98 profile-id=0
a=rtpmap:99 rtx/90000
a=fmtp:99 apt=98
a=rtpmap:100 VP9/90000
a=rtcp-fb:100 goog-remb
a=rtcp-fb:100 transport-cc
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=fmtp:100 profile-id=2
a=rtpmap:101 rtx/90000
a=fmtp:101 apt=100
a=rtpmap:102 H264/90000
a=rtcp-fb:102 goog-remb
a=rtcp-fb:102 transport-cc
a=rtcp-fb:102 ccm fir
a=rtcp-fb:102 nack
a=rtcp-fb:102 nack pli
a=fmtp:102 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42001f
a=rtpmap:121 rtx/90000
a=fmtp:121 apt=102
a=rtpmap:127 H264/90000
a=rtcp-fb:127 goog-remb
a=rtcp-fb:127 transport-cc
a=rtcp-fb:127 ccm fir
a=rtcp-fb:127 nack
a=rtcp-fb:127 nack pli
a=fmtp:127 level-asymmetry-allowed=1;packetization-mode=0;profile-level-id=42001f
a=rtpmap:120 rtx/90000
a=fmtp:120 apt=127
a=rtpmap:125 H264/90000
a=rtcp-fb:125 goog-remb
a=rtcp-fb:125 transport-cc
a=rtcp-fb:125 ccm fir
a=rtcp-fb:125 nack
a=rtcp-fb:125 nack pli
a=fmtp:125 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
a=rtpmap:107 rtx/90000
a=fmtp:107 apt=125
a=rtpmap:108 H264/90000
a=rtcp-fb:108 goog-remb
a=rtcp-fb:108 transport-cc
a=rtcp-fb:108 ccm fir
a=rtcp-fb:108 nack
a=rtcp-fb:108 nack pli
a=fmtp:108 level-asymmetry-allowed=1;packetization-mode=0;profile-level-id=42e01f
a=rtpmap:109 rtx/90000
a=fmtp:109 apt=108
a=rtpmap:35 AV1X/90000
a=rtcp-fb:35 goog-remb
a=rtcp-fb:35 transport-cc
a=rtcp-fb:35 ccm fir
a=rtcp-fb:35 nack
a=rtcp-fb:35 nack pli
a=rtpmap:36 rtx/90000
a=fmtp:36 apt=35
a=rtpmap:124 H264/90000
a=rtcp-fb:124 goog-remb
a=rtcp-fb:124 transport-cc
a=rtcp-fb:124 ccm fir
a=rtcp-fb:124 nack
a=rtcp-fb:124 nack pli
a=fmtp:124 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=4d0032
a=rtpmap:119 rtx/90000
a=fmtp:119 apt=124
a=rtpmap:123 H264/90000
a=rtcp-fb:123 goog-remb
a=rtcp-fb:123 transport-cc
a=rtcp-fb:123 ccm fir
a=rtcp-fb:123 nack
a=rtcp-fb:123 nack pli
a=fmtp:123 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=640032
a=rtpmap:118 rtx/90000
a=fmtp:118 apt=123
a=rtpmap:114 red/90000
a=rtpmap:115 rtx/90000
a=fmtp:115 apt=114
a=rtpmap:116 ulpfec/90000
a=ssrc-group:FID 2456225426 2163325368
a=ssrc:2456225426 cname:k4SO79jR7YMb0Z0n
a=ssrc:2456225426 msid:fr2ZdH5Z5douqinTOprWuS2lAhjkqyNZJqE4 344f6e6d-1bec-478f-bdf1-1265e757dbb8
a=ssrc:2163325368 cname:k4SO79jR7YMb0Z0n
a=ssrc:2163325368 msid:fr2ZdH5Z5douqinTOprWuS2lAhjkqyNZJqE4 344f6e6d-1bec-478f-bdf1-1265e757dbb8
a=ssrc-group:FID 2456225427 2163325369
a=ssrc:2456225427 cname:k4SO79jR7YMb0Z0n
a=ssrc:2456225427 msid:fr2ZdH5Z5douqinTOprWuS2lAhjkqyNZJqE4 344f6e6d-1bec-478f-bdf1-1265e757dbb8
a=ssrc:2163325369 cname:k4SO79jR7YMb0Z0n
a=ssrc:2163325369 msid:fr2ZdH5Z5douqinTOprWuS2lAhjkqyNZJqE4 344f6e6d-1bec-478f-bdf1-1265e757dbb8
a=ssrc-group:FID 2456225428 2163325370
a=ssrc:2456225428 cname:k4SO79jR7YMb0Z0n
a=ssrc:2456225428 msid:fr2ZdH5Z5douqinTOprWuS2lAhjkqyNZJqE4 344f6e6d-1bec-478f-bdf1-1265e757dbb8
a=ssrc:2163325370 cname:k4SO79jR7YMb0Z0n
a=ssrc:2163325370 msid:fr2ZdH5Z5douqinTOprWuS2lAhjkqyNZJqE4 344f6e6d-1bec-478f-bdf1-1265e757dbb8
a=ssrc-group:SIM 2456225426 2456225427 2456225428
```
---
title: SRS 初探之 HLS
top: false
cover: false
toc: true
mathjax: false
date: 2021-05-30 09:54:35
password:
summary: 
categories: 流媒体
tags: 
  - srs
  - hls 
---

## srs 简介

SRS(Simple RTMP Server) 是国内开源的高性能流媒体集群服务器，支持 RTMP/HLS/HTTP-FLV/RTSP/DASH/WebRTC/SRT/GB28181 等多种流媒体协议，高效、稳定、易用，简单而快乐。

本文基于 srs [3.0release](https://github.com/ossrs/srs/tree/3.0release) 版本搭建环境

## HLS 简介

![hls Architecture from <https://developer.apple.com/documentation/http_live_streaming>](/image/srs/hls.png)

HLS(Http Live Streaming) 是 Apple 公司出的流媒体协议，基于 HTTP 协议，以 m3u8 文件形式分发流，支持直播（live）和点播(vod)，一般用于传输 HEVC 或 H.264 视频和 AAC 或 AC-3 音频，流地址如下：

```http
http://www.stream.com/live/livestream.m3u8
```

HLS可以穿过任何允许HTTP数据通过的防火墙或者代理服务器。它也很容易使用内容分发网络来传输媒体流。

### m3u8 文件

`.m3u8` 是播放列表目录文件，存储切片文件索引，可分为 `vod`、`live`、`event` 和 `master` 等多种类型， 前者存储多个分辨率/码率的 media 文件列表，可用于客户端播放自适应多码率视频流，后者仅存储 ts 列表

- live 类型示例：

```m3u8
#EXTM3U                         // m3u8标识标签，
#EXT-X-VERSION:3                // 协议版本
#EXT-X-MEDIA-SEQUENCE:62        // 第一个ts切片的序列号
#EXT-X-TARGETDURATION:5         // ts切片最大时长
#EXTINF:5.000,                  // ts切片信息， <duration>,<title>
stream_62.ts                    // uri地址，ts切片名
#EXTINF:5.000,
stream_63.ts
#EXTINF:5.000,
stream_64.ts
#EXTINF:5.000,
stream_65.ts
#EXTINF:5.000,
stream_66.ts
```

- vod 类型：文件末尾有终止标签 `#EXT-X-ENDLIST`
- event 类型: 文件开头有 类型标签 `EXT-X-PLAYLIST-TYPE`，该标签提供适用于整个播放列表文件的可变性信息。此标签可能包含值 `EVENT` 或 `VOD`。
  - 如果标签存在并且值为 `EVENT`，则服务器不得更改或删除已有播放列表文件的任何部分，可以追加新的 ts 信息，追加完成后以 `#EXT-X-ENDLIST` 结尾。
  - 如果标签存在且值为 `VOD`，则播放列表文件不得做任何更改， 即当播放列表出现 `#EXT-X-ENDLIST` 时，意味着该列表已经固定不会发生任何改动。
- master 文件，提供不同带宽下不同分辨率的流地址，客户端根据测量的网络比特率切换到最合适的地址，包含 `BANDWIDTH`(必须)、`RESOLUTION`、`CODECS`(建议) 等信息

```m3u8
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=150000,RESOLUTION=416x234,CODECS="avc1.42e00a,mp4a.40.2"
http://example.com/low/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=240000,RESOLUTION=416x234,CODECS="avc1.42e00a,mp4a.40.2"
http://example.com/lo_mid/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=440000,RESOLUTION=416x234,CODECS="avc1.42e00a,mp4a.40.2"
http://example.com/hi_mid/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=640000,RESOLUTION=640x360,CODECS="avc1.42e00a,mp4a.40.2"
http://example.com/high/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=64000,CODECS="mp4a.40.5"
http://example.com/audio/index.m3u8
```

### ts 切片


## srs 输出 hls

将 rtmp 流 推送到 srs 服务，通过 srs 将 rtmp 流切片并复用成 ts 格式文件，在没有转码的配置下只支持 h.264+aac 流格式

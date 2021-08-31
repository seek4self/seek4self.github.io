---
title: ffmpeg 命令行
top: false
img: https://cdn.jsdelivr.net/gh/seek4self/imgbed@main/img/blog/ffmpeg/FFmpeg.jpg
cover: false
toc: true
mathjax: false
date: 2021-08-17 18:09:19
password:
summary: ffmpeg 工具官方文档翻译
categories: 音视频
tags: 
- ffmpeg
- video
- audio
- translate
---

FFmpeg 是领先的多媒体框架，能够解码、编码、转码、复用、解复用、流、过滤和播放人类和机器创建的几乎任何东西。它支持最前沿的最晦涩的古代格式。无论它们是由某个标准委员会、社区还是公司设计的。它还具有高度的可移植性：FFmpeg 在各种构建环境、机器架构和配置下编译、运行并通过我们的测试基础设施 FATE，跨越 Linux、Mac OS X、Microsoft Windows、BSD、Solaris 等。

## 概述

```bash
ffmpeg [global_options] {[input_file_options] -i input_url} ... {[output_file_options] output_url} ...
```

`ffmpeg` 是一个非常快速的视频和音频转换器，它也可以从实时音频/视频源中抓取。它还可以在任意采样率之间进行转换，并使用高质量的多相滤波器动态调整视频大小。

`ffmpeg` 从 `-i` 选项指定的任意数量的输入“文件”（可以是常规文件、管道、网络流、抓取设备等）中读取，并写入任意数量的输出“文件”，其中由纯输出 url 指定。在命令行上发现的任何不能被解释为选项的东西都被认为是一个输出 url。

原则上，每个输入或输出 url 可以包含任意数量的不同类型（视频/音频/字幕/附件/数据）的流。允许的流数量和/或类型可能受容器格式的限制。选择哪些流从哪些输入进入哪些输出可以自动完成或使用 `-map` 选项（请参阅流选择章节）。

要在选项中引用输入文件，您必须使用它们的索引（从 0 开始）。例如。第一个输入文件是 0，第二个是 1，等等。类似地，文件中的流由它们的索引引用。例如。 2:3 指的是第三个输入文件中的第四个流。另请参阅[流说明符](#流说明符)一章。

作为一般规则，选项应用于下一个指定的文件。因此，顺序很重要，您可以在命令行上多次使用相同的选项。然后将每次出现应用于下一个输入或输出文件。此规则的例外是全局选项（例如详细级别），应首先指定。

不要混合输入和输出文件——首先指定所有输入文件，然后指定所有输出文件。也不要混合属于不同文件的选项。所有选项仅适用于下一个输入或输出文件，并在文件之间重置。

- 将输出文件的视频比特率设置为 64 kbit/s：

    ```bash
    ffmpeg -i input.avi -b:v 64k -bufsize 64k output.avi
    ```

- 强制输出文件的帧速率为 24 fps:

    ```bash
    ffmpeg -i input.avi -r 24 output.avi
    ```

- 将输入文件的帧速率（仅对原始格式有效）强制为 1 fps，将输出文件的帧速率强制为 24 fps:

    ```bash
    ffmpeg -r 1 -i input.m2v -r 24 output.avi
    ```

原始输入文件可能需要格式选项。

## 详细描述

ffmpeg中每个输出的转码过程可以用下图来描述：

```text
 _______              ______________
|       |            |              |
| input |  demuxer   | encoded data |   decoder
| file  | ---------> | packets      | -----+
|_______|            |______________|      |
                                           v
                                       _________
                                      |         |
                                      | decoded |
                                      | frames  |
                                      |_________|
 ________             ______________       |
|        |           |              |      |
| output | <-------- | encoded data | <----+
| file   |   muxer   | packets      |   encoder
|________|           |______________|

```

`ffmpeg` 调用 `libavformat` 库（包含 demuxers）来读取输入文件并从中获取包含编码数据的数据包。当有多个输入文件时，`ffmpeg`尝试通过跟踪任何活动输入流上的最低时间戳来保持它们同步。

然后将编码的数据包传递给解码器（除非为流选择了 streamcopy，请参阅进一步的说明）。解码器产生未压缩的帧（原始视频/PCM 音频/...），可以通过过滤进一步处理（见下一节）。过滤后，帧被传递到编码器，编码器对它们进行编码并输出编码数据包。最后，这些被传递给 muxer, muxer 将编码的数据包写入输出文件。

### 过滤

在编码之前，`ffmpeg` 可以使用 `libavfilter` 库中的过滤器处理原始音频和视频帧。几个链式过滤器形成一个过滤器图。 `ffmpeg` 区分两种类型的过滤器图：简单的和复杂的。

#### 简单的过滤器图

简单的过滤器图是那些只有一个输入和输出，两者都是相同类型的。在上图中，它们可以通过在解码和编码之间简单地插入一个附加步骤来表示：

```text
 _________                        ______________
|         |                      |              |
| decoded |                      | encoded data |
| frames  |\                   _ | packets      |
|_________| \                  /||______________|
             \   __________   /
  simple     _\||          | /  encoder
  filtergraph   | filtered |/
                | frames   |
                |__________|

```

使用 per-stream `-filter` 选项（分别为视频和音频使用 `-vf` 和 `-af` 别名）配置简单过滤器图。一个简单的视频过滤器图看起来像这样：

```text
 _______        _____________        _______        ________
|       |      |             |      |       |      |        |
| input | ---> | deinterlace | ---> | scale | ---> | output |
|_______|      |_____________|      |_______|      |________|

```

> 请注意，某些过滤器会更改框架属性，但不会更改框架内容。例如。上例中的 `fps` 过滤器会更改帧数，但不会触及帧内容。另一个例子是 `setpts` 过滤器，它只设置时间戳，否则不变地传递帧。

#### 复杂的过滤器图

复杂的过滤器图不能简单地描述为应用于一个流的线性处理链。例如，当图形具有多个输入和/或输出时，或者当输出流类型与输入不同时，就会出现这种情况。它们可以用下图表示：

```text
 _________
|         |
| input 0 |\                    __________
|_________| \                  |          |
             \   _________    /| output 0 |
              \ |         |  / |__________|
 _________     \| complex | /
|         |     |         |/
| input 1 |---->| filter  |\
|_________|     |         | \   __________
               /| graph   |  \ |          |
              / |         |   \| output 1 |
 _________   /  |_________|    |__________|
|         | /
| input 2 |/
|_________|

```

使用 `-filter_complex` 选项配置复杂过滤器图。请注意，此选项是全局的，因为复杂的 filtergraph 就其性质而言，不能明确地与单个流或文件相关联。

`-lavfi` 选项等效于 `-filter_complex`。

复杂过滤器图的一个简单示例是 `overlay` 过滤器，它具有两个视频输入和一个视频输出，其中包含一个叠加在另一个之上的视频。它的音频对应物是 `amix` 过滤器。

### 流复制

流复制是通过将 `copy` 参数提供给 `-codec` 选项来选择的模式。它使 `ffmpeg` 省略了指定流的解码和编码步骤，因此它只进行解复用和复用。它对于更改容器格式或修改容器级元数据很有用。在这种情况下，上图将简化为：

```text
 _______              ______________            ________
|       |            |              |          |        |
| input |  demuxer   | encoded data |  muxer   | output |
| file  | ---------> | packets      | -------> | file   |
|_______|            |______________|          |________|

```

由于没有解码或编码，因此速度非常快且没有质量损失。但是，由于许多因素，它在某些情况下可能不起作用。应用过滤器显然也是不可能的，因为过滤器适用于未压缩的数据。

## 流选择

`ffmpeg` 提供 `-map` 选项用于手动控制每个输出文件中的流选择。用户可以跳过 `-map` 并让 `ffmpeg` 执行自动流选择，如下所述。 `-vn`/`-an`/`-sn`/`-dn` 选项可用于分别跳过包含视频、音频、字幕和数据流，无论是手动映射还是自动选择，除了那些作为复杂过滤器图输出的流。

### 描述

下面的小节描述了流选择中涉及的各种规则。接下来的示例展示了这些规则在实践中的应用。

尽管已尽一切努力准确反映程序的行为，但 FFmpeg 仍在不断开发中，自撰写本文以来，代码可能已更改。

#### 自动流选择

在特定输出文件没有任何映射选项的情况下，ffmpeg 会检查输出格式以检查其中可以包含哪种类型的流，即。视频、音频和/或字幕。对于每种可接受的流类型，ffmpeg 将从所有输入中选择一个可用的流。

它将根据以下标准选择该流：

- 对于视频，它是具有最高分辨率的流，
- 对于音频，它是拥有最多频道的流，
- 对于字幕，它是找到的第一个字幕流，但有一个警告。输出格式的默认字幕编码器可以是基于文本的，也可以是基于图像的，并且只会选择相同类型的字幕流。
  
在几个相同类型的流速率相等的情况下，选择索引最低的流。

数据或附件流不会自动选择，只能使用 `-map` 包含。

#### 手动流选择

使用 `-map` 时，该输出文件中仅包含用户映射的流，下面描述的 filtergraph 输出的一种可能例外。

#### 复杂的过滤器图

如果有任何带有未标记焊盘的复杂 filtergraph 输出流，它们将被添加到第一个输出文件中。如果输出格式不支持流类型，这将导致致命错误。在没有 map 选项的情况下，包含这些流会导致跳过它们类型的自动流选择。如果存在映射选项，除了映射流之外，还包括这些过滤器图流。

带有标记焊盘的复杂 filtergraph 输出流必须映射一次且恰好一次。

#### 流处理

流处理独立于流选择，下面描述的字幕除外。流处理是通过 `-codec` 选项设置的，用于特定输出文件中的流。特别是，编解码器选项由 ffmpeg 在流选择过程之后应用，因此不会影响后者。如果没有为流类型指定 `-codec` 选项，ffmpeg 将选择输出文件复用器注册的默认编码器。

字幕存在例外情况。如果为输出文件指定了字幕编码器，则将包括找到的任何类型、文本或图像的第一个字幕流。 ffmpeg 不验证指定的编码器是否可以转换选定的流，或者转换后的流在输出格式中是否可接受。这也普遍适用：当用户手动设置编码器时，流选择过程无法检查编码流是否可以混合到输出文件中。如果不能，ffmpeg 将中止并且所有输出文件都将无法处理。

### 流选择示例

以下示例说明了 ffmpeg 的流选择方法的行为、怪癖和限制。

这里假设有以下三个输入文件。

```text
input file 'A.avi'
      stream 0: video 640x360
      stream 1: audio 2 channels

input file 'B.mp4'
      stream 0: video 1920x1080
      stream 1: audio 2 channels
      stream 2: subtitles (text)
      stream 3: audio 5.1 channels
      stream 4: subtitles (text)

input file 'C.mkv'
      stream 0: video 1280x720
      stream 1: audio 2 channels
      stream 2: subtitles (image)
```

- 示例：自动流选择

```bash
ffmpeg -i A.avi -i B.mp4 out1.mkv out2.wav -map 1:a -c:a copy out3.mov
```

指定了三个输出文件，对于前两个，没有设置 `-map` 选项，因此 ffmpeg 会自动为这两个文件选择流。

out1.mkv 是 Matroska 容器文件，接受视频、音频和字幕流，因此 ffmpeg 会尝试从每种类型中选择一个。  
对于视频，它将从所有输入视频流中分辨率最高的 B.mp4 中选择 `stream 0`。  
对于音频，它将从 B.mp4 中选择 `stream 3`，因为它具有最多的通道数。  
对于字幕，它会从 B.mp4 中选择 `stream 2`，这是 A.avi 和 B.mp4 中的第一个字幕流。

out2.wav 仅接受音频流，因此仅选择来自 B.mp4 的 `stream 3`。

对于 out3.mov，由于设置了 `-map` 选项，因此不会发生自动流选择。 `-map 1:a` 选项将从第二个输入 B.mp4 中选择所有音频流。此输出文件中不会包含其他流。

对于前两个输出，所有包含的流都将被转码。选择的编码器将是每个输出格式注册的默认编码器，可能与所选输入流的编解码器不匹配。

对于第三个输出，音频流的编解码器选项已设置为 `copy`，因此不会发生或可能发生解码-过滤-编码操作。所选流的数据包应从输入文件传送并在输出文件中混合。

- 示例：自动字幕选择

```bash
ffmpeg -i C.mkv out1.mkv -c:s dvdsub -an out2.mkv
```

虽然 out1.mkv 是接受字幕流的 Matroska 容器文件，但只能选择视频和音频流。 C.mkv 的字幕流是基于图像的，而 Matroska muxer 的默认字幕编码器是基于文本的，因此预计字幕的转码操作会失败，因此未选择该流。然而，在 out2.mkv 中，在命令中指定了字幕编码器，因此除了视频流之外，还选择了字幕流。 `-an` 的存在会禁用 out2.mkv 的音频流选择。

- 示例：未标记的 filtergraph 输出

```bash
ffmpeg -i A.avi -i C.mkv -i B.mp4 -filter_complex "overlay" out1.mp4 out2.srt
```

此处使用 `-filter_complex` 选项设置过滤器图，并由单个视频过滤器组成。`overlay` 过滤器正好需要两个视频输入，但没有指定，因此使用前两个可用的视频流，即 A.avi 和 C.mkv。过滤器的输出垫没有标签，因此被发送到第一个输出文件 out1.mp4。因此，会跳过视频流的自动选择，这会选择 B.mp4 中的流。大多数频道的音频流即。 B.mp4 中的 `stream 3` 是自动选择的。但是没有选择字幕流，因为 MP4 格式没有注册默认的字幕编码器，并且用户没有指定字幕编码器。

第二个输出文件 out2.srt 只接受基于文本的字幕流。因此，即使第一个可用的字幕流属于 C.mkv，它也是基于图像的，因此被跳过。所选流，即 B.mp4 中的 `stream 2`，是第一个基于文本的字幕流。

- 示例：带标签的 filtergraph 输出

```bash
ffmpeg -i A.avi -i B.mp4 -i C.mkv -filter_complex "[1:v]hue=s=0[outv];overlay;aresample" \
       -map '[outv]' -an        out1.mp4 \
                                out2.mkv \
       -map '[outv]' -map 1:a:0 out3.mkv
```

上述命令将失败，因为标记为 `[outv]` 的输出 pad 已被映射两次。不得处理任何输出文件。

```bash
ffmpeg -i A.avi -i B.mp4 -i C.mkv -filter_complex "[1:v]hue=s=0[outv];overlay;aresample" \
       -an        out1.mp4 \
                  out2.mkv \
       -map 1:a:0 out3.mkv
```

上面的这个命令也会失败，因为色调过滤器输出有一个标签，`[outv]`，并且没有被映射到任何地方。

该命令应修改如下，

```bash
ffmpeg -i A.avi -i B.mp4 -i C.mkv -filter_complex "[1:v]hue=s=0,split=2[outv1][outv2];overlay;aresample" \
        -map '[outv1]' -an        out1.mp4 \
                                  out2.mkv \
        -map '[outv2]' -map 1:a:0 out3.mkv
```

来自 B.mp4 的视频流被发送到色调过滤器，其输出使用拆分过滤器克隆一次，并且两个输出都被标记。然后每个副本都映射到第一个和第三个输出文件。

需要两个视频输入的覆盖过滤器使用前两个未使用的视频流。这些是来自 A.avi 和 C.mkv 的流。覆盖输出没有标记，因此无论是否存在 `-map` 选项，它都会发送到第一个输出文件 out1.mp4。

aresample 过滤器发送第一个未使用的音频流，即 A.avi 的音频流。由于此过滤器输出也未标记，因此它也映射到第一个输出文件。 `-an` 的存在只会抑制音频流的自动或手动流选择，而不是从过滤器图发送的输出。这两个映射流都应在 out1.mp4 中的映射流之前排序。

映射到 out2.mkv 的视频、音频和字幕流完全由自动流选择决定。

out3.mkv 包含来自色调过滤器的克隆视频输出和来自 B.mp4 的第一个音频流。

## 选项

如果没有另外指定，所有数字选项都接受表示数字的字符串作为输入，其后可以跟 SI 单位前缀之一，例如：'K'、'M' 或 'G'。

如果将“i”附加到 SI 单位前缀，则完整前缀将被解释为二进制倍数的单位前缀，其基于 1024 的幂而不是 1000 的幂。将“B”附加到 SI 单位前缀将乘以value 为 8。这允许使用，例如：'KB'、'MiB'、'G' 和 'B' 作为数字后缀。

不带参数的选项是布尔选项，并将相应的值设置为 true。可以通过在选项名称前加上“no”来将它们设置为 false。例如，使用 `-nofoo` 会将名称为“foo”的布尔选项设置为false。

### 流说明符

某些选项适用于每个流，例如比特率或编解码器。流说明符用于精确指定给定选项属于哪个流。

流说明符是一个字符串，通常附加到选项名称并用冒号分隔。例如。 `-codec:a:1 ac3` 包含 `a:1` 流说明符，它匹配第二个音频流。因此，它会为第二个音频流选择 ac3 编解码器。

流说明符可以匹配多个流，因此该选项适用于所有流。例如。 `-b:a 128k` 中的流说明符匹配所有音频流。

空流说明符匹配所有流。例如， `-codec copy` 或 `-codec: copy` 将复制所有流而无需重新编码。

流说明符的可能形式是：

- `stream_index`  
  将流与此索引匹配。例如。 `-threads:1 4` 会将第二个流的线程数设置为 4。如果将 *stream_index* 用作附加流说明符（见下文），则它会从匹配的流中选择流编号 *stream_index*。流编号基于 libavformat 检测到的流的顺序，除非还指定了程序 ID。在这种情况下，它基于程序中流的顺序。
- `stream_type[:additional_stream_specifier]`  
  stream_type 是以下之一：'v' 或 'V' 表示视频，'a' 表示音频，'s' 表示字幕，'d' 表示数据，'t' 表示附件。 'v' 匹配所有视频流，'V' 只匹配没有附加图片、视频缩略图或封面艺术的视频流。如果使用了 *additional_stream_specifier*，则它匹配具有此类型并匹配 *additional_stream_specifier* 的流。否则，它匹配指定类型的所有流。
- `p:program_id[:additional_stream_specifier]`  
  匹配程序中带有 id *program_id* 的流。如果使用了 *additional_stream_specifier*，则它匹配既是程序的一部分又匹配 *additional_stream_specifier* 的流。
- `#stream_id or i:stream_id`  
  通过流 ID 匹配流（例如 MPEG-TS 容器中的 PID）。
- `m:key[:value]`  
  匹配具有指定值的元数据标签键的流。如果未给出值，则匹配包含具有任何值的给定标记的流。
- `u`  
  将流与可用配置相匹配，必须定义编解码器，并且必须提供视频维度或音频采样率等基本信息。

> 请注意，在 `ffmpeg` 中，元数据匹配仅适用于输入文件。

### 通用选项

这些选项在 `ff*` 工具之间共享。

- `-L`  
  显示许可证。
- `-h, -?, -help, --help [arg]`  
  显示帮助。可以指定一个可选参数来打印有关特定项目的帮助。如果未指定参数，则仅显示基本（非高级）工具选项。

  *arg* 的可能值是：
  - `long`  
    除基本工具选项外，还打印高级工具选项。
  - `full`  
    打印完整的选项列表，包括编码器、解码器、解复用器、复用器、过滤器等的共享和私有选项。
  - `decoder=decoder_name`  
    打印有关名为 *decoder_name* 的解码器的详细信息。使用 -decoders 选项获取所有解码器的列表。
  - `encoder=encoder_name`  
    打印有关名为 *encoder_name* 的编码器的详细信息。使用 -encoders 选项获取所有编码器的列表。
  - `demuxer=demuxer_name`  
    打印有关名为 *demuxer_name* 的解复用器的详细信息。使用 -formats 选项获取所有 demuxers 和 muxers 的列表。
  - `muxer=muxer_name`  
    打印有关名为 *muxer_name* 的复用器的详细信息。使用 -formats 选项获取所有 demuxers 和 muxers 的列表。
  - `filter=filter_name`  
    打印有关名为 *filter_name* 的过滤器的详细信息。使用 -filters 选项获取所有过滤器的列表。
  - `bsf=bitstream_filter_name`  
    打印有关名为 *bitstream_filter_name* 的比特流过滤器的详细信息。使用 -bsfs 选项获取所有比特流过滤器的列表。
  - `protocol=protocol_name`  
    打印有关名为 *protocol_name* 的协议的详细信息。使用 -protocols 选项获取所有协议的列表。
- `-version`  
  显示版本。
- `-buildconf`  
  显示构建配置，每行一个选项。
- `-formats`  
  显示可用格式（包括设备）。
- `-demuxers`  
  显示可用的解复用器。
- `-muxers`  
  显示可用的复用器。
- `-devices`  
  显示可用设备。
- `-codecs`  
  显示 libavcodec 已知的所有编解码器。
  > 请注意，术语“编解码器”在本文档中用作更准确地称为媒体比特流格式的快捷方式。
- `-decoders`  
  显示可用的解码器。
- `-encoders`  
  显示可用的编码器。
- `-bsfs`  
  显示可用的比特流过滤器。
- `-protocols`  
  显示可用的协议。
- `-filters`  
  显示可用的 libavfilter 过滤器。
- `-sample_fmts`  
  显示可用的样本格式。
- `-layouts`  
  显示频道名称和标准频道布局。
- `-colors`  
  显示识别的颜色名称。
- `-sources device[,opt1=val1[,opt2=val2]...]`  
  显示输入设备的自动检测源。某些设备可能提供无法自动检测的系统相关源名称。不能假定返回的列表总是完整的。

  ```bash
  ffmpeg -sources pulse,server=192.168.0.4
  ```

- `-sinks device[,opt1=val1[,opt2=val2]...]`  
  显示输出设备的自动检测接收器。某些设备可能提供无法自动检测的系统相关接收器名称。不能假定返回的列表总是完整的。

  ```bash
  ffmpeg -sinks pulse,server=192.168.0.4
  ```

- `-loglevel [flags+]loglevel | -v [flags+]loglevel`  
  设置库使用的日志记录级别和标志。
  - 可选的 *flags* 前缀可以由以下值组成：

    `repeat`  
    表示不应该将重复的日志输出压缩到第一行，并且将省略“最后一条消息重复 n 次”这一行。  
    `level`  
    指示日志输出应为每个消息行添加 `[level]` 前缀。这可以用作原木着色的替代方法，例如将日志转储到文件时。

    也可以通过添加 '+'/'-' 前缀来单独使用标志来设置/重置单个标志，而不会影响其他 *flags* 或更改 *loglevel*。当同时设置 *flags* 和 *loglevel* 时，最后一个 *flags* 值和 *loglevel* 之前应该有一个'+'分隔符。

  - *loglevel* 是包含以下值之一的字符串或数字：  

    `quiet, -8`  
    什么都不显示；安静。  
    `panic, 0`  
    只显示可能导致进程崩溃的致命错误，例如断言失败。这目前不用于任何事情。  
    `fatal, 8`  
    只显示致命错误。这些是过程绝对无法继续的错误。  
    `error, 16`  
    显示所有错误，包括可以从中恢复的错误。  
    `warning, 24`  
    显示所有警告和错误。将显示与可能不正确或意外事件相关的任何消息。  
    `info, 32`  
    在处理过程中显示信息性消息。这是警告和错误的补充。这是默认值。  
    `verbose, 40`  
    与 `info` 相同，但更详细。  
    `debug, 48`  
    显示所有内容，包括调试信息。  
    `trace, 56`

    例如启用重复日志输出，添加 `level` 前缀，并将 *loglevel* 设置为 `verbose`：

    ```bash
    ffmpeg -loglevel repeat+level+verbose -i input output
    ```

    启用重复日志输出而不影响 `level` 前缀标志或 *loglevel* 的当前状态的另一个示例：

    ```bash
    ffmpeg [...] -loglevel +repeat
    ```

    默认情况下，程序会记录到 stderr。如果终端支持着色，则使用颜色来标记错误和警告。可以通过设置环境变量 `AV_LOG_FORCE_NOCOLOR` 禁用日志着色，也可以强制设置环境变量 `AV_LOG_FORCE_COLOR`。
- `-report`  
  将完整的命令行和日志输出转储到当前目录中名为 `program-YYYYMMDD-HHMMSS.log` 的文件中。此文件可用于错误报告。它还意味着 `-loglevel debug`。

  将环境变量 `FFREPORT` 设置为任何值具有相同的效果。如果值为‘:’分隔的 key=value 序列，这些选项将影响报告；如果选项值包含特殊字符或选项分隔符 ':'，则必须对其进行转义（请参阅 ffmpeg-utils 手册中的[“引用和转义”]()部分）。

  识别以下选项：

  `file`  
  设置用于报告的文件名； `%p` 扩展为程序名称，`%t` 扩展为时间戳，`%%` 扩展为纯 `%`  
  `level`  
  使用数值设置日志详细级别（请参阅 `-loglevel`）。

  例如，要使用日志级别 `32`（日志级别 `info` 的别名）将报告输出到名为 ffreport.log 的文件：

  ```bash
  FFREPORT=file=ffreport.log:level=32 ffmpeg -i input output
  ```

  解析环境变量的错误不是致命的，不会出现在报告中。
- `-hide_banner`  
  禁止打印横幅。  
  所有 FFmpeg 工具通常都会显示版权声明、构建选项和库版本。此选项可用于禁止打印此信息。
- `-cpuflags flags (global)`  
  允许设置和清除 CPU 标志。此选项用于测试。除非您知道自己在做什么，否则不要使用它。

  ```bash
  ffmpeg -cpuflags -sse+mmx ...
  ffmpeg -cpuflags mmx ...
  ffmpeg -cpuflags 0 ...
  ```

  此选项的可选 flags 如下：

    | Processors          | flags                                                                                                                                                                                |
    | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
    | x86                 | `mmx`, `mmxext`, `sse`, `sse2`, `sse2slow`, `sse3`, `sse3slow`, `ssse3`, `atom`, `sse.1`, `sse.2`, `avx`, `avx2`, `xop`, `fma3`, `fma4`, `3dnow`, `3dnowext`, `bmi1`, `bmi2`, `cmov` |
    | ARM                 | `armv5te`,`armv6`,`armv6t2`,`vfp`,`vfpv3`,`neon`,`setend`                                                                                                                            |
    | AArch64             | `armv8`,`vfp`,`neon`                                                                                                                                                                 |
    | PowerPC             | `altivec`                                                                                                                                                                            |
    | Specific Processors | `pentium2`,`pentium3`,`pentium4`,`k6`,`k62`,`athlon`,`athlonxp`,`k8`                                                                                                                 |
- `-cpucount count (global)`  
  覆盖 CPU 计数检测。此选项用于测试。除非您知道自己在做什么，否则不要使用它。

  ```bash
  ffmpeg -cpucount 2
  ```

- `-max_alloc bytes`  
  通过 ffmpeg 的 malloc 函数系列设置在堆上分配块的最大大小限制。使用此选项时要格外小心。如果您不了解这样做的全部后果，请不要使用。默认值为 INT_MAX。

### AV选项

这些选项由 libavformat、libavdevice 和 libavcodec 库直接提供。要查看可用 AVOptions 的列表，请使用 `-help` 选项。它们分为两类：

- `generic`  
  可以为任何容器、编解码器或设备设置这些选项。通用选项列在容器/设备的 AVFormatContext 选项下和编解码器的 AVCodecContext 选项下。
- `private`  
  这些选项特定于给定的容器、设备或编解码器。私有选项列在其相应的容器/设备/编解码器下。

例如，要将 ID3v2.3 标头而不是默认 ID3v2.4 写入 MP3 文件，请使用 MP3 复用器的 id3v2_version 私有选项：

```bash
ffmpeg -i input.flac -id3v2_version 3 out.mp3
```

所有编解码器 AVOptions 都是每个流，因此应该附加一个流说明符：

```bash
ffmpeg -i multichannel.mxf -map 0:v:0 -map 0:a:0 -map 0:a:0 -c:a:0 ac3 -b:a:0 640k -ac:a:1 2 -c:a:1 aac -b:2 128k out.mp4
```

在上面的示例中，多声道音频流被映射两次以进行输出。第一个实例使用编解码器 ac3 和比特率 640k 进行编码。第二个实例被缩混为 2 个通道并使用编解码器 aac 进行编码。使用输出流的绝对索引为其指定了 128k 的比特率。

> 注意：-nooption 语法不能用于布尔型 AVOptions，请使用 -option 0/-option 1。  
> 注意：通过在选项名称前面加上 v/a/s 来指定每个流 AVOptions 的旧的未记录方式现在已过时，将很快被删除。

### 主要选项

- `-f fmt (input/output)`  
  强制输入或输出文件格式。通常会自动检测输入文件的格式，并根据输出文件的文件扩展名猜测格式，因此在大多数情况下不需要此选项。

- `-i url (input)`  
  输入文件地址

- `-y (global)`  
  无需询问即可覆盖输出文件。

- `-n (global)`  
  不要覆盖输出文件，如果指定的输出文件已经存在，则立即退出。

- `-stream_loop number (input)`  
  设置输入流应循环的次数。循环 0 表示不循环，循环 -1 表示无限循环。

- `-recast_media (global)`  
  允许强制使用与解复用器检测到或指定的媒体类型不同的媒体类型的解码器。用于解码混合为数据流的媒体数据。

- `-c[:stream_specifier] codec (input/output,per-stream)`  
  `-codec[:stream_specifier] codec (input/output,per-stream)`  
  为一个或多个流选择编码器（在输出文件之前使用时）或解码器（在输入文件之前使用时）。`codec` 是解码器/编码器或特殊值 `copy`（仅输出）的名称，以指示不重新编码流。
  例如:

  ```bash
  ffmpeg -i INPUT -map 0 -c:v libx264 -c:a copy OUTPUT
  ```

  使用 libx264 编码所有视频流并复制所有音频流。

  对于每个流，应用最后一个匹配的 `c` 选项，所以

  ```bash
  ffmpeg -i INPUT -map 0 -c copy -c:v:1 libx264 -c:a:137 libvorbis OUTPUT
  ```

  将复制除第二个视频（将使用 libx264 编码）和第 138 个音频（将使用 libvorbis 编码）之外的所有流。

- `-t duration (input/output)`  
  当用作输入选项时（在 -i 之前），限制从输入文件读取数据的持续时间。

  当用作输出选项时（在输出 url 之前），在其持续时间达到持续时间后停止写入输出。

  *duration* 必须是持续时间规范，请参阅 [(ffmpeg-utils) ffmpeg-utils(1) 手册中的持续时间部分](https://ffmpeg.org/ffmpeg-utils.html#time-duration-syntax)。

  -to 和 -t 是互斥的，-t 有优先权。

- `-to position (input/output)`  
  停止在位置写入输出或读取输入。`position` 必须是持续时间规范，请参阅 [(ffmpeg-utils) ffmpeg-utils(1) 手册中的持续时间部分](https://ffmpeg.org/ffmpeg-utils.html#time-duration-syntax)。

  -to 和 -t 是互斥的，-t 有优先权。

- `-fs limit_size (output)`  
  设置文件大小限制，以字节为单位。超出限制后不会再写入更多的字节块。输出文件的大小略大于请求的文件大小。

- `-ss position (input/output)`  
  当用作输入选项（在 `-i` 之前）时，在此输入文件中寻找 *position*。请注意，在大多数格式中，不可能精确查找，因此 `ffmpeg` 将查找 *position* 之前最近的查找点。当转码和 `-accurate_seek` 启用（默认）时，搜索点和 *position* 之间的这个额外段将被解码并丢弃。在进行流复制或使用 `-noaccurate_seek` 时，它将被保留。

  当用作输出选项时（在输出 url 之前），解码但丢弃输入，直到时间戳到达 *position* 。

- `-sseof position (input)`  
  类似于 `-ss` 选项，但相对于“文件结尾”。即负值在文件中较早，0 位于 EOF。

- `-itsoffset offset (input)`
  设置输入时间偏移。

  偏移量被添加到输入文件的时间戳中。指定正偏移量意味着相应的流被延迟了 *offset* 中指定的持续时间。

- `-itsscale scale (input,per-stream)`  
  重新调整输入时间戳。 *scale* 应该是一个浮点数。

- `-timestamp date (output)`  
  在容器中设置录制时间戳。

  *date* 必须是日期规范，请参阅 [(ffmpeg-utils) ffmpeg-utils(1) 手册中的日期部分](https://ffmpeg.org/ffmpeg-utils.html#date-syntax)。

- `-metadata[:metadata_specifier] key=value (output,per-metadata)`
  设置元数据键/值对。

  可以给出可选的 *metadata_specifier* 来设置流、章节或节目的元数据。有关详细信息，请参阅 `-map_metadata` 文档。

  此选项会覆盖使用 `-map_metadata` 设置的元数据。也可以使用空值删除元数据。

  例如，在输出文件中设置标题：

  ```bash
  ffmpeg -i in.avi -metadata title="my title" out.flv
  ```

  设置第一个音频流的语言：

  ```bash
  ffmpeg -i INPUT -metadata:s:a:0 language=eng OUTPUT
  ```

- `-disposition[:stream_specifier] value (output,per-stream)`  
  设置流的处置。

  此选项会覆盖从输入流复制的处置。也可以通过将其设置为 0 来删除处置。

  下面的处置是被允许的：

  `default`, `dub`, `original`, `comment`, `lyrics`, `karaoke`, `forced`, `hearing_impaired`, `visual_impaired`, `clean_effects`, `attached_pic`, `captions`, `descriptions`, `dependent`, `metadata`

  例如，要使第二个音频流成为默认流：

  ```bash
  ffmpeg -i in.mkv -c copy -disposition:a:1 default out.mkv
  ```

  要使第二个字幕流成为默认流并从第一个字幕流中删除默认配置：

  ```bash
  ffmpeg -i in.mkv -c copy -disposition:s:0 0 -disposition:s:1 default out.mkv
  ```

  要添加嵌入式封面/缩略图：

  ```bash
  ffmpeg -i in.mp4 -i IMAGE -map 0 -map 1 -c copy -c:v:1 png -disposition:v:1 attached_pic out.mp4
  ```

  并非所有复用器都支持嵌入式缩略图，支持的多路复用器仅支持几种格式，例如 JPEG 或 PNG。

- `-program [title=title:][program_num=program_num:]st=stream[:st=stream...] (output)`  
  创建具有指定标题 program_num 的程序并将指定的流添加到其中。

- `-target type (output)`  
  指定目标文件类型（`vcd`、`svcd`、`dvd`、`dv`、`dv50`）。 type 可以以 `pal-`、`ntsc-` 或 `film-` 为前缀以使用相应的标准。然后自动设置所有格式选项（比特率、编解码器、缓冲区大小）。您只需键入：

  ```bash
  ffmpeg -i myfile.avi -target vcd /tmp/vcd.mpg
  ```

  不过，您可以指定其他选项，只要您知道它们与标准不冲突，例如：

  ```bash
  ffmpeg -i myfile.avi -target vcd -bf 2 /tmp/vcd.mpg
  ```

- `-dn (input/output)`  
  作为输入选项，阻止文件的所有数据流被过滤或自动选择或映射用于任何输出。请参阅 `-discard` 选项以单独禁用流。

  作为输出选项，禁用数据记录，即任何数据流的自动选择或映射。如需完全手动控制，请参阅 `-map` 选项。

- `-dframes number (output)`  
  设置要输出的数据帧数。这是 `-frames:d` 的过时别名，您应该改用它。

- `-frames[:stream_specifier] framecount (output,per-stream)`  
  在帧计数帧后停止写入流。

- `-q[:stream_specifier] q (output,per-stream)`  
  `-qscale[:stream_specifier] q (output,per-stream)`  
  使用固定质量标度 (VBR)。 q/qscale 的含义取决于编解码器。如果在没有 stream_specifier 的情况下使用 qscale，则它仅适用于视频流，这是为了保持与先前行为的兼容性，并且因为为音频和视频的 2 个不同编解码器指定相同的编解码器特定值通常不是没有 stream_specifier 时的预期目的用来。

- `-filter[:stream_specifier] filtergraph (output,per-stream)`  
  创建由 filtergraph 指定的 filtergraph 并使用它来过滤流。

  *filtergraph* 是对应用于流的 filtergraph 的描述，并且必须具有相同类型的流的单个输入和单个输出。在过滤器图中，输入与标签 `in` 相关联，输出与标签 `out` 相关联。有关 filtergraph 语法的更多信息，请参阅 ffmpeg-filters 手册。

  如果要创建具有多个输入和/或输出的过滤器图，请参阅 [`-filter_complex`](https://ffmpeg.org/ffmpeg.html#filter_005fcomplex_005foption) 选项。

- `-filter_script[:stream_specifier] filename (output,per-stream)`  
  此选项与 -filter 类似，唯一的区别是它的参数是要从中读取 filtergraph 描述的文件的名称。

- `-reinit_filter[:stream_specifier] integer (input,per-stream)`  
  此布尔选项确定当输入帧参数在中途更改时，此流所馈送的过滤器图是否重新初始化。默认情况下启用此选项，因为大多数视频和所有音频过滤器都无法处理输入帧属性中的偏差。重新初始化后，现有的过滤器状态将丢失，例如某些过滤器中可用的帧数 `n` 参考。重新初始化时缓冲的任何帧都将丢失。对于视频，更改触发重新初始化的属性是帧分辨率或像素格式；用于音频、样本格式、采样率、通道数或通道布局。

- `-filter_threads nb_threads (global)`  
  定义用于处理过滤器管道的线程数。每个管道将生成一个线程池，其中包含许多可用于并行处理的线程。默认值为可用 CPU 的数量。

- `-pre[:stream_specifier] preset_name (output,per-stream)`  
  指定匹配流的预设。

- `-stats (global)`  
  打印编码进度/统计信息。默认情况下它是打开的，要明确禁用它，您需要指定 `-nostats`。

- `-stats_period time (global)`  
  设置更新编码进度/统计信息的时间段。默认值为 0.5 秒。

- `-progress url (global)`  
  将程序友好的进度信息发送到 url。

  进度信息会定期写入并在编码过程结束时写入。它由“key=value”行组成。键仅由字母数字字符组成。进度信息序列的最后一个键始终是“进度”。

  使用 `-stats_period` 设置更新周期。

- `-stdin`  
  在标准输入上启用交互。默认情况下打开，除非使用标准输入作为输入。要显式禁用交互，您需要指定 `-nostdin`。

  禁用标准输入上的交互很有用，例如，如果 ffmpeg 在后台进程组中。使用 `ffmpeg ... < /dev/null` 可以实现大致相同的结果，但它需要一个 shell。

- `-debug_ts (global)`  
  打印时间戳信息。默认情况下它是关闭的。此选项主要用于测试和调试目的，输出格式可能会从一个版本更改为另一个版本，因此不应由可移植脚本使用。

- `-attach filename (output)`  
  将附件添加到输出文件。这得到了一些格式的支持，比如 Matroska，例如用于渲染字幕的字体。附件作为特定类型的流实现，因此此选项将向文件添加新流。然后可以以通常的方式在此流上使用每个流选项。使用此选项创建的附件流将在所有其他流（即使用 `-map` 或自动映射创建的流）之后创建。

  > 请注意，对于 Matroska，您还必须设置 mimetype 元数据标签：

  ```bash
  # 假设附件流将是输出文件中的第三个。
  ffmpeg -i INPUT -attach DejaVuSans.ttf -metadata:s:2 mimetype=application/x-truetype-font out.mkv
  ```

- `-dump_attachment[:stream_specifier] filename (input,per-stream)`  
  将匹配的附件流提取到名为 `filename` 的文件中。如果文件名为空，则将使用文件名元数据标签的值。

  例如。将第一个附件提取到名为“out.ttf”的文件中：

  ```bash
  ffmpeg -dump_attachment:t:0 out.ttf -i INPUT
  ```

  提取由 `filename` 标签确定的文件的所有附件：

  ```bash
  ffmpeg -dump_attachment:t "" -i INPUT
  ```

  > 技术说明 - 附件作为编解码器额外数据实现，因此此选项实际上可用于从任何流中提取额外数据，而不仅仅是附件。

## 示例

### 视频和音频抓取

如果指定输入格式和设备，则 ffmpeg 可以直接抓取视频和音频。

```bash
ffmpeg -f oss -i /dev/dsp -f video4linux2 -i /dev/video0 /tmp/out.mpg
```

或者使用 ALSA 音频源（单声道输入，卡 ID 1）而不是 OSS：

```bash
ffmpeg -f alsa -ac 1 -i hw:1 -f video4linux2 -i /dev/video0 /tmp/out.mpg
```

> 请注意，在使用任何电视查看器（例如 Gerd Knorr 的 xawtv）启动 ffmpeg 之前，您必须激活正确的视频源和频道。您还必须使用标准混音器正确设置录音电平。

### 视频和音频文件格式转换

任何支持的文件格式和协议都可以作为 ffmpeg 的输入：

- 您可以使用 YUV 文件作为输入:
  
  ```bash
  ffmpeg -i /tmp/test%d.Y /tmp/out.mpg
  ```

  它将使用以下文件：

  ```text
  /tmp/test0.Y, /tmp/test0.U, /tmp/test0.V,
  /tmp/test1.Y, /tmp/test1.U, /tmp/test1.V, etc...
  ```

  Y 文件使用两倍于 U 和 V 文件的分辨率。它们是原始文件，没有标题。它们可以由所有体面的视频解码器生成。如果 ffmpeg 无法猜测，您必须使用 -s 选项指定图像的大小。

- 您可以从原始 YUV420P 文件输入:

  ```bash
  ffmpeg -i /tmp/test.yuv /tmp/out.avi
  ```

  test.yuv 是一个包含原始 YUV 平面数据的文件。每帧由 Y 平面和 U 平面和 V 平面组成，垂直和水平分辨率的一半。

- 您可以输出到原始 YUV420P 文件：

  ```bash
  ffmpeg -i mydivx.avi hugefile.yuv
  ```

- 可以设置多个输入文件和输出文件:

  ```bash
  # 将音频文件 a.wav 和原始 YUV 视频文件 a.yuv 转换为 MPEG 文件 a.mpg。
  ffmpeg -i /tmp/a.wav -s 640x480 -i /tmp/a.yuv /tmp/a.mpg
  ```

- 您还可以同时进行音频和视频转换:

  ```bash
  # 以 22050 Hz 采样率将 a.wav 转换为 MPEG 音频。
  ffmpeg -i /tmp/a.wav -ar 22050 /tmp/a.mp2
  ```

- 您可以同时编码为多种格式并定义从输入流到输出流的映射:

  ```bash
  # 将 a.wav 转换为 64 kbits 的 a.mp2 和 128 kbits 的 b.mp2。
  ffmpeg -i /tmp/a.wav -map 0:a -b:a 64k /tmp/a.mp2 -map 0:a -b:a 128k /tmp/b.mp2
  ```

   '-map file:index' 按照输出流定义的顺序指定每个输出流使用哪个输入流。

- 您可以对解密的 VOB 进行转码：

  ```bash
  ffmpeg -i snatch_1.vob -f avi -c:v mpeg4 -b:v 800k -g 300 -bf 2 -c:a libmp3lame -b:a 128k snatch.avi
  ```

  这是一个典型的 DVD 翻录示例；输入是一个 VOB 文件，输出一个带有 MPEG-4 视频和 MP3 音频的 AVI 文件。请注意，在此命令中，我们使用 B 帧，因此 MPEG-4 流与 DivX5 兼容，GOP 大小为 300，这意味着对于 29.97fps 输入视频，每 10 秒有一个内帧。此外，音频流是 MP3 编码的，因此您需要通过传递 `--enable-libmp3lame` 进行配置来启用 LAME 支持。该映射对于 DVD 转码以获得所需的音频语言特别有用。

  > 注意：要查看支持的输入格式，请使用 ffmpeg -demuxers。

- 您可以从视频中提取图像，或从多个图像创建视频：

  - 从视频中提取图像：
  
  ```bash
  ffmpeg -i foo.avi -r 1 -s WxH -f image2 foo-%03d.jpeg
  ```

  这将每秒从视频中提取一个视频帧，并将它们输出到名为 foo-001.jpeg、foo-002.jpeg 等的文件中。图像将被重新缩放以适应新的 WxH 值。

  如果您只想提取有限数量的帧，可以将上述命令与 `-frames:v` 或 `-t` 选项结合使用，或者与 `-ss` 结合使用，以从某个时间点开始提取。

  - 从多个图像创建视频：

  ```bash
  ffmpeg -f image2 -framerate 12 -i foo-%03d.jpeg -s WxH foo.avi
  ```

  语法 `foo-%03d.jpeg` 指定使用由三个数字填充零组成的十进制数来表示序列号。它与 C printf 函数支持的语法相同，但只有接受普通整数的格式才是合适的。

  导入图像序列时，-i 还支持通过选择特定于 image2 的 `-pattern_type glob` 选项在内部扩展类似 shell 的通配符模式（通配符）。

  例如，从匹配 glob 模式 foo-*.jpeg 的文件名创建视频：

  ```bash
  ffmpeg -f image2 -pattern_type glob -framerate 12 -i 'foo-*.jpeg' -s WxH foo.avi
  ```

- 您可以在输出中放置许多相同类型的流:

  ```bash
  ffmpeg -i test1.avi -i test2.avi -map 1:1 -map 1:0 -map 0:1 -map 0:0 -c copy -y test12.nut
  ```

  生成的输出文件 test12.nut 将以相反的顺序包含来自输​​入文件的前四个流。

- 强制 CBR 视频输出：

  ```bash
  ffmpeg -i myfile.avi -b 4000k -minrate 4000k -maxrate 4000k -bufsize 1835k out.m2v
  ```

- 四个选项 lmin、lmax、mblmin 和 mblmax 使用 'lambda' 单位，但您可以使用 QP2LAMBDA 常数轻松地从 'q' 单位转换：

  ```bash
  ffmpeg -i src.ext -lmax 21*QP2LAMBDA dst.ext
  ```

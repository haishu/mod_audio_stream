# mod_audio_stream

**Production-grade WebSocket audio streaming for FreeSWITCH.**

Streams real-time audio between FreeSWITCH and external systems with correct lifecycle management, thread safety and predictable memory usage.

***This module supports bi-directional audio streaming (see python example below).***

## About

- The purpose of `mod_audio_stream` was to provide a simple, low-dependency yet effective module for streaming audio and receiving responses from a websocket server.

## Installation

### Dependencies

It requires `libfreeswitch-dev`, `libssl-dev`, `zlib1g-dev`, `libevent-dev` and `libspeexdsp-dev` on Debian/Ubuntu which are regular packages for Freeswitch installation.

### Building

After cloning please execute: **git submodule init** and **git submodule update** to initialize the submodule.

#### Custom path

If you built FreeSWITCH from source, eq. install dir is /usr/local/freeswitch, add path to pkgconfig:

```shell
export PKG_CONFIG_PATH=/usr/local/freeswitch/lib/pkgconfig
```

To build the module, from the cloned repository:

```shell
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make
sudo make install
```

**TLS** is `OFF` by default. To build with TLS support add `-DUSE_TLS=ON` to cmake line.

#### DEB Package

To build DEB package after making the module:

```shell
cpack -G DEB
```

Debian package will be placed in root directory `_packages` folder.

## Scripted Build & Installation

```shell
sudo apt-get -y install git \
    && cd /usr/src/ \
    && git clone https://github.com/voxcom-us/mod_audio_stream.git \
    && cd mod_audio_stream \
    && sudo bash ./build-mod-audio-stream.sh
```

### Channel variables

The following channel variables can be used to fine tune websocket connection and also configure mod_audio_stream logging:

| Variable                               | Description                                             | Default |
| -------------------------------------- | ------------------------------------------------------- | ------- |
| STREAM_MESSAGE_DEFLATE                 | true or 1, disables per message deflate                 | off     |
| STREAM_HEART_BEAT                      | number of seconds, interval to send the heart beat      | off     |
| STREAM_SUPPRESS_LOG                    | true or 1, suppresses printing to log                   | off     |
| STREAM_BUFFER_SIZE                     | buffer duration in milliseconds, divisible by 20, max 500 | 20      |
| STREAM_EXTRA_HEADERS                   | JSON object for additional headers in string format     | none    |
| ~~STREAM_NO_RECONNECT~~                    | true or 1, disables automatic websocket reconnection    | off     |
| STREAM_TLS_CA_FILE                     | CA cert or bundle, or the special values SYSTEM or NONE | SYSTEM  |
| STREAM_TLS_KEY_FILE                    | optional client key for WSS connections                 | none    |
| STREAM_TLS_CERT_FILE                   | optional client cert for WSS connections                | none    |
| STREAM_TLS_DISABLE_HOSTNAME_VALIDATION | true or 1 disable hostname check in WSS connections     | false   |

- Per message deflate compression option is enabled by default. It can lead to a very nice bandwidth savings. To disable it set the channel var to `true|1`.
- Heart beat, sent every xx seconds when there is no traffic to make sure that load balancers do not kill an idle connection.
- Suppress parameter is omitted by default(false). All the responses from websocket server will be printed to the log. Not to flood the log you can suppress it by setting the value to `true|1`. Events are fired still, it only affects printing to the log.
- `Buffer Size` actually represents a duration of audio chunk sent to websocket. If you want to send e.g. 100ms audio packets to your ws endpoint
you would set this variable to 100. If ommited, default packet size of 20ms will be sent as grabbed from the audio channel (which is default FreeSWITCH frame size). The maximum accepted value is 500ms.
- Extra headers should be a JSON object with key-value pairs representing additional HTTP headers. Each key should be a header name, and its corresponding value should be a string.

  ```json
  {
      "Header1": "Value1",
      "Header2": "Value2",
      "Header3": "Value3"
  }
  ```
- ~~Websocket automatic reconnection is on by default. To disable it set this channel variable to true or 1.~~

  - libwsc does not support automatic reconnection.
- TLS (for WSS) options can be fine tuned with the `STREAM_TLS_*` channel variables:
  - `STREAM_TLS_CA_FILE` the ca certificate (or certificate bundle) file. By default is `SYSTEM` which means use the system defaults.
Can be `NONE` which result in no peer verification.
  - `STREAM_TLS_CERT_FILE` optional client tls certificate file sent to the server.
  - `STREAM_TLS_KEY_FILE` optional client tls key file for the given certificate.
  - `STREAM_TLS_DISABLE_HOSTNAME_VALIDATION` if `true`, disables the check of the hostname against the peer server certificate.
Defaults to `false`, which enforces hostname match with the peer certificate.

### ASR/TTS 配置说明

`mod_audio_stream` 常用于把 FreeSWITCH 通话音频实时送到 ASR 服务，并接收 TTS 服务返回的音频进行回放。配置时重点关注音频分片大小、日志量、心跳和 TLS 连接参数。

| 变量 | 说明 | 建议 |
| ---- | ---- | ---- |
| `STREAM_BUFFER_SIZE` | 控制发送到 WebSocket 的音频分片时长，单位毫秒。必须是 20 的整数倍，默认 20，最大 500。 | 实时 ASR/TTS 建议 20-100；网络抖动较大时可调到 100-300；不建议长期使用接近 500 的值，以免对话延迟变高。 |
| `STREAM_HEART_BEAT` | WebSocket 空闲时发送心跳的间隔，单位秒。 | 经过负载均衡或网关时建议设置，例如 15 或 30。 |
| `STREAM_SUPPRESS_LOG` | 设置为 `true` 或 `1` 后减少 WebSocket 响应日志输出。 | 生产环境建议开启，避免 ASR/TTS 高频消息刷屏。排查问题时可临时关闭。 |
| `STREAM_MESSAGE_DEFLATE` | 设置为 `true` 或 `1` 后禁用 per-message deflate。 | 低带宽场景保持默认压缩；如果服务端兼容性不好，可禁用。 |
| `STREAM_EXTRA_HEADERS` | 以 JSON 字符串形式追加 WebSocket 握手 HTTP Header。 | 常用于传递鉴权信息、租户 ID、trace ID。 |
| `STREAM_TLS_CA_FILE` | WSS 连接使用的 CA 证书路径，也可为 `SYSTEM` 或 `NONE`。 | 生产环境建议使用 `SYSTEM` 或指定 CA 文件；仅测试环境使用 `NONE`。 |
| `STREAM_TLS_CERT_FILE` / `STREAM_TLS_KEY_FILE` | WSS 双向 TLS 时使用的客户端证书和私钥。 | 只有 ASR/TTS 网关要求 mTLS 时配置。 |
| `STREAM_TLS_DISABLE_HOSTNAME_VALIDATION` | 设置为 `true` 或 `1` 后跳过证书主机名校验。 | 仅测试环境使用，生产环境保持默认 `false`。 |

缓冲区过小会降低端到端延迟，但对网络抖动更敏感；缓冲区过大会让 ASR 识别和 TTS 回放更稳定一些，但会增加对话等待感。当前实现会把超过 500ms 的 `STREAM_BUFFER_SIZE` 自动限制为 500ms，并写入 warning 日志。

模块在流结束清理时会输出丢包统计，格式类似：

```text
stream stats: upstream_dropped_packets=0 upstream_dropped_bytes=0 downstream_dropped_packets=0 downstream_dropped_bytes=0
```

其中 `upstream` 表示送往 ASR 的上行音频，`downstream` 表示从 TTS 回灌到 FreeSWITCH 的下行音频。如果这些值持续增长，通常说明本地缓冲区被填满、WebSocket 服务端处理变慢、网络抖动较大，或 TTS 回放注入速度跟不上通话媒体时钟。

## API

### Commands

The freeswitch module exposes the following API commands:

```shell
uuid_audio_stream <uuid> start <wss-url> <mix-type> <sampling-rate> <metadata>
```

Attaches a media bug and starts streaming audio (in L16 format) to the websocket server. FS default is 8k. If sampling-rate is other than 8k it will be resampled.

- `uuid` - Freeswitch channel unique id
- `wss-url` - websocket url `ws://` or `wss://`
- `mix-type` - choice of
  - "mono" - single channel containing caller's audio
  - "mixed" - single channel containing both caller and callee audio
  - "stereo" - two channels with caller audio in one and callee audio in the other.
- `sampling-rate` - choice of
  - "8k" = 8000 Hz sample rate will be generated
  - "16k" = 16000 Hz sample rate will be generated
- `metadata` - (optional) a valid `utf-8` text to send. It will be sent the first before audio streaming starts.

```shell
uuid_audio_stream <uuid> send_text <metadata>
```

Sends a text to the websocket server. Requires a valid `utf-8` text.

```shell
uuid_audio_stream <uuid> stop <metadata>
```

Stops audio stream and closes websocket connection. If _metadata* is provided it will be sent before the connection is closed.

```shell
uuid_audio_stream <uuid> pause
```

Pauses audio stream

```shell
uuid_audio_stream <uuid> resume
```

Resumes audio stream

## Events

Module will generate the following event types:

- `mod_audio_stream::json`
- `mod_audio_stream::connect`
- `mod_audio_stream::disconnect`
- `mod_audio_stream::error`
- `mod_audio_stream::play`

### response

Message received from websocket endpoint. Json expected, but it contains whatever the websocket server's response is.

#### Freeswitch event generated

**Name**: mod_audio_stream::json
**Body**: WebSocket server response

### connect

Successfully connected to websocket server.

#### Freeswitch event generated

**Name**: mod_audio_stream::connect
**Body**: JSON

```json
{
 "status": "connected"
}
```

### disconnect

Disconnected from websocket server.

#### Freeswitch event generated

**Name**: mod_audio_stream::disconnect
**Body**: JSON

```json
{
 "status": "disconnected",
 "message": {
  "code": 1000,
  "reason": "Normal closure"
 }
}
```

- code: `<int>`
- reason: `<string>`

### error

There is an error with the connection. Multiple fields will be available on the event to describe the error.

#### Freeswitch event generated

**Name**: mod_audio_stream::error
**Body**: JSON

```json
{
 "status": "error",
 "message": {
  "code": 1,
  "error": "String explaining the error"
 }
}
```

- code: `<int>`
- error: `<string>`

#### Possible `code` values

| Code | Enum Name             | Meaning                                              |
|:----:|:----------------------|:-----------------------------------------------------|
| 1    | `IO`                  | I/O error when reading/writing sockets               |
| 2    | `INVALID_HEADER`      | Server sent a malformed WebSocket header             |
| 3    | `SERVER_MASKED`       | Server frames were masked (not allowed by spec)      |
| 4    | `NOT_SUPPORTED`       | Requested feature (e.g. extension) not supported     |
| 5    | `PING_TIMEOUT`        | No PONG received within timeout                      |
| 6    | `CONNECT_FAILED`      | TCP connection or DNS lookup failed                  |
| 7    | `TLS_INIT_FAILED`     | Couldn't initialize SSL/TLS context                  |
| 8    | `SSL_HANDSHAKE_FAILED`| SSL/TLS handshake with server failed                 |
| 9    | `SSL_ERROR`           | Generic OpenSSL error (certificate, cipher, etc.)    |
| 10   | `TIMEOUT`             | Timeout                                              |
| 11   | `PROTOCOL`            | WebSocket protocol error                             |

### play

**Name**: mod_audio_stream::play
**Body**: JSON

Websocket server may return JSON object containing base64 encoded audio to be played by the user. To use this feature, response must follow the format:

```json
{
  "type": "streamAudio",
  "data": {
    "audioDataType": "raw",
    "sampleRate": 8000,
    "audioData": "base64 encoded audio"
  }
}
```

- audioDataType: `<raw|wav|mp3|ogg>`

Event generated by the module (subclass: _mod_audio_stream::play_) will be the same as the `data` element with the **file** added to it representing filePath:

```json
{
  "audioDataType": "raw",
  "sampleRate": 8000,
  "file": "/path/to/the/file"
}
```

If printing to the log is not suppressed, `response` printed to the console will look the same as the event. The original response containing base64 encoded audio is replaced because it can be quite huge.

All the files generated by this feature will reside at the temp directory and will be deleted when the session is closed.

## Example

See `docker/README.md` for complete working setup for testing bi-directional audio.

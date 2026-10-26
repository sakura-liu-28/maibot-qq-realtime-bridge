# MaiBot QQ 实时语音桥

将 QQ 原生语音通话接入 MaiBot 或兼容的语音客户端。通过 NapCat 插件和独立 AV Host 调用 QQ AVSDK，再用 PulseAudio 接收对方声音、播放机器人语音。

本仓库是 [ClaudiaGardner/maibot-qq-voice-call](https://github.com/ClaudiaGardner/maibot-qq-voice-call) 中桥接组件的修改版，基于提交 `22f30c021cd3170f75af9dff66cc959a07ebda4b`。保留原作者署名，使用 **GPL-3.0-only**。来源及修改见 [NOTICE](NOTICE)。

## 这个版本的行为

- **不自动接听**：由语音客户端鉴权调用接听接口。
- **来电白名单**：仅接听 `allowedCallers` 中的 QQ；空名单拒绝所有来电。
- **核对当前来电**：接听必须携带当前 `inviteAt` 和 `callerUin`，防止误接已经变化的来电。
- **本机服务与鉴权**：控制服务和 AV Host 仅接受回环地址；除最小存活检查外，接口均要求至少 32 字节的 Bearer Token。
- **初始化保护**：过滤登录后五秒内的重复 SDK 初始化回调，防止反复登录。
- **可回退安装**：备份 QQ Loader；卸载时检查标记后恢复。

本仓库包含 AV 桥，**不包含 MaiBot 本体、聊天模型、ASR/TTS 服务或完整的 AI 通话客户端**。需要由 MaiBot/自有客户端把声音送入识别与对话服务，再把合成声音播放回 QQ。

## 环境

Linux、Node.js 22+、Python 3、完整 Linux QQ（含 `libAVSDKPlugin.so`）、兼容版本的 NapCat。

在 Debian/Ubuntu 上安装音频及显示依赖：

```bash
sudo apt-get install pulseaudio pulseaudio-utils xvfb xauth curl python3
```

本修改版实际接通记录对应 **Linux QQ 3.2.34-53644 / NapCat 4.18.41 / Linux x86_64**。其他版本需重新检查内部方法兼容性。普通单元测试无法代替真实 QQ 双向语音验证。

## 安装

```bash
git clone https://github.com/sakura-liu-28/maibot-qq-realtime-bridge.git
cd maibot-qq-realtime-bridge

bash scripts/install.sh \
  --qq-dir /opt/QQ \
  --napcat-dir /opt/QQ/resources/app/app_launcher/napcat \
  --check

bash scripts/install.sh \
  --qq-dir /opt/QQ \
  --napcat-dir /opt/QQ/resources/app/app_launcher/napcat
```

`--napcat-dir` 必须是本机真实存在、包含 `napcat.mjs` 和 `plugins/` 的目录；不要直接照抄路径。修改 QQ Loader 和插件目录需要相应的文件权限。安装器不关闭 QQ 的登录验证。

默认安装至 `$HOME/.local/share/maibot-qq-voice-call`；可用 `--install-dir` 指定目录。随机控制 Token 保存至安装目录的 `runtime/control.token`，权限为 `0600`，不要提交到 Git。

安装之后，编辑 NapCat 插件目录中的 `bridge-config.json`，**保留安装器生成的已有字段**，增加允许来电的 QQ 列表：

```json
{
  "controlHost": "127.0.0.1",
  "controlPort": 6110,
  "avHost": "127.0.0.1",
  "avPort": 6111,
  "tokenFile": "/your/install-dir/runtime/control.token",
  "allowedCallers": ["YOUR_ALLOWED_QQ_NUMBER"]
}
```

把示例中的路径和 QQ 占位符换为自己的值；只有数字 QQ 号会被接受。安装器再次执行会重写插件配置，需重新核对 `allowedCallers`。

NapCat 某些版本还会按内置名单限制插件加载。`compat/allow-reviewed-plugin.py` 仅适配上述固定版本的四项原名单，备份后只增加本插件 ID；结构变化时停止。需要先自行检查该修改和对应版本，再决定是否使用；它不会关闭整个插件加载检查。

## 启动

```bash
MAIBOT_QQ_CALL_BOT_UIN="YOUR_BOT_QQ_NUMBER" \
  /your/install-dir/scripts/run-napcat.sh
```

启动脚本配置隔离的 PulseAudio 和 AV Host。需确保普通 QQ/NapCat 已完成自己的登录。可用环境变量覆盖安装目录、QQ/NapCat 目录、Token 文件及端口，详见各脚本与 [PROTOCOL.md](PROTOCOL.md)。服务监听地址只能是 `127.0.0.1`、`::1` 或 `localhost`。

音频设备：

| 设备 | 用途 |
| --- | --- |
| `maibot_qq_speaker.monitor` | 读取来电方声音；单声道 PCM S16LE 16 kHz |
| `maibot_qq_mic` | 播放机器人声音；单声道 PCM S16LE 24 kHz |
| `maibot_qq_mic_source` | QQ 的默认麦克风 source |

## 客户端对接

1. 使用本地 Token 请求 `GET http://127.0.0.1:6110/v1/calls/current`。
2. `phase=ringing` 时核对自己的接听策略，再鉴权调用 `POST /v1/calls/accept`：

```json
{"inviteAt":"从当前来电响应中读取","callerUin":"从当前来电响应中读取"}
```

3. 只有 `phase=connected` 才开始按上表采集和播放音频。
4. 来电结束或权限变化时停止音频流，并由客户端自行结束模型会话。

所有控制接口使用 `Authorization: Bearer <control.token内容>`。不要把这些内部控制端口开放到公网。`GET /healthz` 只表示进程存活，不代表 AVSDK 或 QQ 已登录就绪。

## 检查与卸载

```bash
npm test
/your/install-dir/scripts/doctor.sh
/your/install-dir/scripts/uninstall.sh
```

测试覆盖原生 Accept 参数映射、配置边界、接口鉴权、诊断字段脱敏和 SDK 初始化回调。卸载脚本默认保留运行目录与 Token；具体行为以 `--help` 为准。

仓库不分发 QQ/NapCat/AVSDK 二进制，不附带登录态、QQ 好友、聊天记忆、真实账号或 API 密钥。

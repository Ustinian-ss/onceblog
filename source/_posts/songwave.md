---
title: SongWave 声浪：用 Electron + 原生 JavaScript 写一个桌面音乐播放器
date: 2026-09-27 10:00:00
categories:
  - 开源项目
tags:
  - Electron
  - JavaScript
  - Web Audio
  - 开源
---
> 在线搜歌能放、播放失败自动换源、能挂 Wallpaper Engine 壁纸、还带 10 段 EQ 和逐字桌面歌词 —— 一个不依赖任何第三方播放器桌面端的独立实现。本文记录它的架构分层、几个关键实现，以及为什么最终做到了「不用网络、不启动 Electron 也能跑完 338 条断言」。

<!--more-->

## 一、为什么不用前端框架

播放器是一个长驻窗口，状态高度集中（当前歌曲、播放列表、进度、音量），UI 更新点也是可枚举的。引入 React/Vue 带来的收益，抵消不了构建链和调试成本的增加。所以最终选择 **Electron + 原生 JavaScript**，只依赖 `electron` 和 `electron-builder` 两个开发依赖。

代价是要自己管 DOM 更新。但因为更新点收敛在播放核心周围，实际并没有变成负担。

## 二、三层架构

```
┌──────────── 渲染层 app/ ────────────┐
│ index.html + renderer.js  播放/曲库/歌词/背景/音效 UI │
│ engine.js                 SongLife 可视化引擎        │
└──────────────┬──────────────────────┘
               │ contextBridge（preload.js，不开 nodeIntegration）
┌──────────────▼──── 主进程 electron/ ─┐
│ main.js  窗口 / 桌面层壁纸窗 / 全局快捷键 / 下载 / IPC │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────── 音源层 src/ ──┐
│ sources/  网易云直链、四平台搜索、脚本仓库、vm 沙箱运行时 │
│ match.js  换源匹配打分（纯函数）                        │
│ download.js / wallpaper-engine.js / audio-effects.js  │
└───────────────────────────────────────┘
```

最值得说的一点：**音源层是纯 Node 模块，不 import 任何 Electron API**。这个约束带来两个好处 —— 它可以脱离 Electron 单独运行，也能在测试里直接调用。后面测试体系能站得住，根子在这里。

安全上，`preload.js` 通过 `contextBridge` 暴露能力，渲染进程不开 `nodeIntegration`。

## 三、关键实现

### 1. 播放失败自动换源

版权受限、VIP 锁、取链超时都会导致播放失败。做法是拿「歌名 + 歌手」去其它平台找同一首歌，但难点在于**判断找到的是不是同一首**。

匹配用纯函数打分：

- 歌名归一化（去括号后缀、繁简统一、全半角统一）
- 歌手名比对（多歌手拆分后取交集）
- 时长差加权（差值越大扣分越多）

实测同曲同歌手可以打到 **138 分**，异曲只有 **18 分**，阈值一卡就能自动拒绝。换源结果会写回播放列表，失败时状态栏逐平台显示原因，比如：

```
换源失败：酷狗（版权限制）、酷我（版权限制）
```

### 2. 音源脚本沙箱

社区音源脚本格式各异，不能直接 `require` 进来跑。方案是放进**独立的 Node `vm` 沙箱**，由沙箱提供脚本需要的 ABI：

| ABI | 说明 |
|---|---|
| `EVENT_NAMES` | `{ request, inited, updateAlert }` |
| `send(EVENT_NAMES.inited, caps)` | 声明支持的平台与动作 |
| `on(EVENT_NAMES.request, handler)` | 注册请求处理器，返回 Promise |
| `request(url, opts, cb)` | 沙箱内 HTTP 请求，响应体自动解析 JSON |
| `utils` | `crypto.md5/aesEncrypt/rsaEncrypt`、`buffer`、`zlib` |

一个最小的取链脚本长这样：

```js
const { EVENT_NAMES, on, send, request } = globalThis.lx;

send(EVENT_NAMES.inited, {
  sources: {
    kw: { name: '酷我', type: 'music', actions: ['musicUrl'], qualitys: ['128k'] },
  },
});

on(EVENT_NAMES.request, ({ source, action, info }) => {
  if (action !== 'musicUrl') return Promise.reject(new Error('不支持的动作'));
  const id = info.musicInfo.songmid || info.musicInfo.id;
  return new Promise((resolve, reject) => {
    request('https://example.com/api/url?id=' + id, { method: 'get' }, (err, resp, body) => {
      if (err) return reject(err);
      if (!body || body.code !== 0) return reject(new Error('取链失败'));
      resolve({ url: body.data.url });
    });
  });
});
```

脚本**无需任何修改**即可运行。播放器会探测它声明了哪些平台、支持哪些动作，再决定搜索走内置接口、取链走脚本。

### 3. 音频链路

```
<audio> ──MediaElementSource──▶ 10 段 EQ ──▶ 混响 ──▶ 输出
                  └──▶ AnalyserNode ──▶ 可视化引擎
```

10 段 EQ 覆盖 31Hz ~ 16kHz ±12dB，混响用**卷积混响**（IR 合成）而不是简单的延迟叠加。

这里踩过一个坑：**系统回环抓取来的音频无法被 Web Audio 处理**。所以一旦开启音效，播放链路会自动从「系统回环」切到「直连模式」，并在状态栏提示用户 —— 否则用户会以为是音效没生效。

### 4. 简繁转换的「一简对多繁」

歌词简繁转换不是查表就能解决的：`发` 可能对应 `發`（发送）或 `髮`（头发），`后` 对应 `後` 或 `后`（皇后）。方案是 **1177 字映射表 + 38 组词组优先规则**，先按词组消歧，再兜底单字映射。

## 四、测试：338 条断言，全离线

```
npm run test:all     # 12 套 / 338 条断言，不需要网络，也不需要启动 Electron
```

| 测试 | 覆盖内容 | 断言 |
|---|---|---|
| `test:renderer` | 渲染层端到端（最小 DOM 桩）：搜索、播放、歌词、音效、换源、拖拽 | 112 |
| `test:download` | 真实 HTTP 服务器：流式下载 / 302 重定向 / 取消 / 非法地址 | 21 |
| `test:srcmgr` | 音源仓库：导入 / 启停 / 删除 / 持久化 / 端到端取链 | 24 |
| `test:match` | 换源匹配打分与选源 | 16 |
| `test:extras` | 命名模板 / ID3v2 标签 / 并发队列、简繁转换、排行榜解析 | 44 |

能离线跑的关键是前面那条架构约束：音源层是纯 Node 模块，网络请求可以 mock；渲染层用最小 DOM 桩就能驱动交互逻辑。下载那套甚至起了一个**真实本地 HTTP 服务器**来验证 302 跟随和取消语义。

## 五、两个坦白

**壁纸层不是真正的桌面层。** Windows 上想真的压到桌面图标下面，需要原生挂载 Explorer 的 `WorkerW`（Wallpaper Engine 的做法）。本项目用的是「置底窗口」方案，观感接近但不等价，README 里也写明了。

**壁纸层关不掉。** 因为它点击穿透、无法聚焦，所以不能靠点它退出，最后加了全局快捷键 `Ctrl+Alt+W` 兜底 —— 这类问题只有真机上才会暴露。

## 六、仓库

- 源码：<https://github.com/Ustinian-ss/songwave>
- 可视化引擎另开源为：[Song-Life](https://github.com/Ustinian-ss/Song-Life)
- 许可证：MIT（在线音源仅供个人学习试听，请尊重版权；项目不含任何绕过付费/版权保护的手段）

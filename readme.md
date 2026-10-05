# PenMusic —— 本地音乐播放器（miniapp）

有道词典笔 miniapp，基于 HaaS UI / falcon-ui（Vue 2.6.12）构建的**本地音乐播放器**。

> ⚠️ **文档修复说明**
> 本文档在 2026-10-01 被蓝色大肥鱼「deepseek」一次误操作损坏过，已用备份 + 自动恢复修回绝大部分。
> 事故原因：用 PowerShell 对 `ui/README.md` 做了 `Get-Content -Raw | Set-Content -Encoding UTF8` 的往返 —— UTF-8 被按系统 ANSI（GBK）解码，中文全变乱码，**每个对不上的字节都永久丢失**（连带着吃掉一些换行，少数标题、列表项被并到上一行）。
> 讽刺的是，这份文档里本来就写着「别用 PowerShell 改源码」（见「四、构建」下面那条）。
> 恢复方式（脚本都在 `.tools_tmp/`，一条命令 `node .tools_tmp/rebuild-readme.js` 重跑全流程）：
> 1. `merge-readme-backup.js`：用仓库根目录那份 9/25 的备份**逐字还原**对得上的小节；
> 2. `fix-readme2.js`：对备份没有覆盖的部分，按字节层扫描出不完整序列，用**文档自身的 n-gram** 判定丢掉的字节（例如「下〔缺〕载」能拼出文档里出现过的「下载」→ 填「载」）；
> 3. `splice-section20.js` / `splice-blocks.js`：把本次会话写的原文逐字还原。
>
> 仍有个别位置没能定论，用**占位方块**（Unicode U+3013）标着，在编辑器里搜这个字符就能列出来 —— 那些是**故意不猜**的地方，宁可留白也不填错。

---

## 一、功能

| 功能 | 说明 |
|---|---|
| 本地曲库扫描 | 递归扫描音乐目录，支持 mp3 / flac / wav / m4a / aac / ogg / opus / ape / wma / amr 等 |
| 播放控制 | 播放 / 暂停 / 上一首 / 下一首 / 停止 |
| 四种播放模式 | 列表循环、顺序播放、单曲循环、随机播放（顶栏一键轮换） |
| 进度调整 | 拖动进度条跳转，显示「当前时间 / 总时长」 |
| 倍速播放 | 0.5x / 0.75x / 1x / 1.25x / 1.5x / 2x，偏好自动记忆 |
| 蓝牙输出 | 连上蓝牙音箱/耳机后自动把声音送到蓝牙通路（不指定就只剩静音，见「关键设计 10」） |
| 歌词显示 | 自动查找同名 `.lrc`，支持多时间标签、翻译行、`[offset:]`；逐行高亮跟随，**长句自动换行完整显示（不截断）**，**可手指上下滑动浏览、松手回到当前行** |
| 卡拉OK逐字歌词 | 支持 QRC/YRC/KRC/增强型 LRC/TTML 字时间轴，词内逐字母扫光动画（见 20.18，默认关） |
| 歌曲封面 | 自动查找同名 `.jpg/.jpeg/.png/.webp/.bmp`；没有封面时显示唱片占位图 |
| 多种排序 | 名称 A→Z / Z→A、时间 新→旧 / 旧→新、体积 大→小 / 小→大、时长 长→短 / 短→长、默认顺序 |
| 搜索筛选 | 调起键盘输入关键字，按歌名 / 歌手 / 文件名过滤 |
| MusicFree 在线音源 | 实验室开关，装音源插件、在线搜索、在线播放、歌单导入、批量下载（见 20） |
| 视频模式（实验室） | 扫描视频文件，交给系统播放器（gst-play）走 Wayland/KMS（见 18） |
| 蓝牙耳键 | 耳机播放/暂停按键（条件支持，见 20.17） |
| 自适应屏幕 | 读取真实分辨率，自动横竖屏纠正 + 等比缩放，不会溢出或留白 |
| 系统键盘桥接 | 桥接固件系统键盘；固件不支持时自动降级为**内置键盘** |
| 诊断页 | 一键查看真实分辨率、框架版本、各项原生能力是否可用 |

---

## 二、目录结构

```
ui/
├── package.json          # appid（8001749598192572）/ appName / 版本 / quickjs 配置
├── app_icon.png          # 256x256 应用图标
└── src/
    ├── app.js            # 应用入口（沿用模板，未修改）
    ├── base-page.js      # 页面基类（沿用模板，未修改）
    ├── app.json          # 路由表：index / page / player / softKeyboard / settings / about / video / sheets
    ├── @types/falcon.d.ts
    ├── utils/
    │   ├── screen.js     # 屏幕自适应：分辨率探测 + 缩放 + screenMixin
    │   ├── format.js     # 时间 / 体积 / 日期 / 文件名解析
    │   ├── touch.js      # 触摸坐标提取（多字段兜底，单份实现）
    │   └── emitter.js    # 极简事件发射器
    ├── services/
    │   ├── platform.js   # 平台能力探测与适配（系统模块 fs / global + $falcon.soundPlayer）
    │   ├── store.js      # 存储适配（兼容三种 jsapi.storage 形态 + 文件兜底）
    │   ├── library.js    # 曲库扫描、封面/歌词查找、排序、筛选、时长缓存
    │   ├── lyrics.js     # LRC 解析与定位（含逐字分支与同时间戳译文配对）
    │   ├── wordLyrics.js # 逐字歌词解析：QRC/YRC/KRC/增强型 LRC/TTML
    │   ├── player.js     # 播放器单例（状态唯一真源 + 250ms 轮询）
    │   ├── keyboard.js   # 系统键盘桥接（跨页 Promise）
    │   ├── mediaKeys.js  # 蓝牙耳机按键（播放/暂停）应用级绑定
    │   ├── probe.js      # exec 返回形状归一（字符串 / 对象两种都认）
    │   ├── video.js      # 系统播放器命令构造（纯函数）
    │   └── musicfree.js  # MusicFree 模式：插件管理 + 搜索 + 在线播放（见 20）
    ├── adapter/          # 音源插件运行时·适配层（见 19）
    │   ├── http.js       # 网络阶梯：custom.Fetch(异步+轮询) → 同步 → 系统 http
    │   ├── axios.js      # 给插件用的 axios 外观（JSON 自动解析 / params / 非 2xx 抛错；实例本身可调用）
    │   ├── storage.js    # 同步读写的 KV（写防抖 300ms，落在 services/store.js 上）
    │   └── require.js    # 沙箱 require 白名单：axios / crypto-js / qs / he / dayjs
    ├── core/             # 音源插件运行时·核心层（见 19）
    │   ├── plugin.js        # 插件沙箱 + 方法包装 + 五级取址回退 + readIsEnd
    │   ├── pluginManager.js # 安装 / 卸载 / 更新 / 启用 / 排序 / 用户变量 / 歌单导入
    │   ├── lrcParser.js     # 插件歌词解析（转发 services/lyrics.js 并补翻译）
    │   └── trackQueue.js    # 播放队列（两种交接方式，见 19.7）
    ├── vendor/           # 自带依赖（不许有 ?. / ??）
    │   ├── hash.js       # 纯 JS md5 / sha256（含字节级入口，供 HMAC 用）
    │   ├── qs.js         # 19 KB，MIT
    │   └── he.js         # 100 KB，MIT（体积大头，见 19.8）
    ├── components/
    │   ├── SongRow.vue       # 曲目行
    │   ├── NowPlayingBar.vue # 底部迷你播放条
    │   ├── CoverArt.vue      # 封面（含唱片占位图）
    │   ├── LyricPanel.vue    # 歌词窗口（居中高亮 + 上下文 + 逐字调度）
    │   ├── KaraokeLine.vue   # 逐字歌词单行（私有时钟 + 词内扫光）
    │   └── OptionSheet.vue   # 通用选择面板（排序 / 倍速 / 目录）
    ├── pages/
    │   ├── index/index.vue          # 首页：曲库列表 / MusicFree 在线音源 / 视频模式
    │   ├── player/player.vue        # 播放页：封面 + 歌词 + 控制
    │   ├── softKeyboard/softKeyboard.vue  # 键盘页（系统键盘 / 内置键盘）
    │   ├── settings/settings.vue    # 设置页
    │   ├── about/about.vue          # 关于页
    │   ├── video/video.vue          # 视频页（实验室）
    │   ├── sheets/sheets.vue        # 歌单页
    │   └── page/page.vue            # 诊断页
    └── styles/
        ├── var.less      # 配色变量
        ├── layout.less   # 页面骨架（首页/设置/关于共用）
        └── base.less     # 视觉令牌（颜色/边框/圆角/flex）
```

---

## 三、关键设计

### 1. 屏幕自适应（`utils/screen.js`）

- 从 `$falcon.env.screenWidth / screenHeight` 读取真实分辨率；
- 若读到竖屏数值（高 > 宽）**自动交换**，兼容「屏幕 260×640、默认旋转 270°」的机型；
- 读取失败时退回 640×260；
- `scale = clamp(min(宽/640, 高/260), 0.8, 2.4)`：取宽高比例中较小者，保证内容永不溢出屏幕高度；屏幕更宽时由 flex 拉伸填充。
- 页面根容器直接设为 `宽 × 高` 像素，因此**任何分辨率下都恰好铺满屏幕**；
- **约定：`styles/*.less` 只写颜色 / 边框 / 圆角 / flex 方向，所有尺寸与字号一律通过 `:style` 内联下发并用 `px()` 缩放**，这样字号也会随屏幕等比变化。

### 2. 平台能力适配与降级（`services/platform.js`）

词典笔运行时（WalOS / HaaS UI）**内置**了若干系统模块，任何 miniapp 都能直接 `import`，**不需要自己交叉编译 `.so`**（这一点很关键，早期版本曾误以为需要自带原生模块）：

| 能力 | 提供方 | 说明 | 不可用时 |
|---|---|---|---|
| 文件系统 | ① 自带私有 jsapi **`custom.FileShell`**；② 系统模块 `fs`（或 `file`/`fileSystem`/`io`）；③ `global.execShell` 兜底 | 自带模块用 POSIX `opendir/stat`；系统模块用 `readdir(path,{withFileTypes:true})`；兜底用 `ls -1Ap` + `find` | 退回上次扫描缓存，界面提示 |
| 音频播放 | ① 自带私有 jsapi **`custom.AudioBridge`**；② `$falcon.soundPlayer`；③ `$falcon.jsapi.media/audio/...` | 自带模块用 `ffmpeg \| aplay` 管道（见下）；框架对象用 `play/pause/resume/stop/seek/getPosition/getDuration` | 仅浏览曲库，提示无播放能力 |
| 系统输入法 | 系统模块 `global` | `new Global().startTextEdit(JSON字符串)` + `textEditFinished` 回传 | 自动切换到内置键盘 |
| shell 命令 | ① 自带 `custom.FileShell.exec`；② `global.execShell` | 诊断与逃生口 | 相关功能降级 |

**`custom.AudioBridge` 为什么是 shell 管道而不是链接 ALSA**

实测设备（X3s）的音频栈完整 —— `rockchip,rk817-codec` 声卡、`/dev/snd/pcmC0D0p`、`libasound.so.2`、`ffmpeg 4.1.3`、`aplay`、`gst-launch-1.0` 全都在，但 **JS 侧一个音频接口都没有**，所以播放由自带 `.so` 承担：

```
ffmpeg -hide_banner -loglevel error [-ss <起点>] -i <文件> [-af atempo=<倍速>] -f wav - | aplay -q -
```

- 解码交给 `ffmpeg`（mp3 / flac / aac 通吃），输出交给 `aplay`；
- **不链接 ALSA / GStreamer 头文件**，因此 `.so` 的动态依赖仍然只有 `libpthread / libstdc++ / libm / libgcc_s / libc`；
- 进程管理用 `fork + execl`（不是 `popen`），这样才能拿到 pid 并随时终止；
- 暂停 / 拖动 / 变速都是「记录位置 → 重启进程 → 从该位置继续」，代价是每次操作约 0.1~0.3 秒的重启延迟；
- **倍速**由 ffmpeg 的 `atempo` 滤镜实现，0.5x~2.0x。

访问方式全部是**动态 `import()` + try/catch**，模块不存在只会降级，不会导致页面加载失败。同一能力在不同固件上方法名可能不同，适配层按候选名逐个探测（例如播放：`play` → `playAudio`；进度：`seek` → `seekAudio` → `setPosition`）。

**时间单位**：`$falcon.soundPlayer` 的 `seek` / `getPosition` / `getDuration` 统一为**秒**（与早期第三方封装的毫秒不同），适配层按秒处理，无需标定。

**其他几条平台约定**（都已按官方文档处理）：

- `<image>` 显示本地文件需要 `file://` 前缀 —— 见 `platform.js` 的 `toLocalUrl()`；
- 进度条要用 **`<seekbar>`**（属性 `:value`，事件 `@changing` / `@change`）；`<slider>` 在本平台是**轮播组件**，不是数值滑块；
- 支持的长度单位是 `px / pt / wx / rpx / vh / vw / %`（`utils/screen.js` 目前用 px 运行时缩放，也可以用 `vh` 直接写响应式尺寸）。

### 3. 播放器单例（`services/player.js`）

全应用播放状态的**唯一真源**：首页迷你条与播放页都只通过 `player.on('stateChanged', ...)` 订阅，从不直接触碰原生播放器。播放列表通过内存注入（`setPlaylist`），路由只传空参数，避免大对象过路由。

### 4. 键盘桥接（`services/keyboard.js` + `softKeyboard` 页）

```
调用方:  await openKeyboard({title, text})   ← 挂起一个 Promise
          → $falcon.navTo('softKeyboard', { config, hasNative })
键盘页:  系统键盘可用 → new Global().startTextEdit(json)
                       textEditFinished.on((uuid, json) => ...)
         系统键盘不可用 → 渲染内置键盘（字母/数字两套布局 + 大写切换）
收尾:    settleKeyboard({confirmed, text}) → 调用方 Promise resolve
          $page.finish() 返回原页面
```

用户直接按返回键离开键盘页时，`beforeDestroy` 会以「取消」结算，避免调用方的 Promise 永久挂起。

### 5. Weex 样式红线（已全部规避）

`falcon-styler` 会对样式逐条校验，以下写法会产生 ERROR 日志：

- ❌ 后代选择器 `.a .b`、标签选择器、`html` 等
- ❌ `border-radius: 0px 8px 8px 8px`（只接受单个数值或 px）
- ❌ `display: block`（只支持 `flex`）
- ❌ `overflow: scroll`（只支持 `visible` / `hidden`）
- ❌ `em` / `rem` 单位
- ❌ 渐变、滤镜、阴影高级写法、`:hover`

本项目：单类名选择器、`border-radius` 单值、`display: flex`、滚动一律用 `scroller` 组件、文字一律包在 `<text>` 内。（`tools/check-styler.js` 可复现上述校验器的报错行为。）

### 6. UI 设计系统与屏幕自适应

**自适应不依赖 `$falcon.env`。** 早期版本用 `env.screenWidth/Height` 算 px 并设置根容器尺寸，一旦 env 上报的分辨率与实际不符，整个 UI 就会错位。现在改成：

| 项 | 做法 |
|---|---|
| 根容器 | `100vw × 100vh`，永远等于屏幕 |
| 所有尺寸 | `vh`（见 `styles/var.less`），随屏幕高度自动缩放 |
| 需要数字型 px 的地方 | 仅 `<seekbar>` 的 `track-size` / `handle-size`（它们是数字参数不是样式），用 `utils/screen.js` 的换算 |
| 动态值 | 只用 `:style` 下发（颜色、百分比宽度）；布局与度量一律走 class |

**关键坑：`box-sizing`。** 本平台默认是 `content-box`，而 flex 列方向下子元素会被拉伸到父容器宽度 —— 此时左右 padding 会「加」在宽度之外，元素比父容器还宽，表现就是**整屏横向溢出、UI 整体偏移**。因此所有带 padding 的容器都显式写 `box-sizing: border-box`。

**`box-shadow` 的坑。** 校验器把值按空白拆开并要求恰好 4 项，而 LESS 会把 `rgba()` 重新格式化成带空格的形式（`rgba(0, 0, 0, 0.45)` 会被拆成 7 项）。解决办法是用 LESS 转义语法：`0px 2px 8px ~"rgba(0,0,0,0.45)"`。

设计令牌集中在 `styles/var.less`（颜色 / 字号 / 间距 / 圆角 / 结构高度），通用类在 `styles/base.less`（`page` / `topbar` / `card` / `pill` / `row` / `mask` / `toast`）。

### 7. 播放进程管理（暂停 / 拖动为什么会失败）

命令形如 `sh -c "ffmpeg ... | aplay ..."`。**如果只 `kill` sh 的 pid，`ffmpeg` 和 `aplay` 会变成孤儿进程继续出声** —— 表现就是「暂停后声音不停，恢复播放时两条音频叠加」。

修法：子进程在 `fork` 之后立刻 `setpgid(0, 0)` 自成进程组，父进程也 `setpgid(child, child)` 消除竞态，之后用 `kill(-pid, SIGKILL)` 一次终止整组；`waitpid` 回收后 `usleep(80ms)` 让 ALSA 释放设备，避免紧接着的 `aplay` 报 Device busy。

### 8. 歌词必须完整显示（变高度行）

**踩过的坑：`lines` 不是「最多显示几行」这么简单。** `aiot-vue-cli` 的预编译器（`falcon-vue-precompiler/src/components/text.js`）遇到 `lines: N` 会自动补上三件套：

```
overflow: hidden;  text-overflow: ellipsis;  -webkit-line-clamp: N;
```

所以早期 `.line-main { lines: 1; text-overflow: ellipsis; }` 会让每一句歌词都被截成 `Win it now! Be the sharp no one can i...`。**去掉 `lines` 就恢复自然换行**（平台的 `<text>` 默认不限行数，`lines` 才是主动加的限制）。

代价是老算法失效：原来「行高固定 = 行号 × 行高」，现在一行可能折成 2~4 行，行高不再统一。`LyricPanel.vue` 因此自己估算行数：

| 步骤 | 做法 |
|---|---|
| 可用宽度 | `面板宽 - 内边距 - 左侧竖条`，再乘安全系数 1.08（宁可早折行） |
| 折行规则 | 全角字符逐字折（字宽 ≈ 1em）；拉丁按「词」折（遇空格才断）；超长单词补行数 |
| 单行高度 | `行数 × line-height + 行间距 + 上下留白` |
| 居中 | `translateY(可视区中心 - 当前行之前累计高度 - 当前行高度/2)` |

两条硬约束：

1. `line-height` 必须与 `<script>` 里的 `LH_MAIN_VH` / `LH_TRANS_VH` **严格一致**（样式里写成 `(@fs-lg * 1.3)`，改字号时要同步改常量）；
2. 估算**宁可偏大**：盒子多留白只是行距松一点，估小了字会叠在一起。

**手动滚动（手指上下滑，松手回到当前行）。** `.viewport` 上挂了 `touchstart / touchmove / touchend / touchcancel`（写法与 `ProgressBar.vue` 完全一致，那套在真机上已验证可用）。要点：

| 变量 | 含义 |
|---|---|
| `autoOffset` | 自动跟随位移（把当前行顶到可视区中央） |
| `frozenOffset` / `frozenStart` | 手指按下瞬间冻结的位移与窗口起点 |
| `dragOffset` | 手指位移，松手后归零 |
| `dragging` | 手指按住中（1:1 跟手，动画 0ms） |
| `paused` | 已松手、但还在**延时停留**期（仍冻结，动画 300ms） |

- **按下即冻结**：拖动与停留期间歌曲唱到下一句也不动，否则手里的歌词会被突然拽走；窗口起点一起冻结，避免行列表整体平移导致跳一下。
- **松手后先停住 `RETURN_DELAY_MS`（3000ms）再滑回当前行**，留出看歌词的时间。停留期间再次按下会取消回位并重新冻结；只是点一下（位移 0）则立刻回位，不必等。
- **拖动中 1:1 跟手**：下发的 `transitionDuration` 置 `0`（后来改为 `1`，见 17.1），松手恢复 `300`（传**数字**，与静态样式 `"transitionDuration": 300` 类型一致 —— `:style` 绑定走运行时、不过校验器）。CSS 里的 `300ms` 必须与 `TRANSITION_MS` 常量同步。
- **夹紧不露白**：位移上限 `0`（顶部贴可视区顶部），下限 `可视区高 - 轨道总高`；内容比可视区短时直接不给拖。
- 窗口两侧各渲染 10 行，轨道比可视区高得多，实测可上下浏览约 3 倍可视区高度。
- `watch: lines` 在**换歌时立刻结束停留**：否则冻结的窗口起点可能落在新歌词范围之外，会出现一小段空白歌词；`beforeDestroy` 里清掉定时器。

### 9. 其它单行文本仍然用 `t-ellipsis`

歌曲名、歌手、状态行等**故意保持** `lines: 1` + 省略号 —— 那些位置多行会把布局撑坏。只有歌词面板不允许截断。

### 10. 蓝牙没有声音（输出设备选择）

**现象**：连上蓝牙音箱/耳机后，本应用**连机身扬声器都不响**，而系统自带播放器能正常走蓝牙。

**原因**：固件在蓝牙连上之后会关掉板载扬声器通路（rk817 那条），声音必须走系统的蓝牙通路。而我们的播放链路是 `ffmpeg ... -f wav - | aplay -q -`，`aplay` 不带 `-D` 就只用 ALSA 的 `default`（板载声卡），于是两边都哑。

**做法**：`AudioBridge` 在启动播放前解析出输出设备，给 aplay 加上 `-D <设备>`。

#### 关键：bluealsa 守护进程要应用自己启动

参考 PulseBox（`_amr_unpack/extracted/bluetoothAudio-*.js` + `libs/libjsapi_sixi_*.so` 的字符串）才知道这一步：**固件里装了 `/usr/bin/bluealsa`，但默认不启动它**，要应用自己拉起来。只做「挑设备名 → `aplay -D`」是不够的 —— 守护进程不在，蓝牙 PCM 根本打不开。

```
ps | grep bluealsa | grep -v grep >/dev/null || \
(/usr/bin/bluealsa --profile=a2dp-source --sbc-quality=0 >/tmp/mlp-bluealsa.log 2>&1 &)
```

`--profile=a2dp-source` = 本机作为 A2DP 发送端。启动后要重定向 stdout/stderr，否则 `popen` 的管道不会关闭、`pclose` 会一直等它退出。

#### 设备名用带 MAC 的精确形式

```
bluealsa:DEV=<MAC>,PROFILE=a2dp
```

MAC 从 BlueALSA 的 D-Bus 接口取（`Manager1.GetPCMs` 返回的对象路径里是 `/org/bluealsa/hci0/dev_AA_BB_CC_DD_EE_FF/a2dpsrc/sink`）。

**这里故意不用 PulseBox 的 `grep -o` 管道**：本机 busybox 的 grep 不支持 `-o`（当初时长一直读成 0 就是这个坑），所以原生模块直接读 `dbus-send` 的原始输出、在 C++ 里解析（`AudioOutput::parseBluealsaMac`）。

解析时有个坑：`a2dpsnk`（本机作接收端）的路径里**也含 "a2dp"**，必须先匹配更长的 `a2dpsrc`，否则第一条就把正确的顶掉了 —— 这条是被单测抓出来的。

#### 蓝牙通路要强制固定格式裸流

A2DP 走 SBC，bluealsa 的 PCM 基本不接受任意格式协商，喂 WAV 很可能直接被拒：

```
ffmpeg ... -ar 48000 -ac 2 -f s16le - | aplay -q -r 48000 -c 2 -f S16_LE -D '<设备>' -
```

`-ar` 与 `aplay -r` **必须同时是 48000**：前者决定 ffmpeg 吐什么，后者决定 aplay 按什么速率解释这段裸流 —— 只改一个，bluealsa 收到的就是被解释错速率的音频（音调不对）。48000 照 `icon/bt.sh` 对齐（该脚本在设备上验证通过，用 `gst-launch-1.0 ... caps="audio/x-raw, rate=48000, ..." ! alsasink` 走同一条 bluealsa PCM）。**不要用源文件采样率**：它随歌曲变，蓝牙侧反而更容易卡。

板载声卡照旧走 WAV（`AudioOutput::isBluetoothDeviceName` 区分）。

#### 设备选择优先级

| 优先级 | 来源 | 说明 |
|---|---|---|
| 0 | 设置里手动钉的设备 | `mlp_settings_v1.outputDevice`，诊断页试听后写入；**钉的是蓝牙设备而当前没蓝牙连着时作废** |
| 1 | D-Bus 拿到的 MAC | `bluealsa:DEV=<MAC>,PROFILE=a2dp` |
| 2 | `aplay -L` 里的蓝牙 PCM | 拿不到 MAC 时退回裸 `bluealsa` |
| 3 | `/proc/asound/cards` 里的蓝牙声卡 | 有些固件单独注册一张卡，返回 `plughw:N,0` |
| 4 | `pulse` PCM | PulseAudio 会跟随系统当前输出 |
| 5 | 都不匹配 | 空串 = 系统默认设备（与改动前行为一致） |

前置门：`ps | grep bluetoothd` —— 蓝牙栈没在跑就直接用默认设备，省掉几次 popen。这比 `bluetoothctl`/`hcitool` 靠谱，那两个工具很多固件根本不装。

#### 血的教训：光有「启动失败回退」不够，闸门必须是实时的

第一版只做了「挑设备 → `aplay -D`」+「启动失败自动回退」，结果**蓝牙一断，机身扬声器也哑了**（比修之前更糟）。原因：`launchVerified` 只在子进程**退出**时才回退，而 `aplay -D bluealsa:...` 在蓝牙没连着时很可能是**打开成功但一个字节都不送**（进程一直活着）—— 于是回退永远不触发，界面还在正常走进度，就是没声音。

现在改成：

| 位置 | 做法 |
|---|---|
| 总闸门 | `bluetoothAudioReady()`：BlueZ 明确报出 `Connected: true` 才允许走蓝牙 |
| 手动钉的设备 | 钉的是**蓝牙**设备、而当前没有蓝牙连着 → **这次不用它**（设置保留，蓝牙回来照样生效） |
| 探测缓存 | 缓存里是**非蓝牙**设备（默认设备）→ 直接复用，普通扬声器路径零额外开销；缓存里是**蓝牙**设备 → **每次启动都重新过一遍闸门**，蓝牙一没就作废 |
| bluealsa 守护进程 | 只有确认有设备连着才启动（不再无脑拉起，减少副作用） |

闸门的权威来源是 `dbus-send … org.bluez … GetManagedObjects`（Device1 的 `Connected` 属性）—— 本机确实有 dbus-send（蓝牙能出声就是靠它取到 MAC 的）；查不到才退到 `bluetoothctl`/`hcitool`。

`hasConnectedDevice` 的解析有个坑，也被单测抓出来了：判断 `"Connected"` 之后的值时，窗口必须**卡在下一个属性名之前**——「设备断开」的真实样本里 `Connected=false` 后面紧跟 `Paired=true`，窗口放宽就会误判成「有连接」，于是又走回蓝牙、扬声器继续哑。

探测要走 popen 起几个进程（上百毫秒），所以**缓存 10 秒**，不会拖慢「下一首 / 拖动」。纯解析逻辑拆在 `Media/AudioOutput.{hpp,cpp}`（不碰系统调用），可以在开发机上用真实命令输出样本直接单测：`.tools_tmp/audio-output-test.cpp`。

诊断页的**「蓝牙输出」按钮**：按一次换下一个候选 → 存进设置 → 现场放一声 660Hz 提示音。听到声音就停手，此时设置里已经是正确的那条。这是自动探测不灵时的兜底手段。「音频探测」里新增了 4 条蓝牙前提探针：`bluetoothd` 进程、`bluealsa` 程序与进程、`dbus-send` 与 `GetPCMs` 输出、ALSA PCM 列表。

#### 10.1 蓝牙断开要「停在本首」，不能跳下一首

**现象**：戴着蓝牙耳机听歌，耳机一断，播放器自动跳到下一首。

**根因在原生模块的 `reapLocked()`**：子进程只要退出，它就无条件把播放位置推到结尾（原注释写的是「避免进度条回跳」）。于是**任何一次异常退出都被当成「播完了」**，JS 侧的 `isEnded()` 返回 true，顺势切下一首。蓝牙一断 `aplay` 进程被杀 —— 正好命中。

**修法**是区分「真播完」和「半路退出」：

| 情况 | 判定 | 结果 |
|---|---|---|
| 正常播完 | `position >= duration - 0.5` | `baseOffset = duration`，`endReason = "eos"` |
| 半路退出 | 其它 | 停在当前位置、**置为暂停态**，`endReason = "aborted"` |

`getState()` 多返回一个 `endReason`；`platform.js` 透出 `endReason()` 与 `outputDevice()`，`player.js` 的轮询里多一步 `checkInterrupted()`：

- 中断前走的是蓝牙（`output` 含 bluealsa/a2dp/bluez）**且**设置里开着「蓝牙断开自动暂停」→ 停在本首，弹提示，`state.error` 写明「点播放可从原处继续」；
- 否则 → 沿用旧行为，当作播完切下一首。

三个必须处理的细节：

- `abortHandled` 标记：否则 250ms 轮询会反复触发同一次中断。在**新曲开播 / 重新播放 / 主动停止**时清零（`playIndex`、`toggle` 的恢复分支、`stop`）。
- `endReason` 在 `launchLocked()` 里清空（每次新起进程，上次的退出原因作废），`stop()` 也清 —— 用户主动停的不算「意外」。
- 判断「中断前是不是蓝牙」用的是**引擎回读的 `output`**，不是当前设置：断开的瞬间探测结果可能已经变回默认设备了。

设置项 `pauseOnBluetoothLost`（默认**开**）在设置页「偏好」分组里；改完会调 `player.refreshSettings()` 让正在跑的播放器立刻用上，不必重启应用。

#### 10.2 蓝牙断开后扬声器不响：中断时强制回避蓝牙一次

**现象**：暂停后按播放，扬声器一点声音都没有（界面显示在播放）。
**根因**：断开那一瞬间，BlueZ / BlueALSA 的信号**还没更新**，仍会报「已连接」，于是下一次启动又挑中 `bluealsa` —— 而这条路**「打开成功却一个字节都不送」**（进程一直活着，`launchVerified` 的「启动失败就回退默认设备」**永远不触发**），表现就是「在播放、但没声音」。
**修法**（原生 `AudioBridge`）：`reapLocked()` 判定为半路中断（`aborted`）时，除了停在本首，还**置一个一次性标记 `preferDefaultOnce_` 并作废设备缓存**。

```cpp
preferDefaultOnce_ = true;   // 下一次启动强制走系统默认设备
autoDevice_.clear();
device_.clear();
deviceProbedAtMs_ = 0;
```

`resolveDeviceLocked()` 开头消费这个标记：直接 `device_ = ""`（板载扬声器）并 return；用完即清 —— 下下次才重新探测蓝牙，那时信号已经稳定。

> 为什么不靠「启动失败再回退」：回退只在子进程**退出**时触发，而这条失败路径是**进程活着但不出声**，回退根本没机会执行。

#### 10.3 构建标记：确认装到机器上的 .so 到底是哪一版

`.so` 编译**不带 `-g`**（构建日志里只有 `not stripped`，没有 `with debug_info`），所以**成员名 / 函数名不会出现在二进制里** —— 用 `strings` 查「新代码有没有编进去」是**无效的**。我为此白查过一次（以为没编进去，其实早编进去了），而且文件大小也没变，是因为 **ELF 段按页对齐**，几十字节的增量不改变总大小。因此加了一个**会实实在在进二进制**的字符串常量：

```cpp
const char *BUILD_TAG = "2026-09-25 bt-abort-snapshot";
```

#### 10.4「蓝牙断开既没暂停、还跳下一首」：证据被自己清掉了

10.1 的判定（停在本首 / 切下一首）完全依赖一个问题：**中断发生的那一刻，音频走的是不是蓝牙？** 而这个答案**只能**来自引擎 —— JS 看不到蓝牙状态。

第一版是这样写的：`reapLocked()` 判为 `aborted` 后，一边置 `endReason_ = "aborted"`，一边把 `device_ / autoDevice_ / deviceProbedAtMs_` 全部清掉（那条通路已经不可信，下次不能再撞上去）。JS 那边则去读 `getState().output` 判断是不是蓝牙。**这两件事是互相拆台的**：等 JS 轮询上来时（几十毫秒后），`device_` 已经被清成空串，`output` 自然成了 `''` —— 空串 = 系统默认 = **不是蓝牙**，于是走进「按老规矩切下一首」，症状就是「蓝牙断开后既不暂停、还跳歌」，和「这段代码没生效」长得一模一样。

修法**先抄证据，再动现场**：在 `reapLocked()` 里进入 `eos / aborted` 分支**之前**，把中断瞬间的设备名和「是不是蓝牙」固定下来：

```cpp
endWasBluetooth_ = AudioOutput::isBluetoothDeviceName(device_);   // 之后才允许 device_.clear()
```

`player.js` 的优先级是：**引擎布尔值 → 中断快照设备 → 当前设备**（最后一条只是旧引擎的兜底）。

`endWasBluetooth` 还是个**免费的能力探测**：它的存在与否就等价于「机器上跑的是不是这一版 .so」。旧引擎的 `getState()` 里没有这个字段，回读得到 `null`，首页启动时据此弹一条「播放引擎是旧版（缺中断快照），请重装新包」—— **不用去和版本号比对**（版本号要在 C++ / JS 两边手工同步，迟早会漂）。

诊断页新增「上次结束原因」一行，把现场直接摊开：

```
半路中断 · 当时输出 bluealsa:DEV=<MAC>,PROFILE=a2dp · 走的是蓝牙（应暂停）
半路中断 · 当时输出 系统默认 · 不是蓝牙（会切歌） / 引擎无中断快照 —— 旧版 .so，请确认新包已安装
```

### 11. 首页重做（参考图：黑底 + 左悬浮导航 + 卡片列表）

配色是从参考图**逐像素采样**出来的，不是肉眼估的：

| 元素 | 取值 | 说明 |
|---|---|---|
| 页面底色 | `#0f0f11` | 规格给定；参考图实测 `#0A0A0C`，两者基本一致 |
| 卡片 / 悬浮播放条 | `#1a1b1f` | **正是 doge-calculator `icon-button.vue` 的默认色**，参考图就是用它做的 |
| 次级文字 | `#888890` | 歌手名 /「25首音频」/ 序号 |

**整页只有两种面**（参考图是「页面 + 一圈纯黑井底 + 卡片」三层，实际看是花了，已统一）：

| 层 | 颜色 | 用在 |
|---|---|---|
| 页面底色 | `#0f0f11`（`@bg`） | `.home` / `.well` / `.list` |
| 抬起的面 | `#1a1b1f`（`@surface`） | 卡片、悬浮播放条、空态占位圆盘 |

`.list`（scroller）仍然**显式**写底色 —— 不写会落到组件的默认底色上。`@well` 这个令牌已经删除，全项目零引用。

尺寸同样是量出来的（图高 ≈ 全屏高，所以像素比例直接就是 vh）：

- 导航按钮 **55/254 = 21.7vh**，与 doge-calculator 的 `22vh` 完全吻合 → 直接沿用 `IconButton.vue` 的 `width/height 22vh + border-radius 7vh + :active opacity 0.6`，只把图标占比从 60% 收到 45%（参考图里图标 22~24px 比按钮小得多）。
- 导航左边距 1.5vw、上边距 6.5vh、间距 7vh。
- 卡片高 **28vh**、封面 **17vh**（`CoverArt` 新增 `card` 尺寸）。

**导航条为什么能滑动**：它是定高 100vh 的 `scroller`，与右侧列表各自独立滚动，所以列表滚动时导航条不动（这就是「固定悬浮」）。前 3 个按钮占 86.5vh 正好占满一屏，第 4 个只露出一角 —— 这本身就是「还能往下滑」的视觉提示。

**Header 不是固定条**：`.head` 放在 `.list` 这个 `scroller` **里面**（渲染树是 `.home > .main > .well > scroller.list > [.head, .list-inner]`），所以它会跟着歌曲列表一起滚走，右侧元素是一个整体。它的左右内边距与 `.list-inner` 一致（1.6vw），标题与卡片左边缘对齐。

**卡片间距**：`.card-row` 带 `margin-bottom: @card-gap`（3.5vh）。加这个之前卡片是紧挨着的 —— 参考图里只有一张卡片，量不出间距，是实际看效果补上的。

**悬浮播放条**：高 `@float-h`（21vh，原来 16vh 太小）、封面 17vh（复用 `CoverArt` 的 `card` 尺寸）、标题 6vh、副标题 5.2vh、按钮 11vh、进度线 1.2vh。列表的 `padding-bottom` 必须跟着涨（28vh），否则最后一首会被条盖住。按钮**不再复用 `.pill-ghost`**：两个类定义同一批属性时谁最后注册谁生效（`.page` 已经踩过一次），所以 `.bar-btn` 自带整套样式，只用一个类名。

图标映射（`icon/` → `src/assets/`，按文件名合理对应）：

| 图标 | 动作 |
|---|---|
| `search_active` 放大镜 | 搜索（调起输入法/内置键盘） |
| `sort` 双向箭头（取自 `icon/新建文件夹`） | 排序方式 |
| `back` 箭头（**转 90°** → 向下折叠） | 折叠 / 展开底部悬浮播放条 |
| `refresh_active` | 重新扫描曲库 |
| `folder` 文件夹（取自 `icon/新建文件夹`，由用户指定） | 扫描目录 |
| `settings_active` 齿轮 | 设置 |

> 挑图标的方法记一下：`icon/新建文件夹` 里 200 个文件全是 MD5 名，肉眼逐个看不可行。做法是先按「正方形 + 尺寸 + 透明底 + 白色墨迹」筛出 139 个，**拼成带编号的索引图**一次看完（`.tools_tmp/icons-p1..p3.png`）；找不到文件夹时又按轮廓特征搜了两轮（实心 + 上窄下宽 / 顶部靠左短行），最后由用户直接指定。
> 顺带纠正一个早先的误判：`settings_active` 放大看是**齿轮**，不是点阵网格 —— 之前一直拿它当「分类网格」给排序用，现在正好归位给设置。

### 12. 两个构建坑（都会静默出错）

**坑 1：注释里不能出现 `require(` 加引号的写法。**

```js
// 千万别这么写注释：
/** 图标地址，用 require('../assets/xxx.png?base64') 传进来 */
```

构建器的依赖扫描**不看上下文**，会把注释里的示例当成真依赖去解析，轻则报一条莫名其妙的 `未找到以下模块: ...?base64` 警告，重则直接 `ENOENT: ... assets/xxx.png` 构建失败。描述时绕开这个写法即可。

**坑 2：不要靠覆盖 `base.less` 里的公共类来改页面骨架。**

每个组件都 `@import "base.less"`，`.page` 在包里被定义了十几次；本平台把样式**按类名全局注册**，重复定义谁最后注册谁生效。首页骨架要横向布局，本来写成 `.page { flex-direction: row }`（覆盖）也能跑，但那是赌注册顺序。现在首页根节点用自己的 `.home` 并写全所有属性，互不干扰。

> 顺带：`?base64` 图标内联是这个 CLI 自带的能力（`src/rollup-plugins/image.js`），本地图片走 `require('../../assets/x.png?base64')` 就会变成 `data:image/png;base64,...` 常量打进包里，`.amr` 里不需要额外的 images 目录。

### 13. 设置页与关于页

新增两个页面（`src/app.json` 里注册路由）：

| 页面 | 路由 | 内容 |
|---|---|---|
| 设置 | `settings` | 曲库分组（排序方式 / 扫描目录 / 重新扫描）、偏好分组（歌词翻译 / **设置自动保存**）、其它分组（诊断 / 关于） |
| 关于 | `about` | 大号应用图标 + 名称 + 简介 + 信息卡片 + 开源许可说明（参考 doge-calculator 的 `info.vue`） |

两页都沿用首页的骨架（同一条 `NavRail` + `scroller` 里的标题），所以看起来是同一个应用。

**设置自动保存**（`library.js` 的 `DEFAULT_SETTINGS.autoSave`，默认开）：

- **开**：设置页里的排序 / 目录 / 翻译改动**立刻写盘并生效**，只弹一条「已自动保存」。
- **关**：改动只落在页面内的 `draft` 上，底部出现「保存设置」按钮，点一下才统一应用；标题右侧显示「有改动未保存」。
- **自动保存这个开关自己永远立即落盘** —— 否则「关掉自动保存」这一步本身就不会被保存，下次进来又变回开启。重新打开时会把之前暂存的改动一起提交。

三个踩到的坑（都在设置页）：

1. **开关不能同时挂在行和开关本体上。** 行上已经有 `@click` 了，开关上再写一个就会触发两次、等于没切换。现在只在行上挂一次。
2. **模板里别写 `@click="commit"`。** 那样点击事件会当成第一个参数传进去（`commit(silent)` 的 `silent` 收到一个事件对象，真值），提示语会被吃掉。改成 `@click="onSaveClick()"`，或者 `@click="commit(false)"`。
3. **滑块不要用「绝对定位 + 绑定 left」**，改用 flex 的 `justifyContent`。`left` 是绑定样式，万一运行时没吃进去，滑块就永远停在左边 —— 看起来就是「开关坏了」。现在 `.switch` 是纯 flex 容器（自带 padding），滑块是普通子元素，靠 `justifyContent: flex-start/flex-end` 挪；每行还额外显示「开 / 关」两个字，状态一眼可见，不指望用户分辨滑块位置。

#### 13.1 设置页重做：样式与首页同款 + 侧边返回键

改版目标就一句话：**设置页要和首页看起来是同一个应用**。做法不是「照着首页再抄一遍尺寸」（抄了就会漂，而且这一页原先抄的那份还和关于页互相打架），而是**共用同一份定义**。

| 项 | 来源 | 设置页的取法 |
|---|---|---|
| 整页骨架 / 页头 / 标题字号 | `styles/layout.less`（与首页、关于页同一份） | 页头 20vh、标题 `@fs-h1`、左右内边距 1.6vw |
| 卡片 | 首页 `.card-row` 的语言 | 圆角 **5vh**（与歌曲卡片、悬浮播放条同值）、底色 `@surface` |
| 分组标题 | 首页「5首音频」那一行 | `@fs-count` + `@text-mute` |
| 组间距 | 首页卡片间距 `@card-gap` | 3.4vh |
| 按钮 | `base.less` 的 `.pill-primary` | 强调色胶囊 + 按下变暗 |

具体改动：

- **行高 14vh → 16vh，标题 `@fs-md` → `@fs-lg`**：原先比首页小一大截，看着不像同一个应用；现在与首页的排版节奏对得上；
- **四行灰字说明收进一张卡片**（`.set-note`）：原先 `autoSaveHint` / `pauseOnLostHint` / `excludeAmrHint` / `保存位置` 四行直接摊在底色上，像一段没人管的注释；现在是一张与其它卡片同圆角同底色的说明卡，「设置保存在：仅内存」仍会在写不进去时变黄（内联样式）；
- **值一律右对齐**：原先带图标的行漏了 `.filler`，值紧贴在标签后面，而开关行是右对齐 —— 同一张卡片里两种对齐方式。现在每行都有 `.filler`；
- **「保存设置」改用 `.pill-primary`**：强调色、按下态与全局主按钮一致，自己只补外边距（改的是**不同属性**，不存在争用）；
- **行按下反馈保留 opacity**（没改底色）：行是矩形、卡片是圆角，给行铺底色会在卡片上下圆角处露出直角。这条是权衡后的选择，不是漏改。

**侧边返回按钮**：导航条第一个按钮就是「返回」，图标 `back.png` **不旋转** —— 它本身就是左向箭头（`<`）。原先照抄了首页折叠箭头那一次 `rotate: 90`，结果侧边显示的是一个**向下**的箭头，完全不像返回（这次要修的就是它）；同时把第一个按钮由「拿放大镜当首页」换成了返回，并把导航顺序对齐首页的语义：返回 / 排序 / 重新扫描 / 目录 / 设置（当前页高亮）。

### 14. 设置改完「不生效」：首页必须在 onShow 里重读设置

**现象**：在设置页把「设置自动保存」打开、改了排序，返回首页 —— 列表顺序没变，看起来就是「自动保存不可用」。

**根因**：首页的 `sortMode` / `currentDir` 只在 `mounted → init()` 里读一次；从设置页返回时首页**并没有被销毁重建**，走的只是 `onShow`，而当时的 `onShow` 只刷新了播放状态、从不重读设置。所以改动其实**已经存盘了**，只是首页没看。

**修法**：`index.vue` 的 `onShow()` 调 `syncSettings()` —— 重读设置，排序变了就应用，**目录变了还要重扫**（否则列表还是旧目录的内容）。

> 依赖前提：框架确实会调 `onShow`（`base-page.js` 里 `this.$root.onShow()`），已核实。

**教训**：凡是「在 A 页面改、在 B 页面显示」的设置，B 页面都得有重新读取的时机 —— 存盘成功 ≠ 界面更新。以后再加这类设置，先想清楚 B 页面什么时候重读。

### 15. 设置重启后丢失：系统存储写不进去，兜底到文件

**现象**：设置当场生效，**关掉应用重新打开就全没了**（比「界面没刷新」更底层）。

**根因在 `store.js`**：它逐个试探 `$falcon.jsapi.storage` 的多种调用形态（`setItem({key,value})` / `setStorage({key,data})` / 位置参数 …），**全都写不进去时就静默退回进程内内存** —— 本次会话正常，重启即丢。更糟的是旧代码**从不检查写入结果**，`saveSettings` 拿到 `false` 也当成功，所以这个洞一直没暴露。

**现在两道保险**（都在 `store.js`）：

| 保险 | 做法 |
|---|---|
| 写后回读校验 | 写完立刻读回来比对。有些接口会「收下但丢掉」，写不报错 ≠ 真存住了 |
| 文件兜底 | 系统存储靠不住时，用自研 FileShell 写 `$falcon.$dataDir/store/<key>.json` |

读的时候先问系统存储，读不到再读兜底文件 —— **重启后能读回设置就靠这一步**。

几个细节：

- 兜底目录的取法与封面缓存一致（`$falcon.$dataDir`，拿不到就 `/userdisk/.mlp-cache`）；原生 `FileShell::writeFile` **会自动建父目录**，写入失败会抛错。
- 文件写完**再回读一次**确认，否则诊断页会谎报「已保存」。
- `store.js` 新增 `probeStore()` 与 `storeBackend()`：诊断页多一行「设置保存位置」，直接写明是**系统存储 / 兜底文件 / 仅内存**；设置页底部也有一行「设置保存在：…」，改成「仅内存」时会变黄示警。
- JS 侧 `writeText` 在这之前从未被使用过（只有定义），所以这条路径原本没有实战验证；现在有了回读校验兜着，真写不进去会如实显示成「仅内存」，不会假装成功。

**诊断页没有搬进设置页**，而是作为设置里的一个入口（`$falcon.navTo('page')`）。原因：那个页面有十几个探针和发声测试，塞进来会让设置页失控；而且工程约定要求 `pages/page/page.vue` 必须保留。入口从主导航移到了设置里，「诊断」仍然一点即达。

#### 15.1 单次加载数目（默认 15）

曲库大时一次渲染几百张卡片会卡（每张卡片含一次封面解码），所以首页**先只渲染 15 首**：列表底部给「加载更多」（再渲染 15 首），右上角计数同时写成 `15/25 首音频`，让人知道下面还有。设置里「曲库 → 单次加载数目」可选 10 / 15 / 20 / 30 / 50 / 全部。

**最容易写错、写错了用户会直接骂的一条：播放列表必须是全部歌曲。** 所以首页有两个列表：

| 计算属性 | 内容 | 用在哪 |
|---|---|---|
| `visibleTracks` | 排序 + 搜索过滤后的**全部**歌曲 | **播放列表**（`onSelectTrack` 里 `player.setPlaylist`） |
| `shownTracks` | `visibleTracks` 的前 `loadLimit` 首 | 只用于 `v-for` 渲染 |

点第 3 首播的是整库里的第 3 首，上一首下一首会走到第 16 首以后（没渲染的那些）；把 `shownTracks` 传给播放器就会把播放列表截断成 15 首 —— 这条不变量由 `node .tools_tmp/test-index-page.js` 守着（用例 4/5 专门断言「只渲染 15 首时点第 3 首，播放列表长度仍是 25、且能取到第 24 首」）。

其它细节：

- `pageSize = 0` 表示「全部」：不截断，也不显示「加载更多」。
- 只有设置里的值**真的变了**才重置已加载数量；否则每次从播放页返回首页，之前点过的「加载更多」就被抹掉了（`syncSettings` 里做了比较）；
- 搜索态沿用原来的 `N / M 首音频` 写法，不受影响。

#### 15.2 排除 .amr 文件（默认开）

`.amr` 在这台设备上**既是音频格式（Adaptive Multi-Rate）、又是小程序安装包格式** —— 本应用自己的安装包就是 `8001749598192572.1_1_3.amr`。所以曲库很容易把安装包当成歌扫进来。设置里新增「排除 .amr 文件」，**默认开**（即默认不把 .amr 当音频）。

实现上有一个细节值得记：`scanLibrary({ dirs, excludeAmr })` **允许按次覆盖**设置里的值。原因是设置页改完**立刻重扫**才能看到效果，而「设置自动保存」关掉时改动还没落盘 —— 如果重扫只读模块里的设置，就会用旧值扫出旧结果，看起来像开关没生效。所以设置页把草稿值显式传进去。

```js
scanLibrary({ dirs: [dir], excludeAmr: this.draft.excludeAmr })
```

### 16. 构建不报、只在运行时炸的一类错误

**踩了两次，都是「调用了但没定义」：**

1. 整文件重写 `store.js` 时漏掉了 `tryWrite` / `tryRead` / `normalizeRead`，装到机器上一扫曲库就 `ReferenceError: tryWrite is not defined`。
2. `player.js` 的 `setOutputDevice()` 里写成了 `emit()`，而那个文件里只有 `notify()`（`emitter.emit(...)` 才是方法）→ 诊断页「蓝牙输出」按钮一点就炸。

**为什么构建不报**：rollup 不做这层检查，`ReferenceError` 是运行时才发生的。**现在有 `.tools_tmp/check-refs.js`**：扫描 `ui/src` 下所有 `.js` / `.vue` 的 script，列出「被调用但没有定义」的标识符（排除关键字与平台全局）。

```bash
node .tools_tmp/check-refs.js          # 扫 ui/src
node .tools_tmp/check-refs.js <目录>   # 扫指定目录（用于自测）
```

**这个检查器自己也被坑过一次，值得记下来**：第一版把「对象键」的正则写成 `[({,\s](\w+)\s*[:(]` —— 里面那个 `(` 让**任何函数调用都被当成了定义**，于是它对一个故意写坏的文件报「检查通过」，等于给假信心。改成「方法简写必须右括号紧跟 `{`」（`foo(a, b) {`）之后才准。**所以它对现在的源码报「通过」是有意义的** —— 这是用一个故意写坏的文件验证过的（那个文件必须被检出、退出码非 0）。

> 教训：检查器写完后**用一个必然失败的反例验证**，否则「通过」两个字一文不值。

### 17. 播放页重做（封面 / 大字歌词 / 悬浮控制条）

参考图是一张桌面播放器的截图，构图是「左边大封面（下面带倒影）、右边大字歌词、底部一条胶囊控制条，整页铺一层封面色调的背景」。搬到 640×260 的横条屏上，能照搬的都照搬了，照搬不了的说清楚为什么。**倒影后来按用户要求取消了**（见下），省下的竖向空间全部给了封面本身。

**构图与占比**（vh = 屏高的 1%，本机 640×260，所以 1vh≈2.6px）：

| 区域 | 取值 | 说明 |
|---|---|---|
| 封面 | `64vh` 见方（CoverArt 的 `hero` 尺寸） | 原先 56vh + 13vh 倒影，倒影取消后整块让给封面 |
| 封面列 | `32vw` | 必须与 LyricPanel 的 `LEFT_COL_VW` 一致；也要装得下封面（见下） |
| 标题行 | `13vh`（曲名 7vh 粗体 + 歌手 3.7vh） | 参考图的标题就在歌词正上方 |
| 歌词窗口 | `59vh`，当前行字号 8vh、邻行 6.6vh | 必须与 LyricPanel 的 `VIEWPORT_VH` 一致 |
| 进度行 | `8vh`（±15 按钮 8vh、轨道 4.5vh） | 自绘进度条，支持点击跳转 |
| 控制条 | `18vh`，圆角 `50vh`，按钮 12vh / 播放键 15vh | 参考图那条胶囊条 |

（竖向预算：底部固定占用 = 进度行 8 + 控制条 18 + 下边距 1.8 = 27.8vh，主体 `flex: 1` 得 72.2vh；右列 13+59=72vh、左列封面 64vh —— **两边都要塞得下**，这个预算由 `tools/check-layout.js` 自动核对。）

> **这一节的数值有两套来源，必须成对修改**，所以专门写了 `node tools/check-layout.js` 来核对（构建前跑，对不上就退出码 1）：样式在 `player.vue`，而 LyricPanel 的折行估算常量、ProgressBar 的点击定位常量在各自的 `<script>` 里。这类「改一处忘了另一处」不会报错、不会崩，只会「看着有点不对」，历史上已经漏过一次（见下面「点击跳转」）。

**环境底色：把封面铺满整屏当背景**（`.pb-amb` + `.pb-shade`）。参考图的紫调背景其实就是封面放大压暗，这里用 `background-image` + `background-size: cover` + `opacity: 0.18`，上面再盖一层 `rgba(9,9,11,0.55)` 压暗保证白字可读。两点考虑：

- `falcon-styler` 对 `background-image` **不做值校验**（原样透传）；
- 但**渲染器吃不吃 `url(本地文件)` 没法在这里验证**，所以这一层被做成纯装饰：拿不到封面就绑空对象，那一层全透明，底下还有页面底色 → 最坏情况就是回到原来那套深色界面，不会「坏成一片白」。

**封面放大到 `64vh`、封面列放宽到 `32vw`**（原为 56vh / 26vw）：

```less
.pb-left {
  box-sizing: border-box;
  width: 32vw;              /* 必须 ≥ 封面换算成 vw，否则封面被裁 */
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}
```

- 封面是**正方形**，边长用 vh 给；列宽用 vw 给 —— 横屏机上这两个单位差着「屏宽 ÷ 屏高」倍（本机 640×260，1vw = 2.46vh），所以**列要装得下封面**，否则两侧会被 `.pb-page` 的 `overflow: hidden` 裁掉，正方形变成竖条。64vh 换算成 vw 是 `64 × 260/640 ≈ 26vw`，取 32vw，左右各留 3vw 呼吸空间。这一条由 `tools/check-layout.js` 自动核对（它从 `ui/src/utils/screen.js` 读 640×260）。
- 竖向：封面 64vh 放得进主体 72.2vh，上下各留约 4vh。
- 历史：原来是「56vh 封面 + 下方 13vh 倒影（`scaleY(-1)` 翻转 + 四边偏移裁切）」，**倒影按用户要求整体删除**（模板节点、`.pb-refl` / `.pb-refl-img` 样式、以及配套的 `reflStyle()` 计算属性一起删），省下的 13vh + 1vh 间距全部让给封面。
- `CoverArt` 那边只改了 `hero` 尺寸；`.cover-img` 仍用「四边偏移」这个设备上验证过的写法。绝对定位元素要在 `.cover` 里定位，靠的是 `.cover` 的 `position: relative` —— 少了它就会「按 A 裁、在 B 定位」，画面直接错位。

**进度条点击跳转（点哪儿跳哪儿）**：按下即定位到手指所在位置，继续拖动就跟手，抬起提交 —— 点击和拖动其实是同一件事：把手指的**屏幕绝对 x** 换算成百分比。

换算需要知道轨道左边缘与实际宽度，而**本渲染器没有可用的测量 API**（`getBoundingClientRect` / `offsetLeft` / `offsetParent` 都不保证存在）。这里踩过一次坑，也正是「点了没反应」的原因：

- 旧写法是「先试着量，量不到就用 `estimatedVw`（34vw）估」。元素 API 整套都不在 → 量出来是 0 → 轨道左边缘被当成 0 → 点屏幕中间直接跳到 100%；
- 而 34vw 这个兜底值是照**旧版**播放页（`.seek { width: 40vw }`）配的，新版轨道约占 90vw，兜底值本身也早就对不上了。

现在改成**从布局常量直接推**：

```js
// ProgressBar.vue
const SEEK_PAD_VH = 3     // player.vue  .pb-seek  padding 0 3vh
const BTN_W_VH = 8        // 本文件      .pbar-btn 宽 8vh
const BTN_GAP_VH = 1.6    // 本文件      .pbar-track 左右外边距
inset = vh(3 + 8 + 1.6) = 32.76px      // 640×260 下换算出来的轨道左边缘
width = 640 − 2×32.76 = 574.48px

percent = (手指 x − inset) / width × 100
```

`vh → px` 用 `LAYOUT.height`（和 LyricPanel 同一套换算），所以结果精确且可控，不依赖任何运行时测量。**改这三处样式里的任何一处都要同步改这三个常量**（`tools/check-layout.js` 会替你对）。

**歌词面板**三处改动：

1. **不再画卡片底**。`.panel` 去掉 `@surface` 底 + 边框 + 圆角 —— 参考图里歌词直接浮在环境底色上。可读性靠压暗层保证。
2. **当前行纯白 + 加粗 + 大一号**（邻行 6.6vh、当前行 8vh，行高 10.4vh）：原先当前行只是换个强调色，而参考图是靠「更大更亮」突出当前行。
3. **逐行字号必须同时喂给行高估算器**：`rows()` 里当前行要用 `FS_ACTIVE_VH / LH_ACTIVE_VH` 算折行数与行高，否则长句会低估一行，滚动定位就偏。

**改了播放页布局，必须同步改 LyricPanel 顶部那组常量**（最容易漏的一处）：

| player.vue | LyricPanel 常量 |
|---|---|
| `.pb-left` 宽 `32vw` | `LEFT_COL_VW = 32` |
| `.pb-body` 左右内边距 `3vh` | `MAIN_PAD_VH = 3` |
| `.pb-right` 左内边距 `2.4vh` | `RIGHT_PAD_VH = 2.4` |
| `.viewport` 高 `59vh` | `VIEWPORT_VH = 59` |

漏了左边 32vw，可用宽度会按整屏算（约 1.5 倍），长句会被误判成一行，真机折成两三行后就会互相压字。**这四条由 `node tools/check-layout.js` 自动核对**（连同下面 ProgressBar 的三条和封面尺寸与封面列的配合），不用再靠人记。

**控制条**：`.pb-bar` 自带整套属性（底色 / 边框 / 圆角 / 高度 / 内边距），**不复用 `.pill`** —— 要改高度和底色，复用就等于「两个类都写同一个属性」，谁生效取决于注册顺序（`.bar-btn` 当初就是同一个理由）。条内两套按钮，各自自带整套盒子属性（同样是为了不赌顺序）：

| 类 | 用途 | 高度 |
|---|---|---|
| `.pb-btn` + `.pb-btn-text` / `.pb-btn-on` | 文字按钮（循环 / 倍速），@fs-md | 12vh |
| `.pb-ibtn` + `.pb-ico` | 图标按钮（上一首 / 下一首 / 翻译），图标 7vh | 12vh |
| `.pb-ibtn-main` + `.pb-ico-big` | 播放/暂停，图标 9vh | 15vh |

`base.less` 的 `.pill-text` 是 `@fs-sm`，在放大后的条里偏小，所以文字按钮用自己的类（`.pb-btn` / `.pb-btn-text` / `.pb-btn-on`）；图标尺寸类（`.pb-ico` / `.pb-ico-big` / `.pb-ico-close`）**一个元素只用其中一个**，不存在属性争用。`check-layout.js` 会核对「条内按钮不超过条高」。

条子左边那块放「时间 + 第几首 + 常驻状态」：既填住了参考图里放曲名/时间的位置，又保住了原来那条常驻状态行（引擎 / 时长 / 倍速请求与实际的差异 / 变速方式 / 错误，出错时变黄）—— 那条是上一轮特意做成常驻的，不能在改版里悄悄丢掉。

**时间为什么在条上、不在进度条右边**：进度条的点击定位是按「进度行内边距 + ±15 按钮宽 + 轨道外边距」算出轨道位置的，同行里多一个元素（时间文本宽度还随 `00:14 / 03:06` 还是 `1:23:45 / 2:00:00` 变化）就会把轨道挤窄，算出来的百分比整体偏移 —— 时间挪进控制条既解决了这个问题，也和参考图一致。`tools/check-layout.js` 会检查「进度行里只有进度条」这一条。

**刻意没有照搬参考图的地方**：参考图底部的音量 / 输出格式（48 kHz）等信息本项目没有对应能力，不画空壳。

#### 17.1「歌词拖不动 / 不跟手」的真正原因：拖动根本没启动

**现象**：手指按住歌词上下滑，歌词纹丝不动（或者偶尔动一下就不跟手）。

**根因**：`onTouchStart` 的第一行是

```js
const y = touchY(event)
if (!isFinite(y)) return      // ← 致命
```

**有些固件只在 touchmove 里带坐标，touchstart 里没有**。于是：

- 按下时 `touchY` 拿不到坐标就 `return` → `dragging` 永远是 `false`；
- 之后每一次 `touchmove` 都被首行 `if (!this.dragging) return` 挡掉；
- 结果就是**整个拖动功能从未启动**，跟不跟手根本无从谈起。

修法是**把「冻结」和「取起点坐标」拆开**：冻结（记录当前跟随位移与窗口起点）不需要坐标，按下时先做；起点坐标推迟到**第一个带坐标的事件**里再取，并且把那一刻当作「手指刚按下」（位移从 0 开始算），所以画面不会先跳一下。

```js
onTouchStart() {
  /* 先冻结 —— 不需要坐标 */
  this.frozenOffset = this.autoOffset
  this.frozenStart = this.liveWindowStart
  this.dragging = true
  this.startPending = !isFinite(y)      // 起点待定
}
onTouchMove(event) {
  if (this.startPending) { this.dragStartY = y; this.startPending = false; return }
  this.dragOffset = this.clampDrag(y - this.dragStartY, this.frozenOffset)
}
```

**同一个 bug 也在进度条上**（`ProgressBar.vue` 的 `onStart` 同样是「取不到坐标就 return」），一并改成同样的写法；顺带修掉另一个相关的坑：如果整次触摸**一次坐标都没拿到**，以前还会无条件 `emit('seek', localPercent)`，把进度「跳到上一次的位置」—— 现在什么都不提交（±15 秒按钮始终可用）。

**坐标提取合并成一个文件**：`ui/src/utils/touch.js`。原先两个组件各抄了一份「从多个字段兜底取坐标」的代码（注释还写着「写法与 XX 完全一致」），一处改进另一处必漏。现在只有一份，且字段覆盖更全：`changedTouches / touches / targetTouches` 里的 `pageX|clientX|screenX|x` 依次兜底，再退到 `event.data.*`、`event.detail.*`、事件对象本身。（坐标是绝对还是相对都不影响拖动 —— 拖动只用两点之差。）

**另外给动画加了一道保险**：拖动时把 `transitionDuration` 从 `0` 改成 `1`。`0` 在某些渲染器上会被当成「没设置」而回落到静态样式里的 `300ms`，那样拖动就是「慢半拍再追上」—— 也是典型的「不跟手」。1ms 与 0 观感无区别，但绕开了歧义。

#### 17.2「划走看歌词后，点一下屏幕就跳回当前行」

**期望的行为**（照用户给的两条）：

```
划动 → 不再划            → 倒数 3 秒后回到当前行
划动 → 3 秒内又划 → 停手 → 倒数从最后一次划动重算，到点回当前行
```

**实际**：划走之后只要手指再碰一下屏幕（哪怕没移动），画面立刻弹回当前行。

**两个原因叠在一起**：

1. **累计位移被丢掉了**（真正的元凶）。冻结基准 `frozenOffset` 记的是「开始拖动**之前**」的位置，而划走的位移一直存在 `dragOffset` 里。下次按下时 `dragOffset` 被清零 → 位移凭空消失 → 画面立刻弹回冻结基准，也就是原来那一行。修法：**松手时把 `dragOffset` 并入 `frozenOffset` 再清零**。并入前后画面位置完全不变（`新 frozenOffset + 0 === 旧值 + dragOffset`），视觉上是「无操作」，只是把状态整理成「基准 = 当前位置、增量 = 0」，下次按下才不跳。
2. **按下时重新锚定**。`onTouchStart` 原先无条件重取「活」的 `autoOffset` 与窗口起点，于是手指刚碰上去（还没抬手）轨道就已被锚到当前行 —— 歌已经往前唱了两句时尤其明显。修法：**只有「正在自动跟随」时才重新冻结**；停留在看某一段时沿用原来的冻结值。

另外把「松手就回位」的规则改成**任何一次抬手都重新计时**：

| 操作 | 旧行为 | 新行为 |
|---|---|---|
| 划动后停手 | 3 秒后回位 | 3 秒后回位（不变） |
| 3 秒内再划 | 定时器被清掉却**不重设** → 永远不回位 | 从最后一次划动重新算 3 秒 |
| 划动后点一下 | **立刻回位** | 保持原位，重新计时（点一下也算「还在看」） |
| 自动跟随时点一下 | 立刻回位 | 保持跟随，**不**白白冻结 3 秒 |
| 按住不放 | 倒计时已取消 → 不会跳走 | 一样（按住期间绝不跳） |

**这块逻辑现在有单元测试**：`node .tools_tmp/test-lyric-drag.js`。做法是把组件真实的 `<script>` 抠出来，用桩替掉平台依赖（`LAYOUT` / `touchY` / 定时器），直接调用**组件自己的方法**跑状态，于是「划动 → 停留 → 回位」的时序能在本地断言，不用靠真机手感去摸。13 条断言覆盖上表全部情形。

> 写测试的收益当场就体现了：第一版把「并入位移」这一步漏了，光看代码觉得没问题，测试直接跑出 `-160 -> -100` 的跳变。**另外还有一条断言是测试自己写错了**（手指上移 30px 时把期望写成了 `+30`）—— 用例 1 已经验证过方向，所以那次是改测试而不是改代码。

#### 17.4 播放中拖动「抢手」：换句把手指底下的字顶走了

（17.3 不存在于原文，编号沿用原稿。）

上一节修好「拖不动」和「点一下弹回去」之后，用户仍反馈**播放时**拖动抢手。关键线索就是「播放时」三个字 —— 不播放时拖得好好的。

**根因**：当前行比邻行**大一号**（8vh vs 6.6vh，行高差约 4.7px），所以「当前行是谁」直接决定每一行盒子的高度：

```
行高 = 折行数 × 行高 + 留白          ← 当前行用 activeLh，邻行用 mainLh
```

播放时当前行每隔几秒往前走一句 → **整条轨道的高度与各行位置都跟着变**。手指按着不动，位移（transform）确实没变，但手指底下的**排版**变了 —— 字自己上下抖一下，感觉就是「抢手」。

修法：再加一个「**排版用的当前行**」`layoutActiveIndex`，在按下（或停留）期间冻结：

```js
layoutActiveIndex() {
  if ((this.dragging || this.paused) && this.frozenActive >= 0) return this.frozenActive
  return this.activeIndex            // 自动跟随时照常跟着歌走
}
```

`rows()` 用它算 `active` / `distance` / 行高，于是按住期间换句对排版零影响，高亮也一起冻结 —— 它跟行高本来就是同一件事的两个面，只冻一个照样抖。松手回位（`resumeFollow`）时解冻，歌词重新跟上歌曲。

**测试**（`test-lyric-drag.js` 用例 6，共 36 条断言）：按住不放的同时把 `index` 连续推 5 次（模拟唱了 5 句），断言**位移与整条轨道的行高序列都一字不变**；松手回位后再断言排版重新跟上最新的当前行。

> 这条用例按老规矩验过反例：把冻结那一行改回 `-1`（等于旧行为），5 条「行高排版仍然没变」的断言立刻报错 —— 而「画面仍然没动」那几条**照样通过**。这恰好印证了根因：transform 没动，动的是排版。

#### 17.5 控制条的图标从哪来（以及为什么有的图标要重新着色）

需求是「播放/暂停等改成图标，翻译按钮搬到控制条里，原返回按钮改成 ✕」。图标都在仓库根目录的 `icon/` 里，但**那是一堆哈希文件名**（`icon/新建文件夹/*.png` 300 多个，看不出哪个是哪个）。找图的过程本身值得记下来，因为下次还要找：

1. **先找「一家人」**。用户给的「关闭」图标是 `36x36 / 317 字节`，而 `icon/暂停.png` 是 `200x200 / 2164 字节` —— 按尺寸+体积分组后，`200x200` 这一族有 33 个，正是**播放器控制图标集**：`0=播放▶ 1=暂停⏸ 10=上一首|◀`（实测 `暂停.png` 与第 1 个文件 MD5 完全相同，确认是同一套）。找图的方法：用 GDI+ 把候选图标拼成**带编号的对照图**（`System.Drawing`，深色/浅色底各画一遍），看一张图就能认出全部，比一个个打开快得多。（`.tools_tmp/_sheet200.png` 就是这么来的；同一招也用来核对新图标的实际颜色。）
2. **图标集里没有「下一首」**（只有 `|◀`），所以「下一首」用同一个图标**水平镜像**：`.pb-ico-mirror { transform: scaleX(-1) }`。
3. **白图标只能放在深色面上**。play/pause/prev 原图是纯白，而播放键是强调色 `#4fe0c4` —— 白字放上去对比度约 **1.35:1，等于看不见**。所以播放/暂停另存了**墨色版**（`icon-play-ink.png` / `icon-pause-ink.png`，`#08080a`）。
4. **用户给的翻译图标原本是黑的**，在深色控制条上同样看不见 → 也改成白色。
5. 重新着色用 GDI+ 的 **ColorMatrix**：颜色贡献清零、第 5 列填目标色、alpha 行保留，即「保留透明度、把所有 RGB 换成目标色」。
6. **顺手把大图压到 128px**：翻译图标原图是 `512x512`，解码一张 RGBA 要 **1MB 内存**，而这台设备是嵌入式 Linux。统一压到最长边 128px（现有的 `refresh_active` 就是 96px，同一档），显示尺寸才 7~9vh（约 18~23px），完全够用。包体也因此从 +19KB 降到 +11KB。

最终 `ui/src/assets/` 里新增 5 个：`icon-close`（用户给的 ✕）、`icon-prev`（白）、`icon-play-ink` / `icon-pause-ink`（墨色）、`icon-translate`（改成白色的翻译图标）。

**按钮分工**：`上一首 / 播放暂停 / 下一首 / 翻译` 用图标；`循环 / 倍速`**保留文字** —— 它们的状态（循环·顺序·单曲·随机、1x/1.5x/2x）一个图标讲不清楚，文字更好认。翻译按钮开着时整颗点亮（内联样式换背景 + 边框色，跟 `.pb-ibtn` 的静态样式不冲突）、播放/暂停按钮显示的是**将要发生的动作**：正在播放时显示暂停图标。原文里那句「参考图是一排图标，本项目继续用文字按钮」到此作废 —— 现在除了循环与倍速，控制条已经是图标了。

### 18. 视频播放探索（实验室 → 视频模式）

用户问「能不能做视频播放」。先说结论，再说依据 —— **扫描与列表已经做完并且能确定工作；画面能不能放出来，必须靠真机探测，我没有设备。**

#### 18.1 先查清楚平台到底给了什么

| 问题 | 结论 | 依据 |
|---|---|---|
| 构建工具链认不认 `<video>` 标签？ | **认** | `aiot-vue-cli/web-libs/falcon-vue-precompiler/src/config.js` 的 `weexRegisteredComponents` 里就有 `video`，属 Weex 标准组件；实测编译产物里有 `_c('video', { staticClass: ["vvideo"] }, …)` |
| JS 侧有没有视频接口？ | **没有** | `$falcon.jsapi` 只有 `http / modal / storage`，系统模块也只有 `global` / `http`（本项目探查过） |
| falcon-ui 里有没有现成的播放器组件？ | **查不到** | `ui/node_modules/falcon-ui` **根本没装**（构建时那句 `找不到theme "theme-default"` 就是它），项目里所有 UI 组件都是自己写的 |
| 有没有元素测量 API（`getComponentRect` / 渲染 API）？ | **没有** | 全仓库与依赖里都没有 `getComponentRect` 之类的用法（这也是进度条点击定位改成按布局算的原因） |

所以「能不能放视频」取决于三件事，**都只能在机器上查**：

1. 固件里有没有能解码视频的程序或库（`ffplay` / `mpv` / `gst-launch` + 相应的 decoder）；
2. 有没有能把画面显示出来的「面」（`/dev/fb0`、`/dev/dri`、wayland/X，或某个厂商的显示服务）；
3. 小程序的**渲染器**有没有实现 `<video>` 这个组件（工具链认它 ≠ 运行时实现它）。

#### 18.2 这次交付了什么

1. **设置 →「实验室」分组**（默认关）。关着的时候，下面的实验项在界面上**根本不出现** —— 万一某个实验特性在真机上会崩，默认状态必须是安全的。
2. 打开实验室后出现**「视频模式」**（默认关）。开启后**首页只列视频文件**（标题变「本地视频」、计数论「个视频」、扫描用 `kind: 'video'`）。
3. **视频页 `pages/video/video.vue`**（新路由）：进去就把文件路径交给平台 `<video>` 组件试放。页面上的状态行会写明「还没收到播放事件」或组件报的错 —— 一眼能看出这台机器的渲染器到底实现没有。同时给一条**保底路线：「只播声音」**，它复用现成的音频引擎（ffmpeg 解视频里的音轨 → aplay），这条链路在真机上已经跑通过。
4. **诊断页新增「视频能力探测」**（`page.vue` 里 8 条命令），一键把上面三件事全查出来：播放器程序、ffmpeg 解码器、GStreamer 视频元件、帧缓冲、GPU/DRM、显示服务、渲染器里有没有 video 组件、机器上到底有没有视频文件。

#### 18.3 已经确定的部分（有测试守着）

- `VIDEO_EXTS` / `isVideoFile()` 的扩展名判断（含大写、含「.amr 不算视频」）；
- 「实验室 + 视频模式**都开**才算生效」的严格判断（`isVideoModeOn`，用 `=== true`，存储里的脏值兜得住）；
- 首页在视频模式下：扫描带 `kind: 'video'` 的条目、**不碰音频播放列表**、点条目跳 `video` 页并带上路径、关掉视频模式后一切回到音频路径。这 5 条在 `test-index-page.js` 的用例 8~10 里（约 40 条断言）。

#### 18.4 还需要真机确认的部分

- `<video>` 组件在这台渲染器上到底渲染不渲染（视频页会自述，诊断页也能查库里的组件名）；
- 有没有可用的视频解码器与显示面 —— 如果没有，画面这条路就得放弃，要么只保留「视频当音频放」，要么走「用 exec 拉起外部播放器」，后者要看固件里有没有那个程序、以及它能不能直接写屏。

**换句话说：这一步是「把探测手段做扎实」，不是「假装视频已经能放」。** 拿到诊断页的探测结果后，才能决定下一步走渲染器组件、外部播放器，还是只做音频轨。

#### 18.5 真机反馈的两个 bug（都是「平台约定猜错了」）

第一版发到机器上，用户报了两件事，根因都属于同一类：**按想当然写了平台 API 的形状/名字，而真机不是那样。**

**① 视频能力探测不显示任何信息**：界面只剩标题、一条正文都没有。原因：`FileShell.exec()` 返回的是**字符串**，而代码写成了 `out = (result && (result.output || result.stdout)) || ''`（← 永远空串）。诊断页其实一直写对了（`typeof result === 'string' ? result.trim() : …`），视频页是自己另写了一遍、抄漏了。

修法：抽出 **`services/probe.js`**，`exec` 的返回形状**只有这一处实现**，而且**两种形状都认**（字符串 / `{output|stdout|data|text}` 对象），诊断页那两处探测也一并改过来 —— 这类「两处各写一遍、抄漏一处」正是 bug 的来源。

> 顺带被测试逼出一个行为差异：「命令无输出」在**环境探测**里该标黄提醒，在**发声测试**里反而是好消息（命令没报错）。第一版统一成不告警，与诊断页原来的 `warn: value === '（无输出）'` 不一致 —— 现在做成 `warnOnEmpty` 选项。

**② 主页能显示视频，点进去却「没有视频路径」**：原因：只读了 `$page.params`。事后在 `base-page.js` 里查到这台设备的真正机制是页面生命周期的 `onLoad(options)` / `onNewOptions(options)`（参数存在 `this.options`）—— 而参考应用 doge-calculator 里 `$falcon.navTo()` **从不传参数**，所以项目里根本没有可抄的先例，纯属猜的。

修法分两层：

1. **主路径换成模块状态**：首页点条目时 `setSelectedVideo(track)` 存进 `library.js`，视频页 `getSelectedVideo()` 取。模块状态在**同一个 JS 上下文**里，`player.js` 早就靠它跨页面保持播放状态（真机一直是好的）—— 这条一定通。
2. **仍然按可靠程度多试几条**：`$page.options` → `$page.params` / `$falcon.query`，并且**把命中的来源写在界面上**（`来源：首页选中的条目（模块状态）`）。三条都取不到时，那一行会把试过的来源全列出来，下次出问题一眼就知道该修哪条，而不是又只看到一句「没有视频路径」。

两个 bug 都有回归测试：`.tools_tmp/test-video-probe.js`（13 条断言）—— 覆盖 `runProbeCommands` 的六种返回形状（字符串 / output / stdout / 空 / 超长 / 抛异常），`resolveSource` 的四条来源路径逐条断言。

#### 18.6 真机探测结果：**能放，但要绕开渲染器**

用户在真机上跑通了「视频能力探测」，八张截图给出的结论非常明确：

| 探测项 | 结果 | 意义 |
|---|---|---|
| 显示服务 | `weston --tty=2 -B=drm-backend.so`（+ weston-desktop-shell） | **有真正的 Wayland 合成器在跑** |
| GPU / DRM | `/dev/dri/card0 card1 renderD128 renderD129`、`/dev/mali0` | 有 DRM/KMS 设备 + Mali GPU |
| 显示相关库 | `libEGL.so`、`libGLESv1_CM.so`、`libGLESv2.so`、`gstreamer-1.0` | GL 栈与 GStreamer 插件目录都在 |
| GStreamer 视频元件 | `autovideosink` / `kmssink` / `waylandsink` / `videoconvert` / `playbin` | **视频链路完整** |
| 能放视频的程序 | `gst-play-1.0`、`gst-launch-1.0` | **现成的播放器** |
| ffmpeg | 4.1.3，构建前缀里写着 `Cherry3566`（Rockchip RK3566） | 带硬解单元的 SoC |
| 运行时里的 video 组件 | （无输出）+ 视频页画面区全黑 | **渲染器确实没实现 `<video>`** → 组件路线作废 |
| 机器上的视频 | `/userdisk/Video/1.mp4`、`1.avi`、`/userdisk/Favorite/bili/…mp4` | 有现成素材可测 |

**结论：视频播放能实现，但必须绕开 miniapp 的渲染器** —— 走「用 exec 拉起系统播放器，让它把画面送到 Wayland/KMS」，也就是和音频（`ffmpeg | aplay`）完全同一套机制，只是这次换成 GStreamer 送画面。

#### 18.7 系统播放器路线（视频页新增两个按钮）

- **「系统播放器」**：拼一条 shell 命令后台拉起 `gst-play-1.0 --no-interactive <路径>`（没有它就退回 `gst-launch-1.0 -q playbin uri=<file://…>`），启动后把日志前几行直接显示在页面 —— 成没成一目了然。
- **「停止」**：`pkill` 掉两种播放器；进页面时也会静默清一次，避免上次那支还占着屏。

命令构造单独放在 `services/video.js`（纯函数），因为有几个坑必须一次处理对：

| 坑 | 处理 |
|---|---|
| 素材名里有中文、空格、括号，甚至单引号 | 单引号转义（`'` → `'\''`）；两种播放器**命令形状不同**（gst-play 收路径，gst-launch 收 `uri=`），分别拼 |
| 显示环境（Weston 的 socket 名各固件不同） | 命令行里现场找 `/run/user/0/wayland-*`、`/run/wayland-*`、`/tmp/wayland-*`，补 `XDG_RUNTIME_DIR` / `WAYLAND_DISPLAY` |
| 这一句 exec 会一直阻塞到播放结束 | `nohup … &` 后台跑 + 日志落 `/tmp/mlp-video.log` |
| 播放器可能把屏幕占死 | 套 `timeout 900` 兜底 |

**转义这件事做了真验证**：Windows 侧没有 POSIX sh，所以测试会生成一个脚本，开头放一个假的 `gst-play-1.0`（只把收到的参数逐行打出来）并加进 `PATH`，再把**生成的命令行原样**在 WSL 里跑一遍。实测四种刁钻路径（空格+括号 / 单引号+空格 / 中文+括号）都完整作为**单个参数**送达：

```
### case2  player=gst-play-1.0 pid=909
ARG=[--no-interactive]
ARG=[/userdisk/a'b c.mp4]          ← 单引号 + 空格，没被切断
```

（第一版的 `gst-launch` 分支写成 `playbin uri=file://$path`，属性与路径被拆成两个参数 —— 是写测试时才发现的。）

### 19. 音源插件系统（MusicFree 协议移植）

> 这一节记的是**新增**的一整套东西：让本应用能直接跑 MusicFree 的音源插件。它与首页/播放页/设置页的现有功能**没有耦合** —— 页面里还没有任何地方调用它，属于「地基先打好，接线另说」。

#### 19.1 红线：只新增，不动现有播放

- `ui/src/app.js`、`ui/src/base-page.js`**一个字都没改**（构建器管着这两个文件）；
- `ui/src/services/player.js`**一个字都没改**。播放队列只在需要时调用它的公开方法（`setPlaylist` / `playAt` / `toggle` / `next` / `seek` / `setRate` / `on('stateChanged')`），而且这条通路是可选的（见 19.7）；
- 原生侧唯一碰过的既有文件是 `jsapi/src/Fetch.cpp` 的一个回调（见 19.3），而 `Fetch` 在这之前**根本没有导出给 JS**，等于改的是死代码。

#### 19.2 为什么非要自己写一套运行时

MusicFree 的插件是**不按 React Native 写的**——插件源码里默认这些东西存在：

| 插件里的写法 | 本机现实 | 顶上的东西 |
|---|---|---|
| `require('axios')` + `axios.get(...)` | 没有 window / fetch / XHR | `adapter/http.js` + `adapter/axios.js` |
| `require('crypto-js')`（签名、解密） | 没有 crypto-js | `vendor/hash.js` + `adapter/require.js` 的 CryptoJS 外观 |
| `require('qs')` / `require('he')` / `require('dayjs')` | 没有 | 自带 `qs` / `he`，dayjs 写了个够用的子集 |
| `storage.set('k', v)` 同步读写 | `$falcon.jsapi.storage` 是异步的 | `adapter/storage.js`（内存同步 + 防抖落盘） |
| `env.getUserVariables()` | —— | 沙箱注入，值来自插件清单 |

白名单里**明确没提供**的包：`cheerio`、`big-integer`，以及 `crypto-js` 的 AES / SHA1 / SHA3（只实现了 MD5 / SHA256 / HmacSHA256 / 编码）。用到它们的插件会在**调用点**拿到一句中文报错（`missingFeature()` 返回一个「既能被调用、又带着常用方法」的替身），而不是含糊的 `undefined is not a function` —— 少一个包要能一眼看出来。

#### 19.3 原生 Fetch（`custom.Fetch`）

**这台机器上 JS 侧拿不到任何 HTTP 能力**：系统模块只有 `global` / `http`，而 `http` 在部分固件上压根不存在。音源插件全靠联网，所以网络必须由原生提供。新增 `jsapi/src/Net/JSFetch.hpp/.cpp`，导出成私有模块 `Fetch`：

| 方法 | 行为 |
|---|---|
| `fetch({url, method, headers, body, timeout, followRedirects})` | **阻塞**实现，实现简单、行为可预测；请求期间占用 JS 线程 |
| `fetchAsync({..., id})` | 后台线程做，完成后用 `publish('fetch_done', …)` 推事件（**不卡界面**，首选） |
| `get(url)` / `post(url, body)` | 语法糖 |
| `getModuleInfo()` | 回读 `BUILD_TAG`，用来确认装到机器上的到底是哪一版 |

- 超时默认 10 秒，上限 120 秒；
- 返回 `{status, ok, headers, body}`，**头名一律小写**；同名多值头用 `\n` 连接（`Set-Cookie` 需要，`Fetch.cpp` 的 `HeaderCallback` 由「后来者覆盖」改成「追加」—— 这是唯一动过的既有代码）；
- `BUILD_TAG = "2026-09-30 custom-fetch"`，同一串也写进了 `JSFileShell.cpp` 的 `BUILD_TAG`，所以 `FileShell` / `AudioBridge` / `Fetch` 三个模块能一眼看出是同一批编的。

`adapter/http.js` 把「能发请求的东西」按可靠程度排成阶梯逐个尝试，任何一层拿到结果就返回：`custom.Fetch.fetchAsync`（超时回退）→ `custom.Fetch.fetch` → 系统 `http` → 明确报错。诊断页的「自带模块」能看出这条链路最后选的是哪一级（`getTransportInfo()`）。

#### 19.4 沙箱：一个「返回值被丢掉」的坑

插件按 MusicFree 的签名执行：

```js
new Function("'use strict';return function(require, __musicfree_require, module, exports, console, env, process){"
  + 插件源码 + "};")
```

**外层 `new Function` 只是「造函数」，得再调一次内层，插件代码才会执行。** 第一版写成 `factory(require, require, module, …)` —— 参数全喂给了外层，返回值（内层函数）被丢掉，于是 `module.exports` 永远是空的，每个插件都报「缺 platform 字段」，症状和「插件代码写错了」一模一样。`test-core.js` 里那 4 条断言（挂载成功 / platform / version / supportedMethods）就是盯这一步的。

沙箱注入的东西与 MusicFree 对齐：`require`（白名单）、`env.getUserVariables()`、`process.env.appVersion`、`console`（转发到宿主日志并带 `[plugin:平台名]` 前缀）。坏插件**不会带崩音源列表**：语法错误、缺 `platform`、取址失败都只落到该插件自己的 `state` / `error` 字段上。

#### 19.5 五级取址回退（`core/plugin.js`）

插件给出的播放地址会过期，所以取址必须按可靠程度逐级退：

| 级 | 条件 | 说明 |
|---|---|---|
| 1 | 内存缓存命中（30 分钟 TTL） | 且 `cacheControl !== 'no-cache'`；按 `platform + 主键 + 音质` 分开缓存 |
| 2 | 插件 `getMediaSource(item, quality)` | 标准通路 |
| 3 | 条目自带的 `qualities[quality].url` | 插件抛错时的兜底 |
| 4 | 失败**重试一次** | 仅当 `err.code === 'NOT RETRY'` 时**不重试**（版权受限这类重试没意义） |
| 5 | 成功结果写回缓存 | 按 `cacheControl` 决定 |

第 4 级这条**必须验「真的重试了」而不是「结果对」**：测试的假插件按条目分别记调用次数，普通错误断言调用 **2** 次、`NOT RETRY` 断言只调用 **1** 次、一直失败断言上限也是 **2** 次。（第一版假插件用的是全局计数器，那三条断言其实一直是在「白过」—— 计数器在别的用例里已经涨上去了。）

#### 19.6 插件目录：用内容指纹当文件名

```
<存储目录>/plugins/
  ├── index.json              # 清单（platform / hash / file / version / srcUrl / enabled / userVariables）
  ├── 30c5dbbed882e3f6.js     # 文件名 = 源码 sha256 的前 16 位
  └── 308263bd34e82772.js
```

- **去重**：同一份代码装两遍（换个 URL、换个文件名）只留一份，第二次返回 `duplicated: true`，且**不当错误**（用户重复点安装是常事）；
- **更新**：先写新文件、清单写成功后再删旧文件，中途断电最多留个孤儿文件；
- **恢复**：`setup()` 除了读清单，还会扫目录，把「清单里没有但文件在」的插件认回来（写清单时断电的情形）；反过来「清单里有但文件没了」只标 `missing`，不悄悄删记录；
- **写盘一律回读校验**：这台设备的文件接口「不报错但没写进去」是有前科的（见 `services/store.js`）；文件系统只读时安装仍然成功，只是插件只活在本次会话里。

`platform` 名是插件自己写的，落盘前会做字符白名单处理（不能让它带着 `../` 当文件名）。

#### 19.7 播放队列：两种交接，避免「一次播完跳两首」

`core/trackQueue.js` 面对两种播放引擎，用同一个适配器接口对接：

| 交接方式 | 什么时候用 | 谁负责切下一首 |
|---|---|---|
| `playlist` | 适配器有列表能力，**且这一批曲目全都有本地路径** | 现有播放器（它有完整的顺序/随机/单曲逻辑），队列只镜像它的 `index` |
| `single` | 曲目来自插件（地址要现取、会过期） | 队列自己算下一个下标，一首一首交 |

这个区分是必需的：现有播放器**自己**会在播完后切下一首，队列要是也跟着切，就是一次播完跳两首。适配器的 `capabilities.autoNext` / `capabilities.playlist` 把差异收在一处，队列里只有 `next()` 和状态镜像两处需要看它。

`single` 交接下「放完了」要认两种形态，缺一个就会漏切：

1. **停在结尾**：位置到达时长且 `isPlaying` 变假（顺序模式放完最后一首、引擎播完即停）；
2. **自己重播**：列表循环单曲模式下播放器从结尾跳回开头（位置归零），这也要算「播完」。

事件：`state` / `trackChanged` / `complete` / `progress` / `error`，其中 `complete` 同时以 `player:complete` 再发一次（方案里是按这个名字写的）。`custom.Player` 还不存在时，适配器降级成 `noop`：错误信息说清缺什么，不会静默什么都不做。

#### 19.8 体积代价

| 文件 | 大小 | 说明 |
|---|---|---|
| `vendor/he.js` | 100 KB | HTML 实体解码，**是这一层最大的单项开销**；只有少数插件真用它 |
| `vendor/qs.js` | 19 KB | query 序列化 |
| `vendor/hash.js` | 8 KB | 自带 md5/sha256，换来不引 crypto-js 整包 |
| `adapter/` + `core/` | 约 110 KB | 源码未压缩 |

`he` 与 `qs` 都取自 `aiot-vue-cli/node_modules`（MIT），已验证不用 `?.` / `??`。若后面确认没有插件用 `he`，把它从白名单里摘掉、改由 `missingFeature()` 提示，能直接省下 100 KB。

#### 19.9 验证情况

```bash
node .tools_tmp/test-hash.js     # 65 条：md5/sha256 与 Node crypto 逐位对齐（含分块边界）
node .tools_tmp/test-adapter.js  # 63 条：HMAC / axios 语义 / storage / require 白名单
node .tools_tmp/test-core.js     # 186 条：沙箱 / 五级回退 / 歌词 / 管理器 / 队列 / 歌单导入
node .tools_tmp/_probe-bundle.js # 一次性探针：把新模块过一遍项目自己的 rollup 配置
```

**为什么还需要那个探针**：这些文件目前没有被任何页面 import，所以正常构建**根本不会解析它们** —— `node tools/build-ui.js` 成功并不说明它们能进包。探针把 `adapter/` `core/` `vendor/` 全量 import 进一个临时入口，用**同一份 rollup 配置**打一遍。实测结果：构建通过；`await import(/* @vite-ignore */ 'custom')` 原样保留（与 `services/platform.js` 现在的行为一致，构建只给一条「未找到以下模块」的警告）；`await import('../services/player.js')` 被拆成一个独立 chunk（现有播放器里 `await import('./library.js')` 一直就是这么跑的）。探针产出的 `_probe_out/` 只是用来看的，验证完可以直接删掉。

测试写法与其它测试一致：**剥掉 `import`/`export`，把依赖作为 stub 注进去，跑真实源码**。`test-core.js` 里的假插件、假文件系统、假播放器都刻意做成「可断言」的：文件写入会被逐条记录（好断言「旧文件已删、新文件在」）、播放器状态可以被主动推一次（好断言「播完只切一首」）。**变异验证**：把队列里识别「播完」的那个分支改成恒假，`test-core.js` 立刻报 8 条失败 —— 说明这些断言真的在盯逻辑，而不是恒真。

**还没在真机上验证的**（都属于「接线之后才能验」）：

1. `custom.Fetch` 在这台笔上的实际联网结果（诊断页可以看到 `Fetch` 是否加载、`BUILD_TAG` 是否对）；
2. 插件列表落盘的真实路径（`<存储目录>/plugins/`，跟 `services/store.js` 选中的目录走）；
3. 把插件曲目交给现有播放器时，引擎能不能直接播 `http://` 地址（ffmpeg 编译时带了 `--enable-network`，理论上可以，但没有实测过）。

### 20. MusicFree 模式（实验室 → 在线音源）

> 19 节那套运行时是「地基」，这一节是**把它接到界面上**：设置里一个开关，首页就能装音源插件、搜在线歌曲、点一首在线播放。

#### 20.1 怎么开、开了之后是什么样

**设置 → 实验室 → MusicFree 音源**（开关写进 `mlp_settings_v1`）。是否生效只有一处判断：`services/library.js` 的 `isMusicfreeOn()` —— 必须「实验室 + MusicFree」都开着，和视频模式一个规矩。

开了之后首页变成：

| 位置 | 本地模式（默认） | MusicFree 模式 |
|---|---|---|
| 标题 | 本地音乐 / 本地视频 | MusicFree 在线音源 |
| 右上角 | 25 首音频 | 2 个音源 / 30 条结果 · 第 1 页 |
| 列表 | 曲库卡片 | 音源列表 或 搜索结果 |
| 放大镜 | 过滤本地曲库 | 搜索在线歌曲 |
| 双向箭头 | 排序方式 | 在线音质（标准 / 高 / 无损） |
| 刷新 | 重扫曲库 | 重扫插件目录 |
| 文件夹 | 切换扫描目录 | 导入音源插件 |
| 折叠箭头 / 齿轮 | 折叠播放条 / 设置 | 一样 |

**没有插件时**首页只有一个「导入音源插件」按钮，输入框同时收两种东西：`http(s)://…/xxx.js` 当成插件地址联网拉取，其它内容当成插件源码直接装（贴代码这条路在没有网络的机器上也用得上）。

**三种模式互斥**（首页只有一个列表位置）：打开 MusicFree 会顺手关掉视频模式，打开视频模式也会关掉 MusicFree；实验室一关，两个都跟着关。

#### 20.2 本地播放仍是主模式（这次没有改它）

- 默认关。`services/player.js`、本地曲库扫描、播放页、视频页**一行都没改**。
- MusicFree 那一整套运行时代码走**动态 `import()`**，构建出来是独立 chunk（`musicfree-*.js`，约 230 KB，含插件沙箱 / qs / he / axios 外观）：**本地模式下这个 chunk 根本不会被加载**，内存里也不会多出这些东西。
- 关掉模式时，如果当前正在播在线曲目，会停掉并清空播放列表 —— 否则现有播放器会把这条在线曲目持久化（`player_last_state`），下次启动时本地模式里会冒出一条放不了的曲目。本地曲目一律不碰。

#### 20.3 在线音频怎么放：两条路

| 路 | 什么时候用 | 怎么放 |
|---|---|---|
| ① 直链 | 插件没要求自定义请求头 | 把插件给的 URL 直接交给**现有引擎**：它就是 `ffmpeg -i <地址> \| aplay`，ffmpeg 带 `--enable-network`。秒开、能拖进度。PulseBox 就是这么放的（`$falcon.soundPlayer.play(streamUrl)`） |
| ② 下载后播 | 插件要求 Referer / UA / Cookie，或①放不出来 | 用原生 libcurl 把文件下到缓存目录，再把**本地文件路径**交给同一个引擎 —— 走的是已在真机上验证过的本地播放链路 |

**「①放不出来」是有依据的判断，不是猜**：原生 `AudioBridge::tryLaunchLocked()` 拉起来之后要等 180ms 做存活确认，ffmpeg 打不开地址会立刻退出 → `play()` 返回 false → `player.setPlaylist()` 返回 false → 我们据此改用②。测试里用「第 1 次交接返回 false」把这条路完整跑了一遍（见 20.10）。

**下载为什么必须由原生做**：`Fetch` 的返回体是**字符串**（Bson string 按 UTF-8 处理），音频这类二进制过一遍字符串会被替换字符破坏。所以 `custom.Fetch` 新增了 `download` / `downloadAsync`（libcurl 直接写文件）、`download_progress`（进度事件）、`cancelDownload`（取消）。都支持自定义请求头，并带低速保护（30 秒内平均低于 1KB/s 判失败，免得网络断了却一直挂着）。

缓存落在 `<应用数据目录>/online/`，文件名是「平台 + 条目 id + 音质 + URL 尾部」的哈希：同一首第二次播放直接命中，不重复下载；目录里超过 40 个文件就按修改时间删最旧的。

#### 20.4 播放交接：整份列表 + 共享对象

`musicfree.playRows(rows, index)` 做三件事：

1. 解析当前这首的地址（插件 `getMediaSource` → 五级回退，见 19.5）；
2. 把**整份结果列表**交给 `player.setPlaylist(list, index, true)` —— 上一首/下一首/随机/单曲、倍速、拖动进度、蓝牙输出，全部原样复用；
3. 列表里其余条目**先留空 path**，随后在后台按顺序补齐（一次一个、最多向前 4 首）。之所以能这样「先交列表后补地址」：`setPlaylist` 存的是**同一批对象的引用**，我们改 `track.path` 它立刻就能读到 —— 和现有 `attachLibrary()` 是同一个套路。

万一播放器跳到了一首还没解析的曲目（用户直接点列表靠后的一首），`trackChanged` 钩子会立刻解析并重开这一首。

#### 20.5 封面与歌词

- **封面**：插件给的封面地址会被下到本地换成 `file://`（渲染器的 `<image>` 认本地文件最稳），路径写回同一条 track 对象，播放页/悬浮条自动跟着显示。
- **歌词**：在**打开播放页之前**取回来、写成 `.lrc` 落到缓存目录，再把路径挂到 `track.lyricPath`。原因是播放页只在挂载时读一次歌词文件（`loadTrackAssets` 里按 `track.path` 去重），晚到的歌词不会被重新加载。附属资源最多等 2.5 秒，超时就先起播。

#### 20.6 这次修掉的原生大坑（值得单独记一笔）

**`Net/JSFetch.cpp` 从来没被编进过 `.so`。**

- `tools/build-native.sh` 的源文件列表只有 SDK / `Shell/` / `Media/` / `JSAPI.cpp`，`Net/` 目录压根不在里面；
- 而链接带了 `-Wl,-unresolved-symbols=ignore-all`，**未定义符号会被放行**，`.so` 照样产出、构建照样报成功；
- 之前的「校验」用的是 `strings | grep custom-fetch` —— 那个字符串其实是 **FileShell 的构建标记**，于是「有 tag」被当成了「Fetch 编进去了」（我自己就被骗过一次）；
- 后果不是「少个功能」：`JSAPI.cpp` 里 `createFetch(env.get())` 是**无条件调用**的，装到机器上 `import 'custom'` 时会调到空地址。

现在改成三重保险：

1. 源文件列表补上 `Net/*.cpp` + `Fetch.cpp` + `strUtils.cpp`；
2. `build-native.sh` 末尾新增**符号自检**：把 `JSAPI.cpp` 里所有 `createXxx(env.get())` 抓出来，逐个确认在 `.so` 里有定义、且不是未定义引用，缺一个直接 `exit 1`（顺带过滤掉注释行 —— 第一版把注释掉的 `createAI` 也算进去了）；
3. `verify-pack.js` 直接读**包里** `.so` 的字节，验 `FileShell / AudioBridge / fetch_done / download_done / download_progress / 构建标记`。

顺带一提：`JSFetch.cpp` 其实**从来没编译通过过**（少了 `ASSERT` 的 include、`{"size", long}` 在 Bson 上是歧义转换）—— 它一真的参与编译就报错了。这也从侧面说明「没编进去」这件事有多隐蔽。`.so` 因此从 724,840 涨到 974,792 字节（aarch64）。

#### 20.7 插件放在 /userdisk/musicfree（**看得见的地方**）

插件目录**不是**应用私有目录，而是用户能看见、能手动放文件的公共路径：

| 顺序 | 目录 | 说明 |
|---|---|---|
| 1 | `/userdisk/musicfree` | 首选。找个文件管理器就能看见装了什么、删掉不想要的 |
| 2 | `<数据目录>/plugins` | 首选建不出来 / 写不进去时退到这里 |
| 3 | `/tmp/musicfree` | 最后一招（重启会丢，但总比装不上强） |

判定方式：对每个候选**真写一个探针文件再删掉**（`.mlp-write-test`）——「目录存在」不算数，写得进去才算。诊断页的「音源探测」会把最终目录、候选列表、目录里现有文件全列出来。

**联网不通时的兜底**：把插件 `.js` 拷进 `/userdisk/musicfree`，回首页点「重新扫描插件目录」就能用 —— 这条路完全不联网。（扫描会把目录里「清单没记录过」的 `.js` 认领进来，点开头的临时文件会跳过。）

清单 `index.json` 也在这个目录里，和插件文件放在一起。

#### 20.8「一直显示下载中」是怎么修掉的

现象：导入插件时提示条永远停在「正在下载插件…」。**根因是异步取结果的通路。** `custom.Fetch.fetchAsync` 的结果原本靠「后台线程 publish 到 JS 线程」的事件送回来，而这条通路在这台设备的 miniapp 运行时里**并不可靠** —— 应用里所有原生调用都是同步的，连音频进度都是 JS 侧每 250ms 轮询出来的。事件不到，适配层只能干等 22 秒，再退化成**同步** fetch 把 JS 线程阻塞 20 秒，界面看起来就是死在那儿了。

四处改动：

| 改动 | 说明 |
|---|---|
| 原生加**取件箱** | `Fetch.pollResult({id})` 同步取走异步结果；`Fetch.pollDownload({id})` 读下载进度。不依赖任何事件机制 |
| 适配层改**轮询** | `fetchAsync` 受理后每 200ms 轮询一次；事件到了就少等一个周期。旧 .so（没有 pollResult）只等 3 秒就改走同步，不白等 |
| **每层硬超时** | `import('custom')`、`fetchAsync` 受理、请求本身、同步兜底、下载——每一层都套了 `withTimeout`，任何一层不返回都会变成一条具体错误 |
| 装插件改**同步下载** | 下到 `/userdisk/musicfree/.incoming-*.js` → 读回来校验 → 按内容指纹改名。一步到位、二进制安全、**完全不碰事件** |

界面也跟着改了：提示条带**秒表**（超过 3 秒显示「（已 N 秒）」），并且安装会分步显示（下载 → 读取 → 检查 → 写入），卡住时能直接看出卡在哪一步。诊断页新增「音源探测」按钮，逐层跑一遍：加载 `custom` → 读 Fetch 版本 → 同步 fetch → 异步 fetch + 轮询 → pollResult 是否可用 → 下载到插件目录，每层都带结果与耗时。

#### 20.9 已知限制

1. 直链能不能放，取决于 ffmpeg 的协议支持（http 应该没问题，https 要看有没有 TLS）与服务端是否强制校验 Referer；
2. 走下载那条路时是**整首下完再播**，不是边下边播 —— 几 MB 的歌在慢网络上要等，界面上有百分比提示；
3. 歌词/封面如果超过 2.5 秒才回来，本次播放页不会显示（下次播放会命中缓存）；
4. 播放页离开时会调 `player.rememberDuration()`，在线曲目会往本地时长缓存里写一条 URL 键（无害，但确实是脏数据）；
5. 插件里 `require('cheerio')` / `big-integer` / AES / SHA1 仍然没有（会明确报错，见 19.2）；
6. 在线歌曲可以**显式**下载进音乐目录（播放页的下载按钮 + 批量下载，见 20.11），但**不会自动进本地曲库** —— 本地曲库仍然只认同名封面/歌词那一套约定，下次扫曲库时才会把它们当成普通本地文件收进来；
7. 歌单**只记来源与链接**，不存曲目：几百首的曲目表会把这台设备的私有存储写爆，所以每次点开歌单都是现取（20.11）。

#### 20.10 验证情况（插件与在线播放这一层）

```bash
node .tools_tmp/test-adapter.js     # 69 项（axios 与沙箱外观；含 20.16 那一组）
node .tools_tmp/test-core.js        # 196 项（含 isEnd 默认值这一组，见 20.14）
node .tools_tmp/test-musicfree.js   # 170 项（原 62 项，见 20.13）
node .tools_tmp/verify-pack.js      # 交付前核对（含 .so 内的 pollResult / pollDownload）
```

`test-musicfree.js` 覆盖：设置门控（三种组合）、行/播放器条目映射（时长毫秒归一化、key 跨音源不撞）、取址阶梯（无头→直链不下载 / 有头→下载 / 二次解析命中缓存 / 并发只取一次 / 失败状态与原因）、播放交接（整份列表 + 后台补齐下一首的路径）、**直链失败 → 下载后重放的完整退路**、封面与歌词的落盘、切回本地模式时只停在线曲目。

`test-adapter.js` 新增的一节专盯这次的 bug：**永远不返回的 Promise 必须在期限上失败**、异步不给结果时自动改走同步（1 秒超时 → 约 4 秒内出结果，且后续请求不再重复等）、`fetchAsync` 不受理时同样落到同步、旧 .so（没有 `pollResult`）只等 3 秒就改道、下载进度从 `pollDownload` 读出来、同步下载卡住时按 `DOWNLOAD_TIMEOUT` 失败。

`test-core.js` 新增的一节专盯安装路径：目录是 `/userdisk/musicfree`（先写探针验证可写）、装插件走原生下载且用的是**同步**那条、先落 `.incoming-` 再改指纹名、临时文件清理干净、安装过程分步回报、下载失败不留垃圾、首选目录写不进去时退到私有目录。

**变异验证**：把「直链失败 → 退回下载」那个分支改成恒假，`test-musicfree.js` 立刻报 4 条失败 —— 说明这条路真的被测住了，不是摆设。

**还没在真机上验证的**：`custom.Fetch` 的实际联网。装上新包后按这个顺序试：① 诊断页点「音源探测」—— 六层全绿说明原生网络这一层通了；② 首页导入插件 —— 提示条会分步走完并说出插件名；③ 搜一首歌 —— 有结果说明插件沙箱 + 插件接口都通；④ 点一首 —— 出声说明在线播放两条路至少通了一条。哪一步卡住，提示条上的秒表与诊断页那几行会直接指向是哪一层。

#### 20.11 音源筛选按钮组 / 歌单页 / 批量下载

用户要的三件事：搜索页标题栏下面加一条**按音源筛选**的按钮组、新增一个**歌单页**（能导入各插件支持的歌单）、搜索页与歌单页侧边加**下载按钮**（点一次选歌、选完再点一次批量下载）。

**一、音源按钮组（`ui/src/pages/index/index.vue`）**

- 位置：`.head`（标题栏）下面、搜索结果上面，横向 `scroller` 里一排按钮，第一个固定是「全部」（`value: ''`），后面每个插件一个，文案取插件自己的名字。
- **为什么用横向滚动而不是换行**：装十几个插件时按钮会一路排到屏幕外，靠滑动看，而不是挤成两行把标题栏下面的空间吃掉。
- 选中态用**内联样式**而不是两个类名 —— 本平台样式按单类名注册，同类名规则谁后加载谁生效，`on` 状态放在类名上就是赌加载顺序。
- 点某个音源 → `mfPickSource(value)`：同一值早返回；换音源会**清掉已勾选的歌**（换了一批结果，之前勾的已经不在列表里了）并立刻重搜第 1 页。
- 搜索走 `musicfree.search(keyword, page, { platform })`：指定音源走 `pluginManager.searchPlatform()`（单音源，**失败直接抛**，用户明确点了某个音源就必须看到原因），空串走原来的 `searchAll` 并发。

**二、歌单页（`ui/src/pages/sheets/sheets.vue`，`app.json` 里注册为 `sheets`）**

- 入口：设置页「在线音源」分组 →「我的歌单」。
- 导入：粘贴音乐 App 的分享链接 → `musicfree.importSheet(url)` → `pluginManager.importSheetFromUrl()` **依次**把链接交给每个实现了 `importMusicSheet` 的插件（串行：五路网络同时打会互相拖死）。**谁认领算谁的** —— 哪条链接归哪个音源是插件的私事，用户手上只有一条链接。全失败时把每个插件的原话一起显示出来，而不是只说「失败」。
- 存储：数据目录下 `musicfree-sheets.json`（+ KV 键 `mlp_musicfree_sheets_v1` 兜底），与插件的 `index.json` **分开**（插件目录是用户会手动翻看的公共位置，混进去会被误认为插件文件）。**只存来源插件 + 链接 + 插件吐回的 sheetItem + 标题/曲数/封面。**
- 取曲目：点开歌单才 `musicfree.sheetTracks(sheet, page)` → `pluginManager.fetchSheetInfo()`；插件补齐的 `sheetItem` 会写回记录，到底了（`isEnd`）也写回，下次不用再试。
- 删除要二次确认（`confirmText` 用一次文本输入代替 confirm：输入 1/y/yes/是 才算确定）。

**三、批量下载（服务层 `musicfree.downloadRows(rows, options)`）**

- 两段式：**第一次**点侧边下载按钮 → 进选择模式（点歌勾选，勾选不会触发播放）；**再点一次** → 开始下载。已选 0 首时再点 = 退出选择模式。
- **串行**下载（`for` + `await downloadView`）：这台设备带宽与插件限流都扛不住并发。进度写在那条提示里（「第 i/n 首、百分比」）。
- **同批重名去重**：先按 `安全文件名` 算一遍，重名的跳过并说明「与第 N 首《xxx》文件名相同」。原因：`downloadView` 遇到**已有同名文件**会返回 `{ok:true, exists:true}`，不先去重的话「选 8 首下了 6 首」会被报成 8 首成功。
- 单首失败不拖累整批，结束时给汇总：`共 N 首：新下载 a 首、已有 b 首、重名跳过 c 首、失败 d 首`。
- 下载完**自动退出选择模式并清空勾选** —— 留在选择模式下用户顺手再点一次，会把整批又下一遍。
- 歌单页复用同一个 `downloadRows`，选择逻辑与首页一致。

#### 20.12 蓝牙采样率改成 48000（与 `icon/bt.sh` 一致）

`jsapi/src/Media/AudioBridge.cpp` 的蓝牙分支原来固定成 `-ar 44100 ... | aplay -q -r 44100`，而 `icon/bt.sh` 用的是 48000。现已统一成 48000（`-ar 48000` 与 `aplay -r 48000` **必须同时**改，只改一边等于让 ffmpeg 与 aplay 吵架）。

**注意不能用 `sampleRate_`** —— 那是源文件采样率，随歌变化，让蓝牙输出跟着变只会更容易卡。

> **踩过的坑（很容易误判成「代码没生效」）**：`.so` 是**预编译产物**，`node tools/build-ui.js --pack` 只把 `ui/libs/` 拷进包。只改 `AudioBridge.cpp` 而没在 WSL 里重新 `bash tools/build-native.sh`，包里躺着的仍是旧 `.so`，蓝牙还是 44.1k。两个架构都要重建（机型架构未知时更该都建）。交付前用 `node .tools_tmp/check-so.js`（查 `ui/libs/`）与 `node .tools_tmp/check-amr-so.js`（查包里的字节）确认 `aplay -q -r 48000` 在、`44100` 一次都不出现。

#### 20.13 验证情况（本次改动）

```bash
node .tools_tmp/test-musicfree.js        # 170 项（新增 八、音源筛选 / 九、歌单 / 十、批量下载 / 指定音源导入）
node .tools_tmp/test-index-musicfree.js  # 51 项（音源按钮组、选择模式两段式、页码与勾选）
node .tools_tmp/test-index-page.js       # 41 项（本地模式不受影响：下载按钮会被过滤掉）
node .tools_tmp/test-core.js             # 186 项（新增 isEnd 默认值、歌单导入：指定音源）
node .tools_tmp/check-so.js              # 12 项（两个架构的 .so 里都是 48000）
node .tools_tmp/check-amr-so.js          # 6 项（打进包里的 .so 也是 48000）
node .tools_tmp/verify-pack.js           # 交付前核对（含新页面、新类名、包里 .so 的采样率）
node .tools_tmp/verify-download.js       # 29 项（批量下载接线 + 歌单页 + 包里 .so）
```

四套测试合计从 588 项加到 **710** 项（后又加到 **738**，见 20.15）。**这次测试逼出了三个真 bug**：① `musicfree.js` 的导入用了 `as` 别名（`searchPlatform as searchOnePlatform`），而测试的模块加载器是删掉 import 再按原导出名塞进 sandbox —— 别名会让调用点变成没定义的名字，运行期必炸；② `importSheet` 里插件吐回的条目常常不带 `platform`，`toRow` 出来的 key 会变成 `::s1`（跨音源撞车），已按「认领这条链接的音源」补上；③ **`isEnd` 的默认值反了**，见下面 20.14。

> 顺带记一条教训：`check-refs.js` 判断「这个名字有没有定义」时会**按逗号切分导入块**，所以**导入块里不要写注释** —— 注释里只要出现 `as` 字样就会把名字切坏，报出「未定义却被调用」的假警报。

#### 20.14 `isEnd` 不传 = 最后一页（协议默认值，曾经写反）

MusicFree 协议里 `search` / `getMusicSheetInfo` 返回的 `isEnd`，**不传时默认为 `true`**（即这一页就是最后一页）。官方协议文档原话：「不传时默认为 true」。

我们原来两处都写成 `!!(result && result.isEnd)` —— 插件**没返回**这个字段时算出 `false`，等于告诉界面「后面还有」。症状非常隐蔽：搜索页和歌单页会**一直显示「加载更多」**，点一次就再发一次请求、拿回同一页、再拼一次，列表越滚越长全是重复项，而且永远不结束。网易云那类老插件会显式返回 `isEnd`，所以只在「忘了写这个字段」的插件上才翻车 —— 属于那种自己测永远测不出来的 bug。

现在统一走 `ui/src/core/plugin.js` 里的 `readIsEnd(result)`：结果不是对象、或 `isEnd` 是 `undefined`/`null` 时返回 `true`，否则如实返回 `result.isEnd === true`。两处调用点都换成它，`ui/src/services/musicfree.js` 与 `ui/src/core/pluginManager.js` 里那些 `result.isEnd === true` 的严格比较不用动。

取「缺失 = 到最后一页」还有一层安全考虑：**真要分页的插件一定会自己写 `isEnd`**（它们靠这个翻页），所以这个默认值不会把正常插件截断，却能把「忘了写」的插件从死循环里救出来。

> **教训（怎么证明修对了）**：改完核心函数要做一次**变异测试** —— 把实现换回旧写法，确认新断言真的会红。第一版只看到「新断言通过」是不够的：断言可能根本没覆盖那条分支。这里变异回 `!!(result && result.isEnd)` 后，`test-core.js` 如期报 `2 项不通过（通过 167）`，`verify-pack.js` 也如期报两项红。
>
> 另一个坑：**构建失败时不要相信上一次的 `.amr`**。`build-ui.js --pack` 在源码有语法错误时退出码是 1（这点是好的），但如果不管退出码直接去查包，读到的就是上一版产物 —— 变异测试里我一度被这个骗过，以为「守卫没生效」。查包之前先确认构建退出码是 0。

#### 20.15 音源卡片上的「导入歌单」按钮 / 改名 PenMusic 1.1.6

**一、每个支持导入的音源，卡片上多一个「导入歌单」**

`ui/src/pages/index/index.vue` 的插件卡片（`.mf-actions` 那一行）现在是三个按钮：「停用/启用」→「**导入歌单**」→「删除」。中间那个带 `v-if`：

```
v-if="p.enabled !== false && mfCanImport(p)"
```

`mfCanImport(p)` 就是看 `getManagerInfo()` 给的 `p.methods` 里有没有 `importMusicSheet`（用 `indexOf` 不用 `includes` —— 这台设备的 JS 引擎版本不好赌）。**不支持的音源上不摆这个按钮**：摆一个点了必然失败的按钮比不摆更糟，用户会以为整个功能坏了。

点下去走 `mfImportSheetFor(p)`：弹输入框（标题写明「导入歌单到「<音源名>」」）→ `mf.importSheet(text, platform)` → 成功 toast「已导入歌单：<标题>（N 首）」，失败把插件原话打到 console。

**二、这里和服务层新增的 `platform` 参数配套**

`ui/src/core/pluginManager.js` 的 `importSheetFromUrl(urlLike, type, platform)` 与 `ui/src/services/musicfree.js` 的 `importSheet(url, platform)` 都多了第三个参数：

| 传法 | 行为 | 谁在用 |
|---|---|---|
| 不传 `platform` | 挨个问每个实现的插件，**谁认领算谁的** | 歌单页的「导入歌单」 |
| 传 `platform` | **只问那一个**；它不支持就报「不支持导入歌单」，不存在就报「没找到音源」 | 音源卡片上的按钮 |

指定音源时**刻意不退回**「挨个试」：用户已经点明了音源，再让别的音源半路把链接认领走，用户会看到歌单莫名其妙挂在另一个音源下。失败就是失败，报清原因。

> **实现约束（别踩）**：`getSortedImportablePlugins()` 是从 `getSortedSearchablePlugins()` 里筛出来的，而后者要求插件**实现了 `search`**。所以只实现 `importMusicSheet`、不实现 `search` 的插件现在不会被列进去（歌单页那条「挨个试」的路也问不到它）。音源卡片上的按钮用的是 `methods`，**不要求 `search`** —— 两处判据不完全一致，日后要收紧就统一到一处。

**三、改名与版本**

`ui/package.json`：`name` → `penmusic`、`appName` → `PenMusic`、`version` → `1.1.6`。appid `8001749598192572` **不动**（动了就是另一个应用）。`ui/src/pages/about/about.vue` 里 `appName()` / `versionText()` 的兜底值同步改（那两处的注释本来就要求跟 package.json 一致）。产物名是构建按 `appid + version` 现算的，所以包名自动变成 `8001749598192572.1_1_6.amr` —— **不要手改包里的 manifest**。

**两个「本地音乐」是故意留着的**：`ui/src/pages/index/index.vue` 的曲库标题（`{{ videoMode ? '本地视频' : '本地音乐' }}`）与 `ui/src/pages/player/player.vue` 无曲目时的显示名。它们指的是**本地模式**（与「本地视频」成对），不是应用名。

> **教训（verify-pack.js 里写死的 chunk 名）**：改名重打包后 `musicfree` chunk 的 hash 从 `musicfree-3e7e17c3.js` 变成 `musicfree-b5c28d57.js`，而 `verify-pack.js` 里 `texts['musicfree-3e7e17c3.js'] || ''` **静默变成空串**，三条 isEnd 守卫于是全红 —— 看起来像「代码里没有」，其实是「查错了文件」。这种假红比不查更坏：它会让人去改没坏的代码。现在统一走 `chunk('musicfree')` 前缀查找。**凡是在 `.tools_tmp` 脚本里按 chunk 全名取文本的地方都是同一个雷**。

**四、验证情况**

```bash
node .tools_tmp/test-core.js        # 196 项（新增 十一、歌单导入：指定音源 + 报错可读性 两组）
node .tools_tmp/test-musicfree.js   # 170 项（新增「指定音源导入」11 条）
node .tools_tmp/test-adapter.js     # 69 项（新增 axios 可调用这一组，见 20.16）
node .tools_tmp/verify-pack.js      # 全部通过（chunk 名改前缀查找之后）
node .tools_tmp/verify-download.js  # 29 项
```

九套测试合计 **754** 项（上一版 710）。新增断言都做过**变异测试**：把 `pluginManager.js` 里 `if (want)` 那个分支短路掉（退回「挨个试」），`test-core.js` 如期报 `8 项不通过（通过 178）`；把 `musicfree.js` 转调时的第三个参数去掉，`test-musicfree.js` 如期报 8 条红；把 `OPAQUE_MESSAGES` 清空，`test-core.js` 如期报 2 条红（见 20.16）；把 axios 的「可调用」拿掉重打包，`verify-pack.js` 的两条守卫全红。还原后全绿。

#### 20.16 `not a function`：axios 必须**自己能被调用**（真机 bug）

**一、症状与第一层困惑**

真机（PenMusic 1.1.6）上点中「元力QQ」这个音源，搜索直接报一句：

```
元力QQ 搜索失败：not a function
```

这句话**不带任何标识符** —— 既看不出是哪个插件、也看不出哪一行。（对比：`xxx is not a function` 至少还给个名字。QuickJS 对 `(0, f)(...)` 这种「取出来直接调」的形态就只报这三个词。）所以这一层的结论只能是「插件内部某次调用炸了」，翻不动。

**二、根因**

去把插件（`https://13413.kstore.vip/yuanli/qq.js`）下下来读：它是 TypeScript + Babel 压过的，里面写的是

```js
const res = (await axios({ url, method, data, headers, ... })).data
```

Babel 把它编成 `(0, _Zw.default)({...})`。`_interopRequireDefault` 对**没有 `__esModule` 标记**的模块是 `{ default: obj }`，所以 `.default` 就是整个 axios 对象 —— 那一句等于**把 axios 本身当函数调**。

而 `ui/src/adapter/axios.js` 原来导出的是一个**纯对象**（只挂了 `.get/.post/.request/.create/...`），自身不可调用 → 插件第一句请求就炸。

**为什么拖到现在**：走 `axios.get(...)` / `axios.post(...)` 的插件（网易云官方版就是）永远踩不到；只有把 axios 当函数调的写法才会翻车。

**三、修法**

`ui/src/adapter/axios.js` 的默认实例改成先声明函数、再挂方法：

```js
function axios(config) { return dispatch(mergeConfig(config)) }
axios.defaults = defaults
axios.interceptors = { ... }
axios.request / get / delete / head / post / put / patch
axios.create = function (config) { return createAxios(config) }
axios.default = axios        /* require('axios').default(...) 这条路也通 */
export default axios
```

两条约束：**不能用箭头函数**（要往它身上挂属性）；`create()` 出来的实例必须同样可调用（它走同一个 `createAxios`，天然满足）。

**四、回归测试**

- `test-adapter.js` 第三节加了 6 条，**直接按 Babel 的调用形态来测**：`typeof axios === 'function'`、`axios.default === axios`、`await axios({url, method:'POST', ...})` 真的发出请求且 body 是 `{"q":1}`、返回值与 `.get` 同形、`create()` 的实例也可调用。
- `test-core.js` 加了 10 条，盯**报错可读性**（见下面第五节）。
- `verify-pack.js` 加了 2 条，在**包里**盯形状：`axios.default = axios` 在不在、默认实例是不是「函数 + 挂方法」。

**五、顺带修的「报错看不懂」（`ui/src/core/pluginManager.js`）**

加了模块级 `describePluginError(err, pluginName)`：消息命中 `OPAQUE_MESSAGES`（`is not a function` / `is not defined` / `Cannot read propert…` / `undefined is not an object` …）时，把 `err.stack` 里**第一个栈帧**附在前面，形如 `not a function @ eval at mount (core/plugin.js:233:25), <anonymous>:14:5677（元力QQ）`。

用在三处，但**界面文案一个字都不改**（仍是 `<平台> 搜索失败：<原消息>`）：`searchPlatform` 的日志、`searchAll` 的 `errors`（这样「所有音源都搜失败了 —— …」里也能看出是谁、在哪一行）、`fetchSheetInfo` 的日志。插件自己写清楚的业务错误（中文那种）**不加栈、原样透传**。

**六、变异测试（证明这些守卫不是摆设）**

- 清空 `OPAQUE_MESSAGES` → `test-core.js` 如期 `2 项不通过（通过 194）`；
- 把 axios 的「可调用」拿掉重新打包 → `verify-pack.js` 如期 2 条红。

> **教训（第二次踩到，这次记牢）**：变异测试时**必须先看构建退出码**。中途一次变异让源码语法出错，`build-ui.js` **退出码 1**，而 `verify-pack.js` 读的还是**上一版 `.amr`**，于是「守卫全绿」—— 那是旧包在通过，不是新代码。先 `exit=0`，再看包。

**七、交付产物（本次）**

```bash
node tools/build-ui.js --pack      # 退出码 0
# → ui/8001749598192572.1_1_6.amr   941,152 字节 / 26 条目
```

> **教训（构建不可复现，别拿体积当证据）**：同一份源码（`ui/src` 最新一次改动停在 19:58）连打两次包，`musicfree` chunk 的 hash 会**变**（`musicfree-931e0fb7.js` → `musicfree-3989d589.js`），`.amr` 体积也跟着抖约 1 KB（942,043 → 941,152）—— hash 会写进文件名、又被别的 chunk 引用，于是散进几十个文件里。所以：**「体积变了」不等于「代码变了」**；判断包更没更新要看时间戳，判断内容对不对要用 `verify-pack.js` 去查。脚本里更不能按 chunk 全名取文本（见 20.13 那条教训）。

#### 20.17 白色音源按钮文字与蓝牙耳机播放/暂停（条件支持）

**音源按钮**：搜索页选中、未选中的文字都改为 `#ffffff`，只用背景色区分状态。`src/pages/index/index.vue`（约 L1599-L1602）中 `.mf-srcbtn-text` 同时明确指定白色，避免原生 `text` 不继承父容器颜色。首页回归和包内容检查都覆盖这两层。

**耳机输入链与音频输出不是一回事**：原来 `ffmpeg | aplay` 只能往蓝牙送声音，没有系统媒体会话，也没有已公开的 Falcon 耳机按键事件。本次增加的是实际原生链路：

```text
固件 BlueZ AVRCP / 蓝牙 HID → Linux /dev/input/event* → MediaKeyMonitor
  → custom.AudioBridge.on('mediaKey', ...) → 应用级 player 串行操作队列
```

- `jsapi/src/Media/MediaKeyMonitor.cpp` 只读、非阻塞监听带媒体键能力的蓝牙/AVRCP 输入节点，不抢占输入设备，不改权限，不启动常驻 shell 命令；每 2 秒重新发现节点，支持连接、断开和节点重用。停止/GC 时结束线程并关闭描述符。
- 映射 `KEY_PLAYPAUSE` → `toggle`，`KEY_PLAY` / `KEY_PLAYCD` → `play`，`KEY_PAUSE` / `KEY_PAUSECD` → `pause`。BlueZ 上游的 AVRCP 使用 PLAYCD/PAUSECD。只处理按下；释放、长按 repeat 和未释放前的重复按下不重复执行。
- `src/services/mediaKeys.js` 在应用级只绑定一次，切页或暂停不会取消耳机 PLAY。独立 PLAY/PAUSE 幂等，屏幕操作和耳机指令共用 `player` 队列，快速连按不重叠。恢复保存列表后的第一次 PLAY 会加载实际曲目，不会对空引擎调用 resume。
- 自研引擎暂停/恢复后回读 `getState().playing`，失败不假报成功；暂停或换歌后，尚未完成的旧轮询不能误判 EOS 自动切歌；应用退出使旧按键和旧 Promise 结果失效。
- 旧版 `.so` 或系统音频引擎没有这些 API 时安全降级，原有屏幕播放按钮仍可使用。蓝牙输出继续使用 **48000 Hz / 双声道 / S16_LE**，本次不改音频格式。

**固件边界（尚未真机验证）**：必须由固件把耳机指令暴露为可读的媒体键节点。只有 A2DP 出声、内核无 uinput、miniapp 无读取权限、或系统播放器先消费 AVRCP 指令时，本应用可能收不到按键；不能仅靠 JS 修补。未实现 BlueZ D-Bus 媒体会话注册，也不承诺固件冻结应用后的后台接收。这里只支持播放/暂停，不扩展上一首、下一首。

**真机验收**：安装新包并连接耳机后，播放一首本地歌或 MusicFree 歌曲。进入诊断页，查看「耳机按键监听」「耳机按键设备」「耳机按键记录」：

1. `running` 仅表示监听线程运行，`supported` 仅表示编译了后端，**都不代表收到耳机指令**。设备数应大于 0；权限错误和设备名会在同页展示。
2. 按耳机暂停/播放，重进诊断页或点音频探测刷新，确认记录次数增长、指令符合预期，且首页和播放页图标同步；暂停后再次 PLAY 应从原处继续。
3. 连按两次切换键、切到其它页面后按键、断开重连后按键；不应重复切换或误切下一首。
4. 若设备数为 0 或记录不增长，提供上述诊断行和音频探测中的「耳机按键」输出，以区分缺少节点、权限不足和固件未传递事件，不能把未收到事件说成已支持。

**本次离线验证**：原有 9 组 JS 测试 756 条；新增 controller/platform 测试 82 条、真实 player 模块的异步/队列测试 73 条，共 **911 条 JS 断言**。原生 decoder/设备识别/监听生命周期测试 **33 条**（没有模拟真实耳机节点）。ARM 与 ARM64 原生模块均实际交叉编译通过。安装包还逐字节核对两架构 `.so` 与当前 `ui/libs/`，并检查白字、按键 API、共享队列、轮询失效与诊断文案确实进包。测试通过不等于已经在耳机或设备上验证。

```bash
node .tools_tmp/test-media-keys.js
node .tools_tmp/test-player-media-keys.js
# 先交叉编译两个架构，再打包；仅打 UI 不会更新 .so
node tools/build-ui.js --pack
node .tools_tmp/verify-pack.js
```

本次最终构建及 `verify-pack` / `verify-download` 均退出 0（下载交付检查 29 项）。蓝牙 + 白字那一版的安装包为 PenMusic 1.1.6（`8001749598192572.1_1_6.amr`）：967,104 字节、26 条目。

```text
SHA256: 1756a3eb2146a772a607cdfece8622f9b809947c2ac6279a5cb560f132bee164
```

构建仍有主题未安装、SongRow 的历史 border 属性和原生运行时模块解析警告；无构建错误。元力QQ 的 callable axios 修复也已在本次包内核对，但真实网络和耳机事件仍需要设备验证。覆盖安装后应完全退出并重启应用，确保加载新的原生模块。（后来加了逐字歌词，包的体积与校验值变了一次，见 20.18 末尾。）

#### 20.18 卡拉OK歌词：开关、抗闪烁、真实逐字数据与类 Apple Music 动画

参考 [amll.dev](https://amll.dev/) 的数据模型，使用自己的零依赖解析器和 Weex 渲染器。时间单位为秒：`Line { time, endTime, text, trans, words? }`，`Word { startTime, endTime, word, spaceAfter }`。没有引入 AMLL 的浏览器渲染器。

**使用方式**：设置页 → 偏好 → **启用卡拉ok歌词**，默认关闭。开启并保存后才拆字上色；关闭时恢复原来的整行歌词，不创建逐字时钟。设置自动保存开启时即时保存；关闭自动保存时须点保存，未保存的草稿不会影响播放页。开关持久化为 `karaokeEnabled`，只认布尔 `true`；旧设置缺字段时仍为关闭。设置页保存后通过 `settingsChanged` 通知已打开的播放页，播放页挂载和 `onShow` 也会重新读取。

**这次排查出的根因：方括号内联的字时间戳被当成了普通 LRC。** LDDC 这类工具导出的是 `[00:02.803]Hear [00:03.467]the [00:03.740]melody[00:04.169]`，即「每个字前面一个方括号戳」。旧解析器只认行首第一个戳，剩下的戳各自又开了一行，于是**同一句歌词按字数重复出现好几遍、而且完全不逐字**。现在这种写法识别为 `lrc-inline`：行首戳是整句起点，行尾那个**没有文字的戳**是整句终点，中间每个戳切出一个字；相邻的重复戳（`[t][t]词`）表示空字或边界，跳过而不生成空节点。判据只看「相邻两个标签之间有没有字」，所以普通 LRC 的 `[t1][t2]同一句` 重复行不会被误判。

**普通 LRC 也逐字（按上下句时间插值）**：开关打开时，只有整句时间戳的行会用它到**下一句起点**之间的时间，按字数与拉丁词音节权重摊到每个字上（一个字最多 0.7 秒，摊满即止）；没有下一句时按字数兜底，长间奏里最后一个字之后保持已唱。这是**显示层的近似**，不是真实字时间轴：有真实字级数据时永远优先，离当前行超过 3 行的插值行不拆字，省文本节点。第三方音源插件仍须返回真实 QRC/YRC/KRC/增强型 LRC/TTML 字时间轴，不会修改其网络接口。

**这次修复的原因与机制**：

- 旧面板把平滑时钟放在响应式 data，每约 33ms 重绘整块歌词；当前行附近还会在整行 text 与拆字分支之间切换。现在高频状态隔离进 `src/components/KaraokeLine.vue`，非响应式私有时钟只在字的未唱/正在唱/已唱三态改变时发布 `states`，同一字区间不写响应数据。`src/components/LyricPanel.vue` 不再由逐帧时钟驱动整个面板或 translateY。
- 开关开启后，窗口内有 words 的行保持拆字分支，不随与当前行的距离反复重建；pieces 按文本、起止时间、空白的内容签名缓存，同内容新数组不会重置本地时钟。
- `src/core/plugin.js` 不再只返回 rawLrc/translation 丢掉其它字段。`src/core/lrcParser.js` 统一解开字符串、`{ lyric }`、`{ rawLrc }`、`{ content }`，在 ttml/qrc/yrc/krc/wordLyric/rawLyric/rawLrc/lyric/lrc 候选中，实际解析存在 words 才优先选择。翻译支持 translation/tlyric/trans，合并翻译不丢 words；不会把对象写成 `[object Object]`。
- `src/services/musicfree.js` 先发布内存歌词及资源 revision，再尝试写缓存。起播等待超时后晚到的歌词、暂停时晚到的歌词、缓存写入失败的歌词都可通知播放页刷新。
- 播放页按歌曲路径、歌词路径、revision 加载资源，不再只盯列表 index；先读歌词再等封面。同歌资源更新保持已有歌词，真正换歌才清空；generation + 完整资源 key 拒绝过期请求和 ABA 竞态。
- `src/services/library.js` 读取函数不再隐式修改 track.lyricPath：首次发现同名歌词从 result.path 返回，避免自身改变资源 key 后把读词结果误判为过期。

**解析范围与保真边界**（`src/services/wordLyrics.js`）：

- QRC/YRC 的圆括号字时间戳按绝对毫秒，KRC `<offset,duration,0>` 恒按相对行首换算。只支持已解码的文本 KRC 和裸 QRC，不解压/解密二进制 KRC，也不提取 QRC XML 属性包装。
- 增强型 LRC 支持尖括号 `[mm:ss.xx]<mm:ss.xx>字` 与**方括号内联** `[mm:ss.xx]字[mm:ss.xx]字`（LDDC 等工具导出）。内联判据：一行至少 3 个戳、时间非递减、相邻戳之间有字，整份文件至少两行命中；行尾那个没有文字的戳当整句终点，`[t][t]词` 这类空字跳过。首字戳前空白允许，未定时文字保文本但整行降级。
- TTML 支持显式绝对 begin/end/dur、简单 namespace、短/长 clock、ms/h/m/s、frameRate/frameRateMultiplier 和声明的 tickRate；默认帧率 30，未声明 tickRate 不猜 tick 等于秒。这是导出型歌词子集，不是完整 TTML 时间继承引擎；容器继承时间/seq 不会被误认成绝对逐字。
- TTML 字必须有有效显式 begin。缺字起点、未定时标点、嵌套或坏结构时保留完整文本和空白，整行降级。QRC 未定时尾巴同样降级，不再用上一字结束时间伪造尾字起点。
- 字时间倒流、非有限值、多字全同起点等不可信时间轴整行降级。真实字起点但缺 end 时，中字用下一字起点、末字用行末估算收尾；先按行时间排序再补 end。
- 混排文件保留普通 LRC 行和多个行头，同句普通副本去重；空白归前字 spaceAfter。`src/services/lyrics.js` 的 parseLrc 先试逐字，offset 同时平移行/字；普通 LRC 行为不变。

**渲染与时钟**：

- Weex `<text>` 不能嵌套，逐字使用多个并列 text + flex-wrap（一行里只有一个 `<text>` 出现在每个字符位上）；字号、行高和颜色明确落到字节点。当前行的每个词会带上 `cells`：把**这个词自己的**时间片按字宽比例切成字符单元，于是高亮能在词内部连续推进；邻行不拆，保持整词一个节点。单行超过 `MAX_PIECES = 160` 整行兜底，切分后超过 `MAX_CELLS = 200` 撤掉 `cells`；窗口仍有上限，不整首同时拆字。
- 普通 LRC 插值：`LRC_MAX_UNIT_SECONDS = 0.7`（一个字最多摊多久）、`LRC_FALLBACK_UNIT_SECONDS = 0.35`、`LRC_FALLBACK_MAX_SECONDS = 6`（没有下一句时按字数兜底）、`KARAOKE_LRC_ROWS = 3`（更远的插值行不拆字）；句尾标点黏在前一个字上，空白用外边距画出来。
- 三态配色：未唱 `#6d6d78`、正在唱 `#4fe0c4`、已唱 `#ffffff`；非当前行减弱。起点即点亮，不动态加粗以避免拉丁词宽度变化。正在唱的字在三态之上再加一层词内进度插值（见下），未唱/已唱的字仍是纯色。
- 只有 enabled && running 的真实当前行开钟；rAF 优先，无此能力时 33ms 定时兜底。时钟按 rate 前进，换倍速重锚；大幅位置变化超过 0.35 秒吸附，小校正不让刚亮的字反复熄灭。暂停/关闭/销毁撤帧，暂停 seek 按真实位置上色；原生位置停止更新时最多外推 1000ms。
- position watcher 接入 500ms 丢帧恢复，rAF/timeout 区分取消，旧回调 token 守卫避免双链。恢复仍需要新 position 或状态事件触发，完全没有原生更新时不会凭空恢复。
- 行位移保留 translateY + 300ms 过渡，不改 rAF 弹簧；原来的拖动冻结、回位定时与布局估算保留。

**类 Apple Music 的逐字动画（本轮重做）**：上一版只做到了「正在唱的那个字整体从青绿化开成白」，而且**起唱那一帧颜色从 `#6d6d78` 直跳 `#4fe0c4`（绿通道一帧 +115/255）**——整字均匀淡出不像扫光，那一跳就是肉眼看到的「闪」。现在两层一起做：**词内逐单元连续推进** ＋ **每个单元走两段首尾相接的连续曲线**，合起来就是一道带硬边界、在词内匀速右移的填充光带。

- **词内扫光 = 词盒 + 并列字符**：KaraokeLine.vue 的模板是 `.line-words`（`flex-wrap: wrap`）→ `.word-box` → 每字一个 `<text class="line-word">`；模板里 `<text>` 恰好只出现在字符位上（Weex 的 `<text>` 不能嵌套、没有 clip/mask，「一行里只有某几个字母变色」只能靠并列节点逐节点上色）。词盒是折行的**原子**（`flexShrink: 0` + `flexGrow: 0`），所以 `love` 拆成 l-o-v-e 也不会被折行断成两个词。
- **时间片来自这个词自己的区间**：LyricPanel.vue 的 `splitChars(text)`（按码点拆——`ui/src` 里没有 `Array.from`，代理对不能劈成半个字）、`splitCells(start, end, chars)` 与 `toPieces(units, letters)`。切分按排版字宽表 `charWidth(code, 100)` 的比例走，所以扫过的速度与字形宽度成比例；**不跨词、不插值、不凭空造时间轴**——数据只给词级时间时，词内这些字母共用词自己的 `[start, end]`，而不是整词一起亮。`letters` 只有正在唱的那一行才为 true（`const letters = distance === 0`），邻行保持整词一个节点。
- **上限与兜底**：节点数超过 `MAX_PIECES = 160` 整行退回整行文本；词内切分后总单元数超过 `MAX_CELLS = 200` 就整体撤掉 `cells`（仍按词上色）。
- **仍然是「读 currentTime 的渲染循环」**：位置 4Hz 推送，rAF 只在其间外推、并在新位置到达时重新锚定；时间轴有序时用二分 `findCellCursor(cells, position)` 找游标（O(log n)），`paint()` 只在跨越单元边界时重建 `states`——`const key = cursor * 2 + (singing ? 0 : 1)` 配 `if (this.states.length !== n || this._stateKey !== key)`；时间轴倒挂/重叠时自动退回逐单元比较。
- **曲线与配色**：三个模块级纯函数 + 一个分段函数都在 KaraokeLine.vue：`pieceProgress(cell, position)`（`duration <= 0` 返回 0，其余钳制在 `[0,1]`）、`interpolateColor(fromHex, toHex, t)`（整数色道线性插值、`t` 钳制，返回 `#rrggbb`）、`easeInOut(t) = t<0.5 ? 2t² : 1-2(1-t)²`、`wordColor(cell, position, offColor, doneColor)`：前 `WORD_GLOW_RISE = 0.35` 从暗色升到满青绿（唱到），之后 65% 化开成已唱色（唱过去），两端斜率为 0。曲线两端分别**恰好等于**未唱色与已唱色：当前行 `#6d6d78` / `#ffffff`，邻行 `#8a8a93` / `#e8e8ee`——顺手修掉了邻行「曲线终点到白、状态却切成 `#e8e8ee`」的隐性跳变。
- **响应式只写「有没有正在唱的单元」**：`paint()` 里只有 `if (singing) this.now = position`；`cellStyle(cell, index)` 读 `const now = this.now` 并交给 `wordColor`。于是每帧只重渲染这一行，其它行、父面板和 translateY 都不动；一个字都不在唱时**一次响应式写都没有**，空转重渲染为零。
- `states` 仍是 0/1/2 三态、仍只在单元边界发布，所以「同一单元区间内连数组引用都不变」的抗闪烁优化和既有断言全部保持不变；动画只是在这一层之上多算一个颜色。
- **暂停 = 精确定格**：`running` 变 false 时先撤帧，再 `this.paint(seconds(this.position))` 按最后一次真实位置重算一次颜色，画面停在「暂停那一刻」的曲线值上（例如 `I never re████ally knew you`），不会自己唱完当前词、不会回退，也不等 CSS 过渡播完。
- **seek / 换行 = 位置说了算**：位置跳变与本地时钟相差超过 `KARAOKE_SNAP_SECONDS = 0.35` 时直接吸附并立即重画——向后 seek 会把已经亮的字重新变暗（用例 19）；连续快速 seek（2.5 → 0.6 → 1.4 → 0.1）每次都只按最新位置重算，不堆积第二条帧链（用例 20）；`pieces` 内容签名变化时才 `this._stateKey = null` 重建，换行/换歌不漏旧状态（用例 21）。
- **点击歌词行 seek**：`.line` 上加了 `@click="onLineTap(row)"`，发 `seek-line { index, time }`；手势划过（`_moved`）、拖动中、以及「点的就是当前行」都不触发——点正在唱的那一行不该把歌倒回句首。player.vue 的 `onSeekLine` 只调 `player.seek(time)` 再 toast，不自己维护「选中了哪一句」，行高亮仍由播放器时间算出来。
- **构建坑（已由守卫锁住）**：打包管线会删掉未使用的局部变量。第一版用 `const beat = this.beat` 只为「建立渲染依赖」，编译后整行被删掉，渲染根本不依赖它——源码里能跑、进包后静默失效。现在这个响应式字段是被样式函数真的读走的值，`.tools_tmp/verify-pack.js` 直接在**打包产物**上检查 `if (singing) this.now = position`、`const now = this.now`、`wordColor(cell, now, offColor, doneColor)` 与两段 `interpolateColor(..., easeInOut(...))` 同时存在（第十五节 112 条守卫，其中 17 条整条锁住这条动画链）。
- 平滑度是**量化**过的（`.tools_tmp/test-karaoke-line.js` 用例 14/18，17ms 一帧 ≈59fps 连续采样整行）：0.5 秒的单元**逐帧单通道增量峰值 21/255**，旧实现的同一指标是 **115/255**（就发生在换字那一帧）；QRC 常见的 0.15 秒短单元实测峰值 **62/255**，物理上界 `ceil(2×17/(0.35×150)×115)+2 = 77`（曲线两端斜率为 0，字越短光带越快，不可能更平滑）；同时断言亮度全程不倒退、一次只前进一个单元。0.5x/2x 由同一个 rate 时钟驱动。
- **一张图看懂差别**：`node .tools_tmp/render-sweep-png.js` 会跑上面这套真实 `paint()` + `cellStyle()`，把 18 帧颜色画成胶片条 `.tools_tmp/sweep-frames.png`（上带＝新版逐字母扫光，下带＝旧版整词变色；白色细线是正在唱的那一个单元的插值边界）。不需要浏览器或真机，也不用装依赖：PNG 由脚本自己按 zlib 手写（只有 Node 内置模块）。
- 不想装机也能看：`node .tools_tmp/preview-karaoke.js` 生成 `.tools_tmp/preview-karaoke.html`（74,656 字节）——它把 KaraokeLine.vue 与 LyricPanel.vue 的**真实脚本**都抠出来，用真实的 `buildPieces`/`groups`/`cellStyle` 渲染成 DOM 的 `.word-box > span`，跑**真实的时钟链**（Date.now + requestAnimationFrame），并按真机节奏喂位置（位置差超过 0.2 秒才推一次，帧间靠组件自己的外推）。可以按 0.25x/0.1x 慢放、在「新版逐单元扫光 ↔ 旧版整词变色」之间切换对比，页面上直接读出时间、逐帧峰值、词内扫光进度（`词（已扫 n/总）`）与最短单元时长。无头冒烟 `.tools_tmp/_smoke-preview.js`：新版峰值 **53/255、0 次跳变**（上界 82；最短单元是 `you're` 里的撇号 140ms），旧版 **116/255、20 次跳变**。它是颜色/时钟层面的仿真，**不是 Weex 渲染**，真机排版仍需装机确认。
- **英文也逐字母扫光**：当前行的拉丁词按字母拆开、共用这个词自己的时间片（词盒保证不被折行拆开），于是中英文都是「词/字内连续推进」；邻行仍是整词一个节点，只有未唱/已唱两种颜色。

**本地文件**：扫描/读取支持同名 `.lrc/.yrc/.qrc/.ttml/.krc` 文本及常见大小写。已知歌词路径优先；无路径时依序探测，`.lrc` 在前。普通 `.lrc` 没有真实字时间轴，开关打开时按上下句时间插值（见上）；方括号内联的 `.lrc` 本身就是真实逐字数据。

**回归验证**：20 个脚本全部通过（`node .tools_tmp/<名字>.js`，退出码 1 = 有用例不通过）。14 组断言套件合计 **1564 条**（各套件自报数直接相加：adapter 69、core 211、hash 65、index-musicfree 53、index-page 41、karaoke-line 126、karaoke-settings 265、lyric-drag 120、media-keys 82、musicfree 185、player-media-keys 79、polyfills 48、video-probe 42、word-lyrics 178），再加折行/定位套件 72 项与下载链 29 项；包校验 112 条守卫全部通过。其中字时间轴解析（`test-word-lyrics.js`）178（含 31 条方括号内联用例，以及 §20.20 的 27 条双语配对用例）、面板与拖动（`test-lyric-drag.js`）**120**、单行时钟与逐字动画（`test-karaoke-line.js`）**126**（本轮 82 → 126：新增词内游标二分与线性扫描逐点一致、词内逐字母推进、重叠时间轴退回逐单元路、暂停精确定格与短单元速度上界、向后 seek、连续快速 seek、换行/换歌不残留）、设置与异步资源（`test-karaoke-settings.js`）38 个场景/265 条断言。覆盖默认关闭、自动/手动保存、同单元多帧零响应写、倍速、丢帧/旧回调、复制 props、晚到数据、写缓存失败、首次本地 YRC 发现、暂停 seek、销毁与异步世代竞争，以及本轮的起唱帧不跳变成青绿、字内亮度单调不倒退、两段曲线两个端点取值、邻行接色逐通道差 ≤2、逐帧单通道增量上界，以及点击行 seek（手势划过不算点击、点当前行不倒回句首、行时间非有限不发 seek）和父页面只让播放器时间说了算（不出现 `selectedLyricTime`）。测试运行真实脚本但 Vue watcher 为手动调用，没有设备原生渲染桥，不能据此宣称真机零闪。引用、类名、布局和样式检查通过；包校验第十五节现在锁的是单元模型（`findCellCursor` 二分游标、`_stateKey` 增量写、`splitChars`/`splitCells`/`toPieces` 词内切分、`MAX_CELLS = 200`、暂停时 `paint(seconds(this.position))`、`onLineTap` 与 `'seek-line'`）以及连续曲线四件套 `wordColor` / `easeInOut` / `WORD_GLOW_RISE = 0.35` / 两段 `interpolateColor(..., easeInOut(...))`。

在仓库根目录运行：

```bash
node .tools_tmp/test-word-lyrics.js
node .tools_tmp/test-lyric-drag.js
node .tools_tmp/test-karaoke-line.js
node .tools_tmp/test-karaoke-settings.js
node tools/build-ui.js --pack
node .tools_tmp/verify-pack.js
node .tools_tmp/verify-download.js
```

构建退出 0，保留主题未安装、历史 SongRow border、custom/http 原生模块及 built-in http 的已知警告。本轮是 JS/UI 改动，两架构原生模块未重编，包中模块与当前 libs 逐字节一致。最新安装包 PenMusic 1.1.6（`8001749598192572.1_1_6.amr`）：**989,587 字节、26 条目**。

```text
SHA256: aaf1ffd013f4786b538f9e0e73415eacc26fb86420cb3d9091a57124cdf8d535
```

**装机验收**：覆盖安装后完全退出再启动；先确认默认关时普通歌词稳定，再开启并保存，用有真实字时间戳的歌词检查跨字/跨行、长句折行、拖住跨行、暂停 seek、0.5x/2x 和后台恢复。再拿两份歌词对比：一份方括号内联的（例如 LDDC 导出）应该**一句只出现一次且逐字亮**，一份普通 `.lrc` 开关打开后整句也要逐字亮（按上下句时间插值，属于近似）。再盯住正在唱的那个字/字母：它应当**从行内暗色连续升到青绿、再化开成白**，既不能在起唱那一帧「啪」地变成青绿（旧实现就是这里一帧跳 115/255），也不能到点一下跳白；英文单词（如 `love`）也应当**在词内部自左向右连续扫过**，而不是整个词一起变色，并且折行时单词不会被拆到两行。连着看几个字，整行的填充边界应当像一道光带连续右移，换字时看不到闪。暂停时该字颜色定格、不再变化，恢复播放后从同一进度继续。点任意非当前行应当跳转到那句并居中，点当前行不应倒回句首，拖动划过之后松手不应触发跳转；拖动进度条快速来回（10% → 90% → 30% → 80%）时，歌词必须每次都直接落在最新位置，不能看到旧位置的高亮被「追」过去。想先看效果再装机，可以先在电脑上打开 `.tools_tmp/preview-karaoke.html`（用 0.1x 慢放最容易看出差别，并可与「旧版实现」对比）。包里没有内置逐字样本；真实音源/Weex 排版和机身性能尚未在设备验证。

#### 20.19 官网（仓库根 `114.html`）与关于页的官网地址 / 版本 1.1.7（→ 1.1.8，见 §20.21）

**需求**：把仓库根的 `114.html` 改成 PenMusic 官网，**主题色与动效不变**；截图放 `screenshot/`；关于页加官网地址 `studio.furina1314.top`；版本升 1.1.7。随后追加一条：**软件免费，官网不许有付费相关内容**。

**一、站点做了什么**

`114.html` 原样保留的部分（这就是「主题色/动效不变」的字面意思，`check-site.js` 逐条守着）：`:root` 与 `@media (prefers-color-scheme: dark)` 里的**每一行**颜色变量（`--pink #fb7299`、`--pink-d #d92f68`、`--pink-l #ff9db9`、`--blue #23ade5`、`--violet #8b7bff` …）、极光三光斑的 rgba、指针跟随光晕；9 条 `@keyframes`（`drift1/2/3`、`floatLogo`、`titleIn`、`rippleGo`、`cue`、`marquee`、`spin`）与 9 条 animation 简写、钉住滚动的行程（`.story` 420vh、`.manifesto` 300vh、`.story-stage{position:sticky;top:0;height:100vh}`）、`[data-reveal]` 的 .85s、`prefers-reduced-motion` 降级；以及 JS 十块（指针极光 rAF 合并、`header.scrolled`、`pointerdown` 波纹、表单 loading 15s 复位 + `pageshow` 复位、IntersectionObserver 揭示、故事钉住滚动、`.ch` 逐字点亮、hero 视差、统一 rAF 滚动、`.step, .purchase-form` 指针倾斜）。

换掉的是**内容**：品牌与图标（`screenshot/app-icon.png`）、hero 标题、4 张幻灯片（离线 / 插件 / 逐字 / 蓝牙）、`#manifesto` 逐字文案、两条跑马灯、三步上手、下载区、页脚。另外**新增截图区** `section.steps.shots#shots`：8 张 `li.step.shot[data-reveal]` 复用 `.step` 的揭示与倾斜动画，**没有新增任何 transition/animation**（新增 CSS 只有 `.shots*` 的网格与图片圆角）。

hero 标题从 4 个字母变成 `PenMusic` 的 8 个 `<span>`：原 `nth-child(1..4)` 的延迟 `.05/.13/.21/.29s` 一个没动，只**追加** 5/6/7/8 = `.37/.45/.53/.61s` —— 仍是同一个 `titleIn`，级联节奏一致。

JS 相对原件只有两处改动（其余逐字未动）：① `submit` 里补 `e.preventDefault();`（本站没有后端，不做真提交）；② 之后按 `form.dataset.download` 造一个 `<a download>` 点一下，把「下载安装包」按钮接上真实的包（`data-download="ui/8001749598192572.1_1_8.amr"`，升 1.1.8 时同步改过）。HTML 注释里写了：包放在站点根目录时把它改成裸文件名。

**二、免费：写了什么，也守住了什么**

hero 副标题、CTA（「免费下载 · v1.1.7」）、下载区 eyebrow（「完全免费」）与标题、下载卡说明、页脚、跑马灯都写明免费；下载卡里那句是「没有内购、没有会员、没有订阅，装完即用，以后升级也不收费」。旧站的付费文案（购买 / 授权 / 一机一授权 / 登记订单 / 设备 S/N / 发票 / 买断制…）全部清除。

**三、截图与图标**

`screenshot/` 里原先只有 8 个指向桌面的 `.lnk` 快捷方式。本轮把 8 张真图按原名复制进来（800 × 254，设备是 640 × 260 设计稿），并新增 `screenshot/app-icon.png`（= `ui/src/assets/app_icon.png`，179 × 179，favicon 与 hero 图标都用它）。站点按**相对路径**引用（`screenshot/xxx.png`）：把站点与 `screenshot/` 放同一层即可，本地双击也能看。

**四、关于页**

`ui/src/pages/about/about.vue` 新增模块级常量 `SITE_URL = 'studio.furina1314.top'`，并在 `rows()` 的「版本」之后插入 `{ label: '官方网站', value: SITE_URL }`；`versionText()` 的兜底值 `'1.1.6'` → `'1.1.7'`（那行注释本就要求它跟 `ui/package.json` 一致；§20.21 又改成 `'1.1.8'`）。**刻意不做成可点链接**：词典笔上没有可用的浏览器入口，点了也打不开，所以跟其他信息行一样是纯文本。

**五、版本 1.1.7**

`ui/package.json` 的 `version` → `1.1.7`。产物名由构建按 `appid + version` 现算，所以自动变成 `ui/8001749598192572.1_1_7.amr`（**不要手改包里的 manifest**，见 §20.15）；`verify-pack.js` 自己按 package.json 推包名，不需要改脚本。版本号随后又升到 **1.1.8**（见 §20.21），产物名同理由构建现算成 `ui/8001749598192572.1_1_8.amr`。

**六、新增守卫 `check-site.js`（139 项）**

`.tools_tmp/check-site.js` 不渲染、只做结构性对比，六节：① 亮/暗主题变量逐行比对 + 极光三色；② 9 条 keyframes + 9 条 animation 简写 + 关键帧体 + 行程高度 + 减弱动效分支 + 各段 JS 钩子；③ 脚本选的选择器 ↔ 页面元素（数量要对得上）；④ 所有 `screenshot/...` 引用在磁盘存在、8 张真机截图都被用上；⑤ 文案（旧站词清干净、付费词为零、`免费` ≥ 4 处、官网地址与版本号在位）；⑥ 标签配对、样式花括号、**脚本能被 `new Function` 解析**（语法错误会让整页动效直接失效）。

三条踩过的坑值得记：

- 数类名**不能**拿 `\bstory\b` 去 grep：`story-stage` / `story-dots` 里也有 `story`，连字符处词边界成立，会把它们一起数进来（第一版守卫就这么误报两条）。改成把 `class="..."` 拆成 token 再精确比对。
- 「不许出现付费词」的关键词守卫会把**澄清句**判红：「没有内购、没有会员、没有订阅，也不收费」里每个词都在。所以逐个命中要回看前一个词，前面是 `没有/不/无/无需/绝不` 才放行。
- 拿 `Pro` 当违禁词会命中 `style.setProperty` —— 词表要挑不会出现在代码里的写法。

**七、验证**

`node .tools_tmp/check-site.js` 140 项全过；回归 20 个脚本全绿（14 组断言 1564 条 + 折行 72 + 下载链 29 + 包校验 112 条守卫）。构建退出 0，只有四个已知警告（falcon-ui 主题未安装、SongRow 的 `border`、`custom`/`http` 原生模块、built-in `http` 偏好）。本轮只改 JS/UI 与官网，两架构原生模块未重编，包内模块与当前 `libs/` 逐字节一致。最新安装包 PenMusic 1.1.8（`8001749598192572.1_1_8.amr`）：**991,032 字节、26 条目**（1.1.7 那份已被它取代：先修 §20.20 的翻译错位重打、再升版本号重打，下面这组数字是**最终 1.1.8** 那份）。

```text
SHA256: 64246eb64117092451e41337968a82b780a9a550994107c7fe348d7c068ef616
```

包内 `manifest.json`：`version` = `1.1.8`、`appName` = `PenMusic`；`about.js` 里能看到 `studio.furina1314.top` 与 `官方网站`。

**装机验收**：覆盖安装后完全退出再启动 → 关于页「官方网站」应显示 `studio.furina1314.top`，右上角版本应显示 `v1.1.8`。官网在浏览器里打开 `114.html`：极光、指针光晕、hero 字母级联、故事钉住滚动、逐字点亮、跑马灯、截图区揭示与倾斜都应与原来一致；点「下载 PenMusic 1.1.8」应当直接下载安装包（站点里包的位置不同时改 `data-download`）。再验一条 §20.20：双语 `.lrc` 的当前行高亮要落在**原文**上、译文在它下面一行灰字、句间不再出现重复的译文行。**真机渲染与站点部署均未验证**（无设备、无部署环境）。

#### 20.20 逐字歌词与翻译「错位」：双语增强型 LRC 的译文没配对（真机 bug）

**症状**：播放 LDDC 导出的双语 `.lrc`（《See You Again》）时，当前行的**逐字高亮落在译文上**，原文那一行反而灰着，译文句还重复出现一行。

**文件长什么样**（关键）：LDDC 的「增强型 LRC」把每句原文写成内联时间戳（A2），译文行紧随其后、**开始时间与原文完全相同**，但整句只有开头一个标签和末尾一个收尾标签：

```text
[00:23.572]We've [00:23.788]come [00:23.978]a [00:24.203]long [00:25.250]way [00:26.402][00:26.972]from [00:27.165]where [00:27.450]we [00:27.810]began[00:29.026]
[00:23.572]回头凝望 我们携手走过漫长的旅程[00:29.337]
```

**根因两条，缺一不可**：

1. `services/lyrics.js` 的 `parseLrc` **逐字分支只排序、不配对翻译** ——「同一时间戳第二条当翻译」这套逻辑当时只写在普通 LRC 分支里。译文于是成了独立一行；它与原文开始时间一样，而 `findLyricIndex` 取的是「最后一个 time ≤ 当前时间」的行 → 当前行落到译文上。
2. `services/wordLyrics.js` 的 `parsePlainLrcLines` 把**每一个** `[mm:ss]` 都当成一句的开头，而译文行是「前面有字、末尾那个标签后面没字」→ 一行译文被拆成两行（其中一行比原文晚 1ms 落地，进一步保证「当前行 = 译文」）。

**改动**：

- `services/lyrics.js` 新增 `SAME_TIME = 0.05`、`CJK_CHAR = /[\u3400-\u9fff\uf900-\ufaff]/` 与 `mergeSameTime(lines)`：同一时间戳的第二条当翻译；**译文在前**（第二条才带逐字）则交换；第二条整句只有一个「字」（只有首尾两个标签的译文行）也算译文 —— 这条判定跟语种无关，所以「英文原文 + 非中文译文」一样配对；两条都带逐字、同属一种文字且都不是单字段（对唱、重叠句）则各自成行；中外对照（CJK 判定不同）则合并。逐字分支与普通分支现在都要过这一道。
- `services/wordLyrics.js` 的 `parsePlainLrcLines` 把「这个标签后面没字、前面有字」的标签当成**这一行的终点**（写进 `endTime`），不再新开一行。

**为什么插件歌词也跟着好了**：`core/lrcParser.js` 全部转发 `parseLrc`，`hasTranslation` 就是看行上有没有 `trans` —— 译文配对上了，插件那条链自然不用再猜翻译。

**验证**：`.tools_tmp/probe-a2-pairs.js`（加载真解析器，打印指定秒数附近的行与当前行）在修复前把这份文件解析成 **205 行**、28 秒时当前行是译文（`#13`）；修复后 **85 行**、当前行是原文 `We've come a long way from where we began`（9 个词、endTime 29.026），`trans` 是 `回头凝望 我们携手走过漫长的旅程`。`test-word-lyrics.js` 新增第十二节 27 条断言（双语配对、真实那一行的当前行、收尾标签不当新句、译文在前要交换、对唱不合并、中外对照合并、非中文译文也配对、普通 LRC 行为不变），**178 条全过**；15 个套件重跑全绿。包已重打两次（1.1.7 同版本号一次，随后升到 1.1.8 —— 体积/SHA256 见 §20.19 第七节与 §20.21）。

**教训**：把「谁是当前行」交给「最后一个 time ≤ t」时，**任何多出来的同时间戳行都会抢走高亮**，解析阶段必须保证「一个时间点一行」；同一份文件里混着两种格式（内联行 + 只有首尾标签的普通行）是常态，不能假设整份文件格式统一。

#### 20.21 版本 1.1.8：修完翻译错位之后升版本号

修 §20.20 的翻译错位时**没有**动版本号（同一个 `1.1.7` 重打了两次：一次加进修复代码，一次收敛「单字段译文」判定），于是同一个版本号对应了三个不同内容的包。用户随后要求用它区分构建，就把版本号整体升到 `1.1.8` —— **只有版本字符串在动，代码与样式一行没改**：

- `ui/package.json` 的 `version` → `1.1.8`；
- `ui/src/pages/about/about.vue` 的 `versionText()` 兜底值 `'1.1.7'` → `'1.1.8'`（它必须跟 package.json 一致，那行注释本来就写着）；
- `114.html` 六处：导航条（`词典笔离线音乐播放器 · v1.1.8`）、hero 的 CTA「免费下载 · v1.1.8」、跑马灯第二行（两个 `<span>`）、下载卡的 `data-download="ui/8001749598192572.1_1_8.amr"`、下载卡正文「当前版本 **1.1.8**」、提交按钮「下载 PenMusic 1.1.8」；
- 根 `README.md` §八里的产物名。

**顺带把守卫改成不会再过期**：`check-site.js` 以前把 `1.1.7` 写死在三处断言里（跑马灯文案、`data-download`、版本号出现次数），一升版本就得改脚本。现在它先读 `ui/package.json`，用 `pkg.version` 现算 `VER`，再算 `AMR = appid + '.' + version.split('.').join('_') + '.amr'`，然后断言页面里 `VER` ≥ 3 次、`data-download` 指到 `ui/<AMR>`，并**顺带检查那个 `.amr` 真在磁盘上**（还没构建时会明确提示「先跑 tools/build-ui.js --pack」）。断言数 139 → 140。

产物：`ui/8001749598192572.1_1_8.amr`，**991,032 字节、26 条目**（与 1.1.7 那次的字节数完全相同 —— 版本字符串等长，包里没有别的东西变），SHA256 `64246eb64117092451e41337968a82b780a9a550994107c7fe348d7c068ef616`；包内 `manifest.json` 的 `version` = `1.1.8`。verify-pack / verify-download（29）/ check-site（140）/ check-syntax / check-refs 全过，15 个测试套件全绿。

---

## 四、构建

### 本地快速构建（Windows / Linux / macOS 通用）

```bash
# 1. 安装构建器依赖（只需一次）
cd aiot-vue-cli
npm install --legacy-peer-deps --ignore-scripts
npm install --no-save typescript@5.6.3        # @rollup/plugin-typescript 的 peer 依赖

# 2. 构建 UI
cd ..
node tools/build-ui.js            # 仅打包页面到 ui/.falcon_
node tools/build-ui.js --minify   # 压缩输出
node tools/build-ui.js --pack     # 额外生成 manifest.json 与 .amr
```

产物：`ui/8001749598192572.1_1_3.amr`

辅助脚本：

```bash
node tools/check-classes.js        # 类名体检：页面之间不许重名（谁最后注册谁生效）
node tools/check-layout.js         # 布局耦合体检：样式尺寸 vs 各处 JS 常量
node .tools_tmp/test-lyric-drag.js # 歌词拖动/停留状态机的单元测试
node .tools_tmp/test-adapter.js    # 网络阶梯 / axios / storage / require 白名单
node .tools_tmp/test-core.js       # 音源插件运行时：沙箱/取址回退/管理器/播放队列
node .tools_tmp/test-musicfree.js  # MusicFree 模式：模式门控/取址两条路/播放交接
node tools/check-styler.js         # 校验 falcon-styler 是否正常工作（探针）
node tools/make-icon.js            # 重新生成 256x256 应用图标
```

**为什么还需要那个探针**：`adapter/` `core/` `vendor/` 这些文件目前没有被任何页面 import，所以正常构建**根本不会解析它们** —— `node tools/build-ui.js` 成功并不说明它们能进包。探针（`.tools_tmp/_probe-bundle.js`）把它们全量 import 进一个临时入口，用**同一份 rollup 配置**打一遍。实测结果：构建通过；`await import(/* @vite-ignore */ 'custom')` 原样保留（与 `services/platform.js` 现在的行为一致，构建只给一条「未找到以下模块」的警告）；`await import('../services/player.js')` 被拆成一个独立 chunk（现有播放器里 `await import('./library.js')` 一直就是这么跑的）。探针产出的 `_probe_out/` 只是用来看的，验证完可以直接删掉。

### 构建私有 jsapi 原生模块（**做成本地播放器必须要这一步**）

部分机型（如 X3s）既没有系统模块 `fs`，也没有可用的 shell，纯 JS 拿不到本地文件列表，必须由本应用自带一个 `.so`。

```bash
# 1. 下载工具链 + versionInfo(SDK)
node tools/fetch-toolchain.js aarch64     # 或 armv7 / armv7-uclibc

# 2. 解压（Linux / WSL 里执行，必须保留符号链接）
tar -xf jsapi/versionInfo_p5.tar.gz -C jsapi
tar -xjf jsapi/toolchains/aarch64--glibc--stable-2018.11-1.tar.bz2 -C jsapi/toolchains

# 3. 交叉编译 -> ui/libs/libjsapi_langningchen.so
bash tools/build-native.sh
```

编译完成后重新执行 `node tools/build-ui.js --pack`，`ui/libs/` 会被 `aiot-vue-cli` 自动同步并打进 `.amr`。

> **架构必须与机型匹配。** A6P / X5 / S6P 是 armv7，P5 及较新的笔是 aarch64。装错架构的表现是：`libjsapi_*.so` 无法 dlopen，诊断页的「自研模块 FileShell」显示「未加载」。查机型架构最直接的方式：`adb shell uname -m`（`aarch64` 或 `armv7l`）。

### 完整构建（含字节码，需交叉工具链）

```bash
./tools/build.sh -a          # 模板自带脚本，会编译全部原生模块
```

GitHub Actions（`.github/workflows/build.yml`）会在 push 到 `main` 时为四个机型自动构建并上传 `.amr` 产物。

> **注意：不要用 PowerShell 直接改写源码/文档文件**（`Get-Content | Set-Content` 会按系统 ANSI 解码 UTF-8，中文全毁且不可逆 —— 本文档头部的事故就是这么来的）。

---

## 五、安装到设备

1. 把 `.amr` 拷到设备（或通过 CI 下载）；
2. 用设备自带的 miniapp 安装入口安装；
3. 首次启动进入**诊断页**，确认：
   - `屏幕分辨率` 是否与预期一致（决定布局缩放）；
   - **`自研模块 FileShell`** 是否显示版本与架构（显示「未加载」= `.so` 没生效，多半是架构不匹配，见上一节）；
   - `文件访问来源` 用的是哪一级（自带模块 / 系统 fs / shell 兜底）；
   - `文件元信息` 是否可读（决定时间/体积排序是否精确）；
   - `音频引擎`、`倍速支持`、`进度拖动`、`系统键盘` 的状态。

---

## 六、已知限制

1. **本地文件扫描有三级回退，取决于固件给了什么。**

   | 优先级 | 来源 | 能力 |
   |---|---|---|
   | 1 | **自带私有 jsapi `custom.FileShell`**（`ui/libs/libjsapi_langningchen.so`） | 完整：列目录 + 读文件 + 文件元信息 |
   | 2 | 系统模块 `fs`（依次尝试 `fs` / `file` / `fileSystem` / `io`） | 完整，但依赖固件版本 |
   | 3 | `global.execShell` 兜底 | 可列目录、可读文本；**没有文件体积 / 修改时间** |
   | 4 | 都没有 | 只能用缓存曲库，首页与诊断页明确提示 |

   实测部分设备（如 X3s）**第 2、3 级都不可用** —— 社区同类应用在这些机型上会直接黑屏。本应用不会黑屏：优先用自带模块，再逐级降级，并把当前用的是哪一级显示在诊断页的「文件访问来源」一行。

   > 自带模块的架构必须匹配机型：A6P/X5/S6P 是 armv7，P5 及较新机型是 aarch64。不匹配时 `.so` 无法 dlopen，会自动落到下一级。

2. **倍速由 `custom.AudioBridge` 提供**（ffmpeg `atempo` 滤镜），0.5x~2.0x。若降级到 `$falcon.soundPlayer`，那边没有调速接口，界面会提示不支持但偏好仍会记忆。诊断页的「倍速支持」一项显示当前实际走的是哪条路。

3. **进度拖动**：`custom.AudioBridge` 通过带 `-ss` 重启 ffmpeg 实现；降级到 `soundPlayer` 时用其 `seek(秒)`。诊断页会显示真实结果。

4. **文件体积 / 修改时间取决于 `fs.stat` 是否存在。** 官方文档只承诺 `readFile / writeFile / readdir / mkdir / exists`。若没有 `stat`，体积显示为 `--`，时间排序自动退回「入库时间」（即首次扫描到该文件的时刻，记录在 `mlp_added_v1`）。

5. **时长信息**在首次播放该曲目后才会有（没有原生元数据解析接口），之后会缓存到存储，供「按时长排序」使用。

6. 歌词 / 封面均为**同名文件**约定（`歌曲.mp3` ↔ `歌曲.lrc` / `歌曲.jpg`），不支持内嵌标签读取。本地图片需要 `file://` 前缀才能被 `<image>` 渲染，已由 `toLocalUrl()` 处理。

7. **视频播放（实验室）**：扫列表、交给系统播放器都不难，难点是**这台设备的 miniapp 渲染器没有实现 `<video>`** —— 画面必须靠 `gst-play-1.0` 之类的系统播放器往 Wayland/KMS 上送。所以视频是「能放，但走的是另一条路」，不是网页里那种内嵌播放。

---

## 七、验证情况

**前端（已验证）**

- `node tools/build-ui.js --pack` 退出码 0，**零样式校验错误**（仅剩与业务无关的 falcon-ui 主题提示——CI 安装 falcon-ui 后即消失）；
- 产物 `ui/8001749598192572.1_1_3.amr`（246 KB，12 个条目）已生成并通过内容核对：`app.js` / `app_icon.png` / `manifest.json` / 4 个页面包 / 3 个共享 chunk / `libs/libjsapi_langningchen.so`；
- 已核对产物中：无任何 `sixi` 残留、`import('fs')` 与 `import('global')` 为运行时动态导入、播放走 `$falcon.soundPlayer`、进度用 `<seekbar>`、输入法回传字段为 `contents` / `editCanceled`；
- `tools/check-styler.js` 复现了历史报错（后代选择器、border-radius 四值、`display:block`），证明样式校验链路真实生效。

**原生模块（已用真交叉工具链编译并校验）**

用 Buildroot `aarch64--glibc--stable-2018.11-1`（g++ 7.3.0）在 WSL 中实际编译通过：

```
ui/libs/libjsapi_langningchen.so
  → ELF 64-bit LSB shared object, ARM aarch64, dynamically linked
  → 导出 custom_init_jsapis / createFileShell / JSFileShell::*
  → NEEDED 仅 libpthread / libstdc++ / libm / libgcc_s / libc（均为设备自带）
```

依赖是通过 `-Wl,-unresolved-symbols=ignore-all` 留给运行时解析的，与 PulseBox 的做法一致 —— 也就是说它不会引入额外的 so 依赖。

**尚未验证**：真机运行表现（扫描、播放、歌词显示、系统输入法实际弹出）。装到设备后请先看诊断页。

- 视频（实验室）：`node .tools_tmp/test-video-probe.js`（42 条）守着命令构造与路径转义；真机上「系统播放器」按钮的日志会直接显示在视频页上，成没成一目了然。
- MusicFree 模式：见「三、20.10」（离线测试 500+ 条断言，含原生 `.so` 的符号自检）。

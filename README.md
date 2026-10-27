# MetaOnticVideo Agent（视频导演）

一个 **DeepSeek Harness 智能体预设**：你把一段创意发给它，它先写视频脚本；**你确认脚本后，它才调用 `video_generate` 生成**。
它复用 [MetaOnticVideo](https://github.com/wxsimon2022/metaontic-video) 的 Seedance 生成能力，
自己不接上游、不存密钥，只负责「导演」——定规格、写脚本、分镜、逐镜提示词、调用生成、验收、合成、交付。

- 预设 id：`video-director`
- 显示名：**视频导演**
- 包名：`metaontic-video-agent`
- 形态：纯配置 bundle（无 JS、无新服务），安装后出现在新会话的智能体选择器里

工具名以 MetaOnticVideo 版本为准：**1.2+** 用 `video_generate` / `video_extend` / `video_status` / `video_config`；
**1.1 及以前**用 `seedance_video` / `seedance_extend` / `seedance_extend_task` / `seedance_extend_config`。
本包的 persona 和 skill 以新名字为主，并提示 Agent 先看实际工具列表。

## 脚本确认门槛（默认行为）

1. 你发一段创意（一句话或一段描述）。
2. Agent 写 `video-projects/<slug>/script.md`：一句话钩子、分镜表、完整台词、声音、结尾。
3. Agent 把脚本完整给你看，并问：**「确认这版脚本吗？确认后我才调用 video_generate。」**
4. 你可以直接改；改完 Agent 会重新展示、再次确认。
5. 只有你明确确认后，Agent 才写 `shots.json` / 逐镜提示词，并调用 `video_generate`。
6. 脚本确认前不会消耗生成额度；脚本改了，之前的确认作废。

「不错」「再改改」不算确认；要生成请明确说「确认」「可以」「开始生成」「按这个来」。

## 它能做什么

| 用户说 | Agent 做 |
| --- | --- |
| “外卖员雨天送餐，客户递伞，最后发现是差评客户” | 写脚本 → 你确认 → 15 秒一镜到底提示词 → `video_generate` → 验收 → 交付 mp4 + 制作包 |
| “这条 15 秒口播再加 5 秒，切到听众反应” | 先改脚本并确认 → 用 `video_extend` 续拍，或重新分镜成两镜再合并 |
| “做一个 30 秒的三镜短片” | 先给脚本确认 → 三个镜头分别生成 → ffmpeg 拼接 → 抽帧验收 |
| “把这条片改成 9:16 竖屏” | 先按竖屏改脚本确认；已有成片则用 ffmpeg 裁切/补边 |

产物固定落在 `video-projects/<slug>/`：

```text
brief.md            一句话创意、规格、受众、限制
script.md           分镜脚本 + 完整台词（draft / approved）
shots.json          镜头清单、任务 ID、生成状态
prompts/01-*.txt    每镜最终提交的提示词
clips/              Seedance 分段
final/              合成后的成片
production.md       任务 ID、耗时、验收记录
```

## 从零安装（含 MetaOnticVideo）

按顺序做 0–5 步。只装本 Agent 是不够的：**没有 MetaOnticVideo，就没有 `video_generate`，只能写脚本不能出片。**

### 0. 装 Harness 和 ffmpeg

| 要装什么 | 为什么 | 怎么装 |
| --- | --- | --- |
| DeepSeek Harness | 插件、工具、智能体都跑在它里面 | 装 Desktop 版（或 Web 版） |
| `ffmpeg` + `ffprobe` | 多镜合成、探测时长 | Mac：`brew install ffmpeg`；Windows：`winget install Gyan.FFmpeg`；Linux：`sudo apt install ffmpeg` |

验证：

```bash
node -v && ffmpeg -version | head -1 && ffprobe -version | head -1
```

三条都打印版本号就齐了。  
装了但不在 PATH：在 MetaOnticVideo 配置里把 `ffmpegBinary` / `ffprobeBinary` 填绝对路径（Mac 常见 `/opt/homebrew/bin/ffmpeg`）。  
完全没装 ffmpeg 也能出片，只是不会自动合并成一条完整视频。

### 1. 装 MetaOnticVideo

侧栏 **插件 → 添加插件 / 安装**，填下面任意一种 spec：

| 来源 | 填什么 |
| --- | --- |
| GitHub（推荐） | `github:wxsimon2022/metaontic-video` |
| npm | `metaontic-video` |
| 本地目录 | 本机 `metaontic-video` 目录的绝对路径 |

装好后确认 **MetaOnticVideo** 的开关是**打开**的。  
改过插件代码、或刚装完第一次，**重启一次 Harness** 再继续。

> 不需要 `npm install`：插件没有第三方依赖，`@deepseek-ai/schemastery` 由 Harness 自带。

### 2. 配 MetaOnticVideo（填一次密钥）

在 **插件 → 已安装插件 → MetaOnticVideo → 查看 / 配置** 里填：

| 填什么 | 例子 |
| --- | --- |
| 上游 | New API 中转站（一般选这个） |
| API Key | `sk-...`（你自己的） |
| 基础地址 | `https://ai.bileego.com`（不要带 `/v1`，不要重复 `/api/v3`） |
| 模型 | `doubao-seedance-2-0-mini-260615` |
| 每段秒数 | `14`（建议 ≤14，避免刚好卡 15 秒被上游拒） |
| 单段上限 | `15` |
| 分辨率 | `480p` 草稿 / `720p` 定稿 |
| 画面比例 | `9:16` 或 `16:9` |

官方火山方舟（`provider: ark`）：

```text
API Key: sk-...
模型: doubao-seedance-2-5-260628
```

也可以直接对 Agent 说「把视频插件的地址/Key/模型改成 …」，它会调 `video_config` 写回 profile。  
手改的话，配置在 profile 的 patch 层（桌面端 `~/.dsh/profiles/desktop/cordis.patch.yml`）：

```yaml
- id: metaontic-video
  name: metaontic-video
  config:
    provider: newapi
    apiKey: 'sk-...'
    baseUrl: 'https://ai.bileego.com'
    model: 'doubao-seedance-2-0-mini-260615'
    duration: 14
    ratio: 9:16
    resolution: 480p
```

不想把 Key 写进文件，也可以用环境变量：`NEWAPI_API_KEY`、`NEWAPI_VIDEO_BASE_URL`、`NEWAPI_VIDEO_MODEL`，官方用 `ARK_API_KEY`。  
**Key 只放在自己的 profile / 环境变量里，不要提交到仓库，也不要写进 README。**

### 3. 验证 MetaOnticVideo

新会话里问一句：

```text
你现在有哪些视频工具？把名字列出来。
```

应该看到 `video_generate`、`video_extend`、`video_status`、`video_config`（旧版是 `seedance_*`）。  
也可以：

```text
帮我查一下视频插件当前配置（video_config action=get）。
```

如果工具不存在：插件没启用、装完没重启，或者你在旧会话里；先重启 / 新建会话。  
想真跑一次，可以让它生成一个 4 秒 480p 的测试片段（会消耗一点额度）：

```text
用 4 秒、480p、9:16 生成一个测试片段：清晨木桌上一杯冒热气的豆浆。
```

### 4. 装本 Agent

侧栏 **插件 → 添加插件 / 安装**，填：

| 来源 | 填什么 |
| --- | --- |
| GitHub | `github:wxsimon2022/metaontic-video-agent` |
| 固定版本 | `github:wxsimon2022/metaontic-video-agent#v0.1.1` |
| 本地目录 | 本目录绝对路径，例如 `~/code/metaontic-video-agent` |
| tgz | `~/Downloads/metaontic-video-agent-0.1.1.tgz` |

也可以在本仓库自己打包：

```bash
npm pack
# 生成 metaontic-video-agent-0.1.1.tgz
```

或者让 Agent 用 `plugin_manager` 的 `install_bundle`，`target` 填同一个 spec。

安装后**新建会话**，在智能体选择器里选 **视频导演**。已有会话不会自动切换预设。

### 5. 验证 Agent

新会话里问：

```text
你现在是什么角色？有哪些视频工具？会不会先让我确认脚本？
```

应该看到：

- 身份是「视频导演」；
- 有 `video_generate` / `video_extend` / `video_status` / `video_config`；
- 有 `bash`、文件读写、`skill`、`present`；
- 明确说「先写脚本，你确认后才生成」。

再发一个测试创意（不用真生成），看它是否只给脚本、不调用生成工具：

```text
测试一下流程，别生成：一个人早上咖啡洒了，最后笑了。15 秒竖屏，有台词。
```

它应该只写 `script.md` 并问「确认这版脚本吗？」，**不会**调用 `video_generate`。

## 第一次出片

打开 **视频导演** 会话，发一段创意：

```text
做一个 15 秒竖屏短视频：外卖员雨天送餐，客户开门递给他一把伞；
他下楼后发现这单正是刚才那个差评客户下的。真实生活感，有台词，不要字幕。
```

1. 它回一份完整脚本；
2. 你回「确认」或提修改；
3. 它才调用 `video_generate`；
4. 生成通常 1–5 分钟；
5. Agent 会把成片放到 `video-projects/<slug>/final/`，并把路径和任务 ID 发给你；如果只看到 `metaontic-video/` 下的原始输出，让它移到 `final/` 或直接看那个路径。

## 分享给别人

对方也要走上面的 **0–5 步**。这个包**不含 API Key，也不含 MetaOnticVideo 本身**。

三种分发方式：

| 方式 | 你做什么 | 对方怎么装 |
| --- | --- | --- |
| 发 tgz | 把 `metaontic-video-agent-0.1.1.tgz` 发给对方 | 插件页 → 安装 → 填 tgz 绝对路径 |
| GitHub | 把本目录推到仓库 | 插件页 → 安装 → 填 `github:你的用户名/metaontic-video-agent` |
| npm | `npm publish` | 插件页 → 安装 → 填 `metaontic-video-agent`（或 `@scope/名字`） |

npm 发布前先确认名字没被占：

```bash
npm view metaontic-video-agent version   # 404 = 可用
npm publish
```

**改包名要注意**：`cordis.patch.yml` 里 `skill-filesystem` 的技能目录是用
`createRequire(baseUrl).resolve('metaontic-video-agent/package.json')` 找的。
如果你发布成 `@your-scope/metaontic-video-agent` 或改了包名，这段字符串要同步改，否则技能挂不上。

会话记录不会随包走：对方装好后自己新建会话、发自己的创意。  
不要把 API Key 写进包里或 README 里；Key 只放在各自的 profile 配置中。

## 使用示例

发一段创意即可；Agent 会先回脚本，不会直接生成：

```text
做一个 15 秒竖屏搞笑视频：年轻人街头采访两位农村大爷，
有仨儿子的老人月月轮流搬家很无奈，五保户老人反而清闲自在，结尾第一个大爷愣住。
真实生活纪录感，中文同期声，不要字幕。
```

```text
给这瓶粉饼做一个 10 秒竖屏带货口播，真实自然光，像朋友聊天，不要播音腔。
产品图在 ./assets/powder.jpg。
```

确认脚本后，它会继续生成；也可以只改一镜：

```text
把 video-projects/two-grandpas 的第 2 镜重生成：人物保持一致，但把机位改成近景，
台词不变，只重生成这一镜，然后重新合成。
```

## 工作方式

每次制片的固定顺序：

1. **你发一段创意**（一句话或一段描述）。
2. **Agent 写 `script.md`**：钩子、分镜表、完整台词、声音、结尾。
3. **Agent 展示脚本并等你确认**；此时 `script.md` 里没有 `status: approved`，不会生成。
4. **你确认后**：写 `shots.json` + 逐镜提示词，把 `script.md` 标成 `approved`。
5. **调用 `video_generate`**：≤15 秒直出；>15 秒自动续拍合并；多镜分别生成。
6. **验收 / 合成 / 交付**：ffprobe、抽帧、ffmpeg 拼接，`present` 成片。

预设只做三件事：

1. **persona**：把 Agent 的身份定成「视频导演」，并写死「脚本确认前不生成」。
2. **工具**：挂载文件、Shell、待办、提问、技能、展示、压缩等常规工具；
   Seedance 工具来自 Host 侧的 MetaOnticVideo，不需要在本包里重复配置。
3. **技能**：`skills/video-director/SKILL.md` 是完整工作流；`references/` 里是提示词手册、
   ffmpeg 速查和项目模板，按需读取，不占常驻上下文。

## 配置

本包没有自己的配置项。生成参数（上游、模型、每段秒数、分辨率、水印等）都在
**MetaOnticVideo** 的配置里：

- 插件页：MetaOnticVideo 的配置卡片；
- 对话里：让 Agent 调 `video_config`；
- 手改：profile 的 `cordis.patch.yml` 中 `- id: metaontic-video` 那一项。

Agent 的默认规格写在 `cordis.patch.yml` 的 `persona.prefix` 里，想改默认画幅/时长/风格直接改这段文字，
重新安装或重载 bundle 后对**新会话**生效。

## 定制

| 想改 | 改哪里 |
| --- | --- |
| 默认时长/画幅/风格 | `cordis.patch.yml` → `persona.prefix` |
| 工作流、验收标准 | `skills/video-director/SKILL.md` |
| 提示词写法、范例 | `skills/video-director/references/prompt-playbook.md` |
| ffmpeg 命令 | `skills/video-director/references/ffmpeg.md` |
| 项目文件模板 | `skills/video-director/references/project-template.md` |
| 挂载的工具 | `cordis.patch.yml` → `plugins` 列表 |

改完重新安装本 bundle，新建会话验证。技能正文是每次加载时重读的，改 `SKILL.md` 不需要重装；
`references/` 也是按需读取，改完立即生效。

## 常见问题

| 现象 | 原因 / 怎么办 |
| --- | --- |
| 插件页找不到 MetaOnticVideo | spec 写错、网络不通；用 `github:wxsimon2022/metaontic-video` 重试 |
| 没有 `video_generate` 工具 | MetaOnticVideo 没装/没启用，或装完没重启，或还在旧会话；重启 + 新建会话 |
| 「未配置 API Key」 | 第二步没填或没保存 |
| 「鉴权失败」 | Key 错、过期，或中转站地址填错（不要带 `/v1`） |
| 「额度不足」 | 中转站余额不够，充值后再试 |
| 「内容审核拒绝」 | 去掉真人/名人/血腥/政治/未成年风险描述，改虚构人物和中性表述 |
| 「时长超限」 | 每段改成 14 秒，或拆镜 |
| 找不到 ffmpeg | `brew install ffmpeg`，或在配置里填 `ffmpegBinary` 绝对路径 |
| 生成很慢 | 正常，通常 1–5 分钟；用 `video_status` 查任务，不要重复提交 |
| 视频导演不在智能体选择器 | 本 bundle 没安装/没启用；插件页确认后新建会话 |
| 它没让我确认脚本 | 你可能在旧会话或旧预设；新建「视频导演」会话，并检查 `script.md` 的 `status` 是否为 `draft` |
| 成片在哪 | Agent 的成片在 `video-projects/<slug>/final/`；MetaOnticVideo 的原始输出/分段在 `metaontic-video/` 或配置的 `outputDir`（留空=会话工作目录下） |

## 限制

- 本包**不包含**生成后端；没有 MetaOnticVideo 或上游不可用时，Agent 只能写脚本，不能出片。
- 脚本确认是 persona + 技能 + `script.md` 的 `status` 共同约束，不是宿主级拦截；换掉 persona 或让 Agent 忽略技能，门槛会失效。
- 续拍的参考视频必须是上游能拉取的公网 URL；本地文件不支持直接续拍。
- 含人脸续拍通常需要同一上游账号下的受信原始产物，否则可能被内容审核拒绝。
- 不保证人脸、声线、口型和背景完全连续；多段拼接的接缝需要人工预览。
- 生成按上游计费；Agent 会先在脚本阶段停下，确认后才生成，避免误烧额度。

## 卸载

在插件页关闭/移除本 bundle 即可；它不写 profile 配置以外的任何状态。
`video-projects/` 里的项目文件会保留，按需删除。  
卸载 MetaOnticVideo 会让 `video_generate` 等工具消失，但不会删除已生成的视频文件。

## License

MIT

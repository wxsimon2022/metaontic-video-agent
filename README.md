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
script.md           分镜脚本 + 完整台词
shots.json          镜头清单、任务 ID、生成状态
prompts/01-*.txt    每镜最终提交的提示词
clips/              Seedance 分段
final/              合成后的成片
production.md       任务 ID、耗时、验收记录
```

## 安装

### 前置

1. 已安装并启用 **MetaOnticVideo** 插件，且上游可用（`video_generate` 能正常生成）。
2. 本机有 `ffmpeg` / `ffprobe`（多镜合成需要；单镜直出不需要）。

### 安装本包

在 Harness 侧栏打开 **插件 → 安装**，填下面任意一种 spec：

| 来源 | 填什么 |
| --- | --- |
| GitHub | `github:wxsimon2022/metaontic-video-agent` |
| 本地目录 | 本目录的绝对路径，例如 `~/code/metaontic-video-agent` |
| tgz | 打包文件的绝对路径，例如 `~/Downloads/metaontic-video-agent-0.1.0.tgz` |

也可以在仓库里自己打包：

```bash
npm pack
# 生成 metaontic-video-agent-0.1.0.tgz
```

或者让 Agent 用 `plugin_manager` 的 `install_bundle`，`target` 填同一个 spec。

安装后**新建会话**，在智能体选择器里选 **视频导演**。已有会话不会自动切换预设。

### 验证

新会话里问一句：

```text
你现在是什么角色？有哪些视频工具？
```

应该能看到：视频导演 persona、`video_generate` / `video_extend` / `video_status`、
`bash`、文件读写、`skill`、`present`。再让它读一下 `video-director` 技能，确认技能目录挂载成功。

## 分享给别人

这个包**不含 API Key，也不含 MetaOnticVideo 本身**。对方需要：

1. 装好 DeepSeek Harness（Desktop 或 Web）；
2. 装好并启用 **MetaOnticVideo**，填自己的中转站地址和 Key；
3. 本机有 `ffmpeg` / `ffprobe`；
4. 装这个 agent 包，新建会话选 **视频导演**。

三种分发方式：

| 方式 | 你做什么 | 对方怎么装 |
| --- | --- | --- |
| 发 tgz | 把 `metaontic-video-agent-0.1.0.tgz` 发给对方 | 插件页 → 安装 → 填 tgz 的绝对路径 |
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
3. **Agent 展示脚本并等你确认**；此时代码里没有 `status: approved`，不会生成。
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

- 对话里：让 Agent 调 `video_config`；
- 插件页：MetaOnticVideo 的配置卡片；
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

## License

MIT

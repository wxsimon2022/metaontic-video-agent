# metaontic-video-agent（已废弃）

> **这个包已经合并进 [metaontic-video](https://github.com/wxsimon2022/metaontic-video) 插件。**
> 从 `metaontic-video@2.1.0` 起，「视频导演」智能体预设、persona 内容、
> `skills/video-director` 技能目录都随插件一起安装，**不需要再装本包**。

## 我该怎么做

1. 在 profile 里加/更新插件（只用这一个包）：

   ```bash
   dsh plugin --profile desktop add github:wxsimon2022/metaontic-video
   ```

2. 从 profile 的 `dsh.profile.bundles` 和 `dependencies` 里删掉 `metaontic-video-agent`；
   不删也不会重复注册预设（本包现在什么都不插入），只是白占一个依赖。

3. 重启 Harness，插件页里应看到 **MetaOnticVideo**，新区里能选到 **视频导演**。

## 为什么合并

之前预设住在本包、工具住在插件包，两边各写一份硬规则（比如「≤15 秒不拆段」），
很容易漂移。合并成一个包后：一份规则、一个版本、一次安装。

预设与技能的新家：

- 预设定义：`metaontic-video/cordis.patch.yml` → `preset-video-director`
- 技能目录：`metaontic-video/skills/video-director/`
- 工具：`video_generate` / `video_extend` / `video_status` / `video_config`

## 本包现状

`cordis.patch.yml` 是一个空补丁（`[]`），不注册任何东西，仅为兼容残留安装保留。

# AGENTS.md — AstrBot 群聊日报报告视觉模板开发守则

本仓库是 AstrBot「群聊日常分析」插件的**报告视觉模板**仓库，可作为任意模板仓库的起点。
以下规则适用于仓库内的模板开发，以及 AI 辅助修改（AI 参与时必须以本文件为准）。

> 插件侧完整规范以
> [docs/REPORT_TEMPLATE_GUIDE.md](https://github.com/SXP-Simon/astrbot_plugin_qq_group_daily_analysis/blob/main/docs/REPORT_TEMPLATE_GUIDE.md)
> 为准（本仓库的 README 与该指南均指向同一契约）；两者冲突时以插件当前版本的指南为准。

## 仓库结构约定

- **模板目录**：`<模板名>/`，一个仓库可含多个独立模板目录，但**一次安装 / 一次打包只装一个模板**。
  模板名建议小写英文蛇形并带命名空间前缀（如 `gda_sky_diary`、`gda_miku_dream`），
  仅含小写字母、数字与下划线，≤50 字符，无空格与特殊字符，不与插件内置模板重名。
- 模板目录内放置插件契约的 HTML 文件（见「渲染变量契约」）：
  - `image_template.html` / `html_template.html` —— **至少其一**（image→长图，html→网页，可同时提供）；
  - `topic_item.html` / `user_title_item.html` / `quote_item.html` / `activity_chart.html` /
    `chat_quality_item.html` —— 可选子模块，缺失时自动回退插件默认手账对应文件；
  - 可选：`template.json`（展示元信息）、`preview.jpg/png` 或 `demo.jpg/png`（随包预览图）、
    `shared_styles.html` / `inline_assets.html`（共享样式片段与内联资源，参考内置模板）。
- 文件名与变量契约是插件的固定契约面；模板目录内允许放任意附加文件（安装器原样保留、运行时会忽略），
  但**配套脚本**（模板自检、预览图生成等）一律放**仓库根**，避免徒增安装体积。
- 仓库根 README 是仓库使用说明；本文件是开发守则，两者冲突时以本文件为准。

## 修改工作流（模板任何改动之后）

1. **校验**：运行仓库根的自检脚本（存在 `verify_*.py` 等时，用法见 README）。
   自检应至少：校验全部 HTML 的 Jinja2 语法，并以 StrictUndefined 实际渲染主骨架与全部子模块
   （任何变量缺失/结构错误立即失败）；若配合插件仓库路径，还应模拟打包 zip 走一遍安装/卸载。
   没有脚本时，逐项自查「渲染变量契约」与「资源引用」。
2. **同步更新说明与产物**（凡是改动模板，必须一并更新，防止仓库与模板脱节）：
   - **README**：模板显示名 / 目录结构 / 预览图与 demo 图说明 / 打包规范等发生变化的，同步更新仓库根 README；
   - **demo 图**：改动视觉样式后，若存在预览图生成脚本，运行它并**重新生成并提交仓库展示用 demo 图**
     （如 `assets/<模板名>-demo.jpg` 与 `-thumb.jpg`），以及随模板打包的预览图（如 `preview.jpg`），
     确保 README 中引用/展示的图片与最新模板视觉一致。
3. **提交**：一次提交包含「模板改动 + 自检通过」；README 更新与全部产物图（demo 图 + 随包预览图）一并提交。

## 渲染变量契约（插件固定，不可修改）

主骨架（`image_template.html` / `html_template.html`）接收：
`topics_html`、`titles_html`、`quotes_html`、`hourly_chart_html`、
`chat_quality_html`（可能为空串）、`message_count`、`participant_count`、
`total_characters`、`emoji_count`、`most_active_period`、`current_date`、
`current_datetime`、`total_tokens` / `prompt_tokens` / `completion_tokens`、
`hide_user_names`、`t2i_font_source` / `t2i_google_fonts_mirror` /
`t2i_gstatic_mirror` / `t2i_atri_font_mirror`。

子模块：

| 文件 | 变量 |
| --- | --- |
| `topic_item.html` | `topics` |
| `user_title_item.html` | `titles` |
| `quote_item.html` | `quotes` |
| `activity_chart.html` | `chart_data` |
| `chat_quality_item.html` | 直接展开 `title`/`subtitle`/`summary`/`dimensions` |

注意：`topic.detail`、`quote.reason` 是系统预渲染 HTML（含头像），输出必须加
`| safe`；其余字段由插件负责转义，模板不要二次转义；所有模板还共同获得
`hide_user_names` 与 `t2i_*` 字体配置。

## 资源引用（渲染链路约束，极易踩坑）

报告 HTML 是**字符串**形式交给远端 T2I 渲染服务，**渲染端没有模板文件的文件系统上下文**。
模板内引用的每一张图片/字体等资源，只允许两种写法：

| 写法 | 是否可用 | 说明 |
| --- | --- | --- |
| 稳定可访问的 HTTPS 链接（`https://` 开头） | ✅ | 公开可达、长期稳定、不依赖模板文件位置的正常链接 |
| 内联 `data:` URI / 内联 `<svg>` | ✅ | 自包含、零网络依赖，小图标/纯装饰优先 |
| 相对路径（`assets/bg.png`、`./img/x.png`） | ❌ | 远端渲染端无法解析，必然 404 |
| 本地路径（`/data/...`、`C:\...`、`file://`） | ❌ | 渲染端没有该文件系统上下文，与相对路径同样必然失败 |

> 模板内引用资源时，只需给出一个**能正常访问、长期稳定的 HTTPS 链接**（`https://` 开头、公网可达）即可；
> 相对路径与本机文件路径（`/data/...`、`C:\...`）在渲染时都不可用。

- 例外：模板目录内的预览图（`preview.*` / `demo.*`）用于插件 `/查看模板` 与 WebUI 画廊展示，
  走本地文件读取路径，**不经过渲染管线**，可用。
- 大小建议：小图标/贴纸（<50KB）用 base64 `data:` URI 或内联 `<svg>`；
  大装饰图（背景/立绘等）可用稳定的 `https://` 链接引用（base64 会膨胀约 33%，且每次渲染都整串携带）。

## 命名与元信息

- 模板名建议小写英文蛇形并带命名空间前缀（如 `gda_sky_diary`）；≤50 字符；
  禁空格、路径分隔符与文件系统危险字符（`\ / : * ? " < > |` 及控制字符）；与内置模板重名会被拒绝。
- `template.json` **仅支持 JSON**（不接受 YAML），放模板根目录，全部字段可选、字符串 ≤100 字符：
  `{"name": 显示名, "desc": 描述, "tag": 风格标签, "tag_color": 标签配色}`
  （`tag_color` 为 antd Tag 颜色名：default/blue/green/cyan/pink/purple/red/orange/lime/geekblue）。
  文件缺失、解析失败或字段为空时自动忽略，模板依旧可用。

## 打包与安装器校验（强制）

- 一个 zip / 一次安装只装一个模板；模板根目录至少含 `image_template.html` 或 `html_template.html` 其一。
- 允许外层套一层目录（GitHub 归档的 `repo-main` / `repo-master` 形式自动剥离，模板名从根目录名推断）。
- 限额（安装器强制）：解压后总量 ≤ 5MB、单文件 ≤ 3MB、成员 ≤ 100、预览图 ≤ 3MB、
  GitHub 下载体 ≤ 5MB；超过会被拒绝。
- 安全：路径穿越（`../`、绝对路径）、控制字符文件名、加密 zip 会被拒绝；
  符号链接（模板目录本身或其内主模板文件）会被拒绝加载。
- 同时提供 `image_template.html` 与 `html_template.html` 时两者各司其职
  （image→长图，html→网页；html 渲染失败自动回退 image），互不冲突。

## 禁止事项

- 修改/复制插件功能代码（本仓库不含插件实现；模板只能消费上述变量契约）。
- 硬编码本机或他人的绝对路径（含预览图生成脚本的输出路径）。
- 引入新运行时依赖（模板是纯 HTML/Jinja，无 Python 依赖）。
- 改动 HTML 文件名与变量契约（这是与插件渲染器的契约面）。

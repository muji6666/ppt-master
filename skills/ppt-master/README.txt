================================================================================
  PPT Master — AI 驱动的可原生编辑 PPTX 生成技能
================================================================================

PPT Master 是一个 Claude Code 技能，通过多角色 prompt 流水线将源文档（PDF/DOCX/
URL/Markdown）转换为包含真实 PowerPoint 形状（DrawingML）的可原生编辑 PPTX 文件。


一、运作原理
================================================================================

1. 核心流水线

  源文档 → 源内容转换 → 项目初始化 → [模板] → 策略师 → [图像获取] → 执行器
  → 质量检查 → 后处理 → 导出 PPTX

  流水线严格串行执行，8 条全局规则（见 SKILL.md）确保每步顺序进行，禁止跨阶段
  打包、预判执行和批量 SVG 生成。标记 ⛔ 阻断的步骤需要用户明确确认后才继续。

2. 为什么用 SVG 作为中间格式

  SVG 是唯一同时满足三个流水线角色的格式：
  - AI 可以可靠生成
  - 人类可以在浏览器中预览和调试
  - 脚本可以精确转换为 DrawingML（PPTX 底层 XML）

  SVG 与 DrawingML 共享相同的概念模型（绝对坐标二维矢量图形），转换是同一概念
  在两种方言之间的翻译。

3. 角色系统

  流水线使用单一主代理内的角色切换（非并行子代理），每个角色按需加载：

  ┌─────────────────────┬──────────────────────────────┬──────────────────┐
  │ 角色                │ 定义文件                     │ 职责             │
  ├─────────────────────┼──────────────────────────────┼──────────────────┤
  │ 策略师 Strategist   │ references/strategist.md     │ 内容分析、八项   │
  │                     │                              │ 确认、设计规范   │
  ├─────────────────────┼──────────────────────────────┼──────────────────┤
  │ 图像生成器          │ references/image-generator.md│ AI 图像生成      │
  │ Image_Generator     │                              │                  │
  ├─────────────────────┼──────────────────────────────┼──────────────────┤
  │ 图像搜索器          │ references/image-searcher.md │ 网络图片搜索     │
  │ Image_Searcher      │                              │                  │
  ├─────────────────────┼──────────────────────────────┼──────────────────┤
  │ 执行器 Executor     │ references/executor-base.md  │ 逐页生成 SVG +   │
  │                     │ + 风格变体文件               │ 演讲者备注       │
  ├─────────────────────┼──────────────────────────────┼──────────────────┤
  │ 模板设计师          │ references/template-designer │ 模板库创建       │
  │ Template_Designer   │ .md                          │                  │
  └─────────────────────┴──────────────────────────────┴──────────────────┘

4. 规范防漂移机制

  策略师阶段产出两个文件：
  - design_spec.md — 人类可读的设计叙事（为什么要这样设计）
  - spec_lock.md   — 机器可读的执行契约（精确的 HEX 色值、字体、图标库等）

  SKILL.md 要求在生成每个 SVG 页面前必须读取 spec_lock.md，确保在 20+ 页的长
  文档中颜色和字体保持逐字一致，防止上下文压缩导致的风格漂移。

5. 工程转换阶段（后处理）

  三个脚本必须严格按顺序执行（绝不能合并为一条命令）：

  Step 7.1  total_md_split.py  → 拆分演讲者备注为每页独立文件
  Step 7.2  finalize_svg.py    → 图标嵌入、图片裁剪/内嵌、文本扁平化等
  Step 7.3  svg_to_pptx.py     → SVG 转 DrawingML，生成原生 .pptx

  scripts/svg_finalize/ 下的模块有两个消费者：
  - 磁盘端：finalize_svg.py 写入 svg_final/
  - 内存端：svg_to_pptx.py 在内存中展开图标和扁平化文本


二、主要步骤（7 步流水线）
================================================================================

  Step 1 — 源内容转换
      将 PDF/DOCX/PPTX/EPUB/HTML/URL/XLSX 等转换为 Markdown。
      使用 scripts/source_to_md/ 下的对应转换脚本。

  Step 2 — 项目初始化
      创建标准项目目录结构并导入源素材。
      命令：project_manager.py init <名称> --format ppt169

  Step 3 — 模板选项
      默认自由设计。仅在用户显式提供模板目录路径时触发；模板名称、风格描述、
      模糊匹配均不会触发。

  Step 4 — 策略师阶段（⛔ 唯一阻断点）
      AI 阅读 references/strategist.md，分析内容后输出八项确认：
      画布格式 → 页数 → 受众 → 风格 → 配色 → 图标 → 字体 → 图片
      用户确认后，输出 design_spec.md 和 spec_lock.md。

  Step 5 — 图像获取阶段（条件性）
      当设计规范中有 ai 或 web 来源的图片时才触发。
      分别调用 image_gen.py（AI 生图）或 image_search.py（网络搜图）。

  Step 6 — 执行器阶段
      逐页生成 SVG → svg_output/，自动启动浏览器实时预览（http://localhost:5050），
      运行 svg_quality_checker.py 质量检查，生成演讲者备注 → notes/total.md。

  Step 7 — 后处理与导出
      严格按序执行 total_md_split.py → finalize_svg.py → svg_to_pptx.py。
      最终输出 exports/<名称>_<时间戳>.pptx。


三、参考资料索引
================================================================================

  入口文件：
    SKILL.md                 主工作流入口，7 步流水线的完整定义

  角色定义（references/）：
    strategist.md            策略师 — 内容分析与设计规划
    executor-base.md         执行器通用指南
    executor-general.md      通用灵活风格
    executor-consultant.md   咨询风格
    executor-consultant-top.md 顶级咨询风格（MBB 级别）
    image-base.md            图片获取通用框架
    image-generator.md       AI 图像生成角色
    image-searcher.md        网络图片搜索角色
    template-designer.md     模板创建角色
    visual-review.md         逐页评分表式视觉复核

  技术规范（references/）：
    shared-standards.md      SVG 禁用特性黑名单、PPTX 约束
    canvas-formats.md        输出格式规范（16:9、4:3、社交媒体等）
    animations.md            动画/切换参考
    image-layout-patterns.md 72 种版式模式（主结构 + 修饰层）
    image-layout-spec.md     图片容器版式尺寸计算
    svg-image-embedding.md   SVG 图片嵌入规范

  AI 图像素材库（references/）：
    image-palettes/          14 套 AI 图像调色板预设
    image-renderings/        22 种视觉渲染风格预设
    image-type-templates/    12 种图像构图类型模板
    ai-image-comparison/     调色板/渲染/类型比对清单

  独立工作流（workflows/）：
    topic-research.md        无源素材时的网络调研
    create-template.md       版式/整库模板创建
    create-brand.md          品牌身份模板创建
    resume-execute.md        分屏模式 B 阶段接续
    verify-charts.md         图表坐标校准
    customize-animations.md  对象级动画调优
    live-preview.md          浏览器实时预览
    generate-audio.md        TTS 语音旁白与视频导出
    visual-review.md         逐页视觉自查

  模板库（templates/）：
    brands/                  品牌身份预设（颜色/字体/Logo/语调）
    layouts/                 页面版式模板库
    decks/                   完整整库复刻（身份 + 结构）
    charts/                  图表/信息图 SVG 模板
    icons/                   图标库（Tabler、Phosphor、Simple Icons 等）

  技术文档（docs/）：
    technical-design.md       架构设计、设计理念、为什么选 SVG
    faq.md                    常见问题解答
    rules/code-style.md       Python 代码风格规范
    rules/prompt-style.md     Reference 文档风格规范


四、可用的 Slash Command
================================================================================

  PPT Master 不是通过传统 slash command 调用，而是通过工作流名称触发。
  以下工作流对应独立的功能入口：

  /ppt-master (默认)
      主入口。当用户说"生成PPT""做PPT""create PPT""make presentation"
      或提及 "ppt-master" 时自动激活，启动完整的 7 步流水线。

  独立工作流（在主流水线之外单独触发）：
  ┌──────────────────────────┬────────────────────────────────────────────┐
  │ 触发词 / 场景            │ 对应工作流                  │ 功能          │
  ├──────────────────────────┼────────────────────────────┼──────────────┤
  │ "创建模板" /             │ workflows/create-template   │ 版式/整库     │
  │ "create template"        │ .md                         │ 模板创建      │
  ├──────────────────────────┼────────────────────────────┼──────────────┤
  │ "建立品牌" /             │ workflows/create-brand      │ 品牌身份      │
  │ "set up brand"           │ .md                         │ 模板创建      │
  ├──────────────────────────┼────────────────────────────┼──────────────┤
  │ "继续生成 projects/"     │ workflows/resume-execute    │ 分屏模式      │
  │                          │ .md                         │ B 阶段接续    │
  ├──────────────────────────┼────────────────────────────┼──────────────┤
  │ "校准图表" /             │ workflows/verify-charts     │ 图表坐标      │
  │ "verify charts"          │ .md                         │ 校准          │
  ├──────────────────────────┼────────────────────────────┼──────────────┤
  │ "live preview" /         │ workflows/live-preview      │ 浏览器实时    │
  │ "预览" / "看效果"        │ .md                         │ 预览          │
  ├──────────────────────────┼────────────────────────────┼──────────────┤
  │ "调整动画" /             │ workflows/customize-        │ 对象级动画    │
  │ "customize animations"   │ animations.md               │ 调优          │
  ├──────────────────────────┼────────────────────────────┼──────────────┤
  │ "生成语音旁白" /         │ workflows/generate-audio    │ TTS 语音旁白  │
  │ "generate audio"         │ .md                         │ 与视频导出    │
  ├──────────────────────────┼────────────────────────────┼──────────────┤
  │ "视觉自检" /             │ workflows/visual-review     │ 逐页视觉      │
  │ "visual review"          │ .md                         │ 自查          │
  ├──────────────────────────┼────────────────────────────┼──────────────┤
  │ 仅有主题无素材           │ workflows/topic-research    │ 网络调研      │
  │                          │ .md                         │ 搜集素材      │
  └──────────────────────────┴────────────────────────────┴──────────────┘


五、.env.example 填写方法
================================================================================

  .env.example 是可选的环境变量配置模板，用于启用图片生成、网络搜图和语音旁白等
  可选功能。拷贝为 .env 后编辑：

    cp .env.example .env

  .env 文件查找顺序（只读第一个存在的文件，不合并）：
    1. 当前工作目录下的 ./.env
    2. 仓库根目录下的 .env（仅 clone 模式）
    3. ~/.ppt-master/.env（用户级永久配置，推荐 skill marketplace 安装者使用）

  ─── 图片生成（AI 生图）───────────────────────────────────────────────

  核心配置（以 OpenAI 为例，推荐默认）：
    IMAGE_BACKEND=openai              # 选择后端，必填
    OPENAI_API_KEY=sk-xxx             # OpenAI API 密钥
    OPENAI_MODEL=gpt-image-2          # 模型名称
    OPENAI_BASE_URL=http://...        # 代理地址（可选）
    OPENAI_OUTPUT_FORMAT=png          # 输出格式：png/jpeg/webp
    OPENAI_OUTPUT_COMPRESSION=80      # 仅 jpeg/webp，0-100
    OPENAI_BACKGROUND=auto            # gpt-image-2：auto/opaque
    OPENAI_MODERATION=auto            # auto/low

  切换到其他后端只需修改 IMAGE_BACKEND：
    IMAGE_BACKEND=gemini    → 需配 GEMINI_API_KEY + GEMINI_MODEL
    IMAGE_BACKEND=qwen      → 需配 QWEN_API_KEY + QWEN_MODEL
    IMAGE_BACKEND=zhipu     → 需配 ZHIPU_API_KEY + ZHIPU_MODEL
    IMAGE_BACKEND=minimax   → 需配 MINIMAX_API_KEY + MINIMAX_MODEL
    IMAGE_BACKEND=volcengine → 需配 VOLCENGINE_API_KEY + VOLCENGINE_MODEL

  查看所有可用后端：python3 scripts/image_gen.py --list-backends

  ─── 网络图片搜索 ───────────────────────────────────────────────────

  Openverse 和 Wikimedia 无需 API Key 即可使用，但图片质量不稳定。
  建议配置以获得更好的商业图库效果：

    PEXELS_API_KEY=你的-pexels-key      # 在 https://www.pexels.com/api/ 注册
    PIXABAY_API_KEY=你的-pixabay-key    # 在 https://pixabay.com/api/docs/ 注册

  ─── 语音旁白 TTS ──────────────────────────────────────────────────

  edge-tts 是默认旁白后端，无需 API Key。
  如需高质量云端旁白或复刻音色，配置以下其一：

    ELEVENLABS_API_KEY=你的-key         # ElevenLabs
    MINIMAX_API_KEY=你的-key            # MiniMax
    QWEN_API_KEY=你的-key               # 通义千问（DashScope）
    COSYVOICE_API_KEY=你的-key          # CosyVoice（DashScope）
    DASHSCOPE_API_KEY=你的-key          # DashScope 通用 Key

  复刻音色：先在对应提供商控制台复刻得到 voice_id，再用 --voice-id 传参。

  可选 TTS 接入点覆盖（默认国内地址，海外用户可切换）：
    MINIMAX_TTS_BASE_URL=https://api.minimax.io/v1/t2a_v2
    QWEN_TTS_BASE_URL=https://dashscope-intl.aliyuncs.com/api/v1/...
    COSYVOICE_TTS_BASE_URL=https://dashscope.aliyuncs.com/api/v1/...

  ─── 注意事项 ──────────────────────────────────────────────────────

  - 不再支持 IMAGE_API_KEY / IMAGE_MODEL / IMAGE_BASE_URL 通用变量，
    请使用各提供商专属的 *_API_KEY 变量。
  - 环境变量优先级高于 .env 文件。多提供商 Key 可以共存于同一个 .env，
    通过修改 IMAGE_BACKEND 的值切换后端。
  - 技能 marketplace 安装者建议将 .env 放在 ~/.ppt-master/.env，
    作为用户级永久配置，无需每个项目都重新设置。


六、脚本目录结构
================================================================================

  scripts/
  ├── source_to_md/     文档 → Markdown 转换器
  │   ├── pdf_to_md.py       PDF 转换
  │   ├── doc_to_md.py       DOCX/EPUB/HTML/LaTeX 转换
  │   ├── excel_to_md.py     XLSX/XLSM 转换
  │   ├── ppt_to_md.py       PPTX 转换
  │   └── web_to_md.py       URL/微信公众号 转换
  ├── image_backends/   AI 图像生成后端（openai/gemini/qwen/zhipu/...）
  ├── image_sources/    网络图片搜索供应商（pexels/pixabay/openverse/...）
  ├── tts_backends/     语音旁白后端（edge/elevenlabs/minimax/qwen/...）
  ├── svg_finalize/     SVG 后处理（图标嵌入/图片裁剪/文本扁平化/...)
  ├── svg_to_pptx/      SVG → DrawingML 转换引擎
  ├── pptx_to_svg/      PPTX → SVG 逆向转换（模板导入用）
  ├── svg_editor/       浏览器实时预览服务器
  ├── template_import/  模板导入辅助
  ├── docs/             脚本文档（按主题分类）
  ├── project_manager.py     项目初始化/验证
  ├── image_gen.py           AI 图像生成调度器
  ├── image_search.py        网络图片搜索调度器
  ├── finalize_svg.py        SVG 后处理入口
  ├── svg_to_pptx.py         PPTX 导出入口
  ├── svg_quality_checker.py SVG 质量检查
  ├── total_md_split.py      演讲者备注拆分
  ├── notes_to_audio.py      TTS 语音旁白生成
  ├── update_spec.py         规范传播（spec_lock 变更同步到 SVG）
  ├── update_repo.py         仓库更新
  └── project_utils.py       公共工具函数

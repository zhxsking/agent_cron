# arXiv 每日论文推荐

每天早上从 arXiv 筛选与用户研究方向高度相关的最新论文 5 篇，输出"一句话摘要 + 是否值得精读"判断，并生成 HTML 报告。

## 【研究方向关键词】（仅用于英文检索，中文仅用于最终标题翻译）

- **AI / 大语言模型**：large language model, LLM, GPT, instruction tuning, RLHF, alignment, agent
- **Agent / 多智能体**：AI agent, multi-agent, tool use, ReAct, planning, reasoning
- **深度学习 / 机器学习**：deep learning, machine learning, neural network, transformer
- **深度估计 / 3D 视觉**：depth estimation, monocular depth, 3D vision, 3D reconstruction, NeRF, Gaussian Splatting, point cloud
- **多模态 / 大模型**：multimodal, vision-language model, VLM, MLLM, foundation model

> 注意：arXiv 论文标题和摘要均为英文，检索**只使用上述英文关键词**，不要用中文关键词检索（无效果，会浪费调用次数）。中文仅在第4步生成中文译名时使用。

## 【执行步骤】

### 1. 判断是否需要重新抓取
先检查 `arxiv_daily` 目录下今天日期对应的文件是否已存在：
- 若存在 → 直接读取该 HTML 文件内容，在对话中原样展示当天的5篇推荐摘要（标题+一句话摘要+星级），**不重复抓取**，直接结束任务。
- 若不存在 → 继续执行第2步。

### 2. 抓取候选论文池
- 数据源：arXiv API，端点为 ``
  - 请求示例：`"large language model"&sortBy=submittedDate&sortOrder=descending&max_results=20`
  - 分类范围：`cs.AI`、`cs.CL`、`cs.CV`、`cs.LG`、`cs.RO`、`cs.MA`
- 检索方式：对上述每个英文关键词组合分别请求，**每个关键词取最新 20 条**作为候选
- 时间过滤：只保留 `submittedDate` 或 `updated` 在过去 24 小时内的条目（即今天日期对应的 latest 版本）
- 将所有关键词检索结果合并，按 arXiv ID 去重（同一篇论文命中多个关键词只保留一次，并记录命中的关键词数量，用于后续评分）

### 3. 跨天去重（近7天）
- 读取 `arxiv_daily` 目录下**过去7天**已生成的 HTML 文件（若目录/文件不存在则跳过此步）
- 从文件内容中提取此前已推荐过的 arXiv ID 列表
- 将候选池中与该列表重复的 ID（包括同一论文的不同版本 v1/v2/v3，即 ID 数字部分相同）剔除

### 4. 筛选并生成最终 5 篇
从去重后的候选池中挑选最相关的 5 篇，优先级参考：
- 命中关键词数量（命中 ≥2 个关键词优先于只命中 1 个）
- 摘要与研究方向的语义相关度

每篇输出以下字段：
- **标题**：英文标题保留原文，附中文译名（如：`Title in English` / 中文译名）
- **作者**：第一作者 + 通讯作者（若摘要页未明确标注通讯作者，则填第一作者 + 最后一位作者）
- **arXiv 链接**：``（摘要页），如需 PDF 则为 ``
- **一句话中文摘要**：不超过 50 字
- **值得精读判断**：⭐⭐⭐ 强烈推荐 / ⭐⭐ 推荐 / ⭐ 一般，并给出 1 句理由。评分时参考以下硬指标（至少满足2条给⭐⭐⭐，满足1条给⭐⭐，均不满足给⭐）：
  1. 命中 2 个及以上关键词分类
  2. 摘要提及有开源代码/数据集（如出现 "code available", "github.com" 等字样）
  3. 方法为新范式/新架构，而非对已有工作的增量改进（微调参数、复现实验等视为增量）
  4. 作者单位/机构在摘要页可见，且为知名实验室或高校（如 Google DeepMind、OpenAI、Meta AI、Stanford、MIT 等）

### 5. 展示与保存
- 无论文件是新生成还是读取已有，**均在对话中展示**5篇推荐的完整摘要内容（标题、一句话摘要、星级），不要静默无输出
- 同时将完整报告写入 HTML 文件（见下方路径与样式规范），写入过程无需额外征求用户确认

### 6. 失败处理
- 若某个关键词的 arXiv API 请求失败：等待 5 秒后重试，最多重试 2 次
- 若重试后仍失败：跳过该关键词，继续处理其余关键词，最终在输出末尾注明"以下关键词检索失败：XXX"
- 若全部关键词均检索失败：告知用户"今日 arXiv 抓取失败"及具体原因（如网络错误/API 返回异常），不生成文件

---

## HTML 严格风格规范（学术报告风）

使用 html-report skill 生成 HTML 格式的论文推荐报告，文件应自包含、美观、可独立打开浏览。

**字体加载（必须加载 Web 字体，不得使用系统字体回退或伪斜体）：**
- 标题/正文：Lora（需加载包含 ital 轴的完整字重，即 `Lora:ital,wght@0,400..700;1,400..700`），禁止使用 `font-style: italic` 伪斜体模拟，所有斜体必须通过真实字体的 ital 轴实现
- 代码/数据/标签/图注：JetBrains Mono，表格内数字必须加 `font-variant-numeric: tabular-nums`
- 数学变量与公式：统一使用 KaTeX 渲染，不得使用 `<em>` 标签或 CSS 伪斜体替代单个数学符号

**色彩系统（Palette）：**
```css
:root {
  --bg:      #F4EFE4;
  --bg2:     #ECE4D4;
  --ink:     #2E2A22;
  --muted:   #96897A;
  --rule:    #96897A;
  --accent:  #B54A2C;
  --accent2: #24555A;
}
```

**背景与材质约束（最高优先级，必须严格遵守）：**
- 页面背景 background 只允许使用纯色 `var(--bg)`，绝对禁止 background-image、SVG pattern、噪点滤镜（feTurbulence）、paper-texture、grid-texture、repeating-linear-gradient、radial-gradient 等任何叠加纹理或图案
- 卡片与代码块底色使用 `var(--bg2)`，同样为纯色，不加任何纹理
- "纸面"效果仅通过色相与色温（暖白米色）体现，不通过任何视觉材质效果体现
- 禁止使用任何 CSS noise、grain、texture 相关的背景效果

**组件规范：**
- 表格：三线表样式，仅表头下方与表底各一条 `--rule` 线，行间用 `--muted` 极低透明度（约 0.06）斑马纹
- 代码块：`--bg2` 底色 + 左侧 3px `--accent2` 竖线，无阴影、无大圆角（border-radius 不超过 2px）
- 公式块：独立居中显示，右侧小号 JetBrains Mono 编号
- 分割线：全篇统一使用 1px `--rule` 细线，禁止使用粗框线

**视觉风格：**
- Academic paper aesthetic，克制严肃，无装饰性背景效果
- 排版美观，有良好的视觉层次，配色舒适，适合长时间阅读

---

## 禁止操作（必须严格遵守）

- **禁止使用浏览器工具**：不得使用 browser_navigate、browser_take_screenshot、browser_click、browser_snapshot 等任何浏览器相关工具
- **禁止截图操作**：不得使用任何截图功能
- **禁止网页抓取预览**：不得通过浏览器打开生成的 HTML 文件进行预览或截图
- 所有论文信息通过 WebSearch 和 WebFetch 获取，图表通过 SVG 代码生成，HTML 文件通过 Write 工具直接写入
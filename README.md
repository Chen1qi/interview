# Java 面试题库项目交接文档

> 本文档面向后续开发者（或 AI 助手），说明项目结构、功能、数据格式和开发规范。

## 一、项目概览

| 项目 | 说明 |
|------|------|
| 类型 | 纯前端单页应用（无后端、无构建工具、无框架依赖） |
| 文件 | 只有一个 `index.html`（约 10270 行） |
| 线上地址 | https://chen1qi.github.io/interview/ |
| 仓库 | https://github.com/Chen1qi/interview |
| 题目总数 | 279 题（全部含双答案） |
| 用户数据 | 存浏览器 localStorage，不上传服务器 |

## 二、文件结构

```
interview/
├── index.html          # 全部代码（CSS + 题库数据 + JS 逻辑）
├── README.md           # 本文档
└── .github/workflows/  # （未使用，Pages 用 legacy 模式从 main 分支直接部署）
```

## 三、核心数据结构

### 3.1 题库数组 QUESTIONS（约 337 行 ~ 9890 行）

```javascript
const QUESTIONS = [
  {
    c: "分类名",       // 如 "基础"、"Spring"、"场景实战"
    d: "难度",         // "★" / "★★" / "★★★"
    q: "题目文本",      // 纯文本
    a: `普通答案 HTML`,  // 通俗版答案，用 <p>/<ul>/<table>/<blockquote> 等标签
    b: `八股文详解 HTML` // 深度版答案（分点详解+源码+面试标准答法），所有题都有
  },
  // ...共 279 个对象
];
```

**分类及题数**（12 个分类）：
基础44、Spring30、MySQL24、Redis24、JVM23、并发22、网络22、分布式21、框架21、数据结构与算法18、集合16、场景实战14

**答案里可用的 HTML 标签**：`<p> <ul> <ol> <li> <strong> <code> <pre><code> <blockquote> <table> <h5>`

### 3.2 ⚠️ 重要转义规则（新增题目必读）

答案的 `a` 和 `b` 字段用**反引号模板字符串**包裹，因此：

1. **答案中出现 `${}` 必须写成 `\${}`**，否则 JS 把它当模板插值解析，直接报语法错误（曾在 MyBatis `#{}`/`${}` 题目上踩过此坑）
2. HTML 内容中的 `<` `>` 在 code 标签内写成 `&lt;` `&gt;`
3. 反引号字符串内不能出现未转义的反引号

### 3.3 用户状态 state（localStorage，key = `java_interview_state_v1`）

```javascript
state = {
  seen: {},        // id -> true，已练习过的题
  mastered: {},    // id -> true，已掌握的题
  activeCats: [],  // 当前选中的分类（单选，[] 表示全部）
  currentId: null, // 当前题目 id
  reviewMode: null,// null | 'mastered' | 'seen' 复习模式
  history: [],     // 浏览历史（题目 id 有序数组）
  historyIdx: -1   // 当前在历史中的位置
}
```

注意：save() 只持久化 `seen / mastered / activeCats`，其余字段会话内有效。

## 四、功能清单与实现位置

以下按代码中出现的大致顺序列出（行号为约数，以函数名搜索为准）：

| 功能 | 函数/位置 | 说明 |
|------|----------|------|
| 题库数据 | `const QUESTIONS` | 337 行起 |
| 状态管理 | `load() / save()` | localStorage 读写 |
| 分类筛选 | `getCategories() / renderChips() / toggleCat(c)` | 单选模式；标签显示各分类题数；选分类自动跳该分类第一题并清除复习模式 |
| 统计 | `renderStats()` | 总题数/已掌握/已练习，进度条 |
| 出题渲染 | `renderQuestion(id, isBack, autoExpand)` | 核心渲染函数。题号按当前筛选池计算"第X/Y题"；默认展开答案+八股文（autoExpand=false 时收起） |
| 英文发音 | `speak(text) / wrapWords(container)` | Web Speech API。wrapWords 用 TreeWalker 遍历文本节点（跳过 pre/code），把 3+ 字母英文词包成 `<span class="w">`，点击朗读（en-US, 0.8 倍速） |
| 八股文展开 | `toggleBagwen()` | 收起/展开 b 字段内容 |
| 复习模式 | `openReview(type)` | 点击统计区"已掌握/已练习"进入，再点退出；getFiltered() 优先按 reviewMode 过滤；进入时自动跳第一题并展开答案 |
| 掌握标记 | `markMastered() / updateMasteredBtn()` | 切换当前题的 mastered 状态 |
| 下一题 | `next()` | 复习模式：池内顺序下一题；正常模式：随机（优先未掌握的） |
| 上一题 | `prev()` | **基于 history 浏览历史回退**（用户强需求：上一题必须是刚看过的那题）；无历史时复习模式按池顺序、正常模式 fallback 到 next() |
| 自动展开 | `autoExpandAnswer()` | 复习模式翻页后展开答案+八股文 |
| 分类切换 | `toggleCat(c)` | 单选；退出复习模式；跳到分类第一题 |
| 重置 | `resetAll()` | 清空 localStorage 记录 |

### 浏览历史机制（重要）

- `renderQuestion` 每次渲染新题（非回退）时：先截断 `history` 到 `historyIdx+1`，再 push 当前 id
- `prev()` 传 `isBack=true`，只移动 `historyIdx` 指针、不写历史
- 这实现了浏览器式"后退"体验

## 五、页面 UI 结构

```html
<div class="wrap">
  统计区 .stats（总题数 | 已掌握[可点击进复习] | 已练习[可点击进复习]）
  进度条 .progress
  筛选区 .filters > .chips（全部(279) | 基础(44) | ...）
  题目卡片 #card
    ├── 题目 #question
    ├── 答案区 #answerWrap > #answer
    │     └── 八股文 #bagwenToggle > #bagwenBtn + #bagwenBody
    └── 操作按钮 .actions（查看答案 | 标记掌握 | ⬅上一题 | ⏭下一题）
</div>
<footer>（🎲随机一题 | 🔁重置）
复习弹窗 DOM（#reviewMask/#reviewModal，当前 openReview 已改为页面内模式，弹窗仅作兼容保留）
```

### 分类标签颜色 CSS

`.tag.c-基础` 到 `.tag.c-场景实战`，新增分类需在 CSS 里加对应 `.tag.c-新分类名{background:...;color:...}`。

## 六、开发规范（给后续 AI 的重要提示）

1. **单文件架构**：所有改动都在 index.html 里，不引入构建工具和外部依赖
2. **验证手段**：每次改完必须跑语法检查：
   ```bash
   node -e 'const fs=require("fs");const html=fs.readFileSync("index.html","utf8");
   try{new Function([...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]).join("\n"));console.log("✅ 语法通过");}catch(e){console.log("❌ "+e.message);}'
   ```
   并确认题库仍能 eval：`const m=html.match(/const QUESTIONS = (\[[\s\S]*?\n\]);/);eval(m[1]).length` 应为 279
3. **加题方法**：在 QUESTIONS 数组末尾（最后一个 `}` 之后、`];` 之前）插入新对象，格式见 3.1；新分类记得加标签颜色 CSS
4. **部署**：本地验证后：
   ```bash
   cd /Users/qchen/ZCodeProject/interview
   git add index.html && git commit -m "描述" && git push
   ```
   GitHub Pages 用 legacy 模式（main 分支根目录），push 后 1-2 分钟线上生效，无需其他操作
5. **不引入后端**：用户数据只存 localStorage，保持纯静态
6. **手机适配**：改 CSS 时注意 @media(max-width:520px) 断点；代码块已设置 `white-space:pre-wrap` 自动换行（手机不横滑）

## 七、当前未做/可扩展方向

- 搜索功能（按关键词搜题目）
- 错题本（答错自动收集）
- 每日复习计划（艾宾浩斯）
- 深色/浅色主题切换
- 导出/导入学习记录（localStorage 转移到新设备）
- 题目编辑界面（现在加题要直接改 HTML）

## 八、快速上手示例：新增一道题

在 QUESTIONS 数组末尾 `];` 前加：

```javascript
  {c:"场景实战",d:"★★",q:"新题目？",
   a:`<p>普通答案，通俗易懂。</p>
      <blockquote>重点提示。</blockquote>`,
   b:`<h5>📐 八股文详解</h5>
      <p><strong>1. 要点</strong>：分点详解。</p>
      <p><strong>2. 面试标准答法</strong>：一句话总结。</p>`}
```

然后跑第六节的语法验证，通过后 git push 即上线。

## 九、环境信息

- 本地路径：`/Users/qchen/ZCodeProject/interview/index.html`
- gh CLI 已登录（账号 Chen1qi），git push 走 https + gh 凭证
- 测试环境：macOS + Chrome/Safari；用户主要在手机浏览器访问线上地址

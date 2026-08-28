# LLM 越狱 · 领域追踪

一张数据驱动的**立场图**:把 LLM 越狱(jailbreak)这个领域画成签名边网络——谁继承谁、谁佐证谁、谁证伪谁。
姊妹项目是 [`../llm-backdoor-tracker`](../llm-backdoor-tracker)(LLM backdoor 检测),
两者用同一套构建管线与同一套内容纪律,可以对照着读。

> ### 这是什么 / 不是什么
>
> **这是一名二进制安全研究者在自学 LLM 越狱方向时,借助 AI 整理出来的领域索引。**
> 要解决的问题很具体:这个方向三年出了几千篇,散在 arXiv 的 `cs.CR` 与 `cs.CL`、安全顶会、
> 机器学习顶会和厂商博客四个互不相通的地方,一百多篇里谁在反驳谁没人画过。
> 它的用途是**快速检索与分类**——迅速定位「哪条线还在打、该先读哪篇、这篇的对手是谁」,并给出直达原文的链接。
>
> **它不是综述,不应当被当作引用来源。**
>
> - 图上每条 note、每张战线卡的裁定(「已证伪」「已反转」「未决」)都是**编者的概括与判断**,
>   基于摘要或原文整理,**不是原作者的表述,也不代表领域共识**。任何一条要用之前,请点开链接读原文。
> - **「常被误读」标记针对的是这些论文被转述和引用的方式,不是被标注论文本身。**
>   恰恰相反——被标的通常正是这个领域最重要、被引最多的那几篇。
> - **「核验状态」标的是编者是否亲自核对过书目**(标题 / 作者 / v1 日期 / id),**不是**对该论文结论正确性的背书。
> - **整理过程有 AI 参与,出过错。** 建库时凭记忆填的 arXiv id 当场抓出两处错号
>   (详见 [MAINTENANCE.md](MAINTENANCE.md) 的「历史留档」)。

---

## 范围

**收:** 文本 LLM 的越狱——让已对齐模型输出被拒内容。含手工/社群越狱、离散优化攻击、
语义与黑盒攻击、分布偏移与多轮攻击、**微调攻击与开放权重篡改**、推理期与训练期防御、
评测基建、拒答的机制可解释性、理论上界。

**不收(判超范围,记录在此不是漏掉):** prompt injection / 间接注入 / agent 劫持
(威胁模型是突破**指令层级**,不是突破 harmfulness 拒答);多模态越狱(VLM / 语音);
纯应用型 red-teaming 报告。少数横跨条目按其**文本部分**收录。

---

## 一分钟上手

```bash
cd notes/llm-jailbreak-tracker
python3 build.py --open      # 生成 index.html 并打开
```

无依赖,纯标准库。`index.html` 是**单文件**——数据全部内联,双击就能看。

---

## 目录

```
llm-jailbreak-tracker/
├── README.md          ← 本文件
├── MAINTENANCE.md     ← 维护笔记:更新流程 / 图的约定 / 历史留档
├── LICENSE            ← 内容 CC BY 4.0 / 代码 MIT
├── build.py           ← 唯一的构建入口(带自检)
├── spread.py          ← 标签太挤时重排显示坐标(幂等,从 t0 重算)
├── template.html      ← 页面骨架 + CSS + 渲染器,不含数据
├── index.html         ← 生成产物,不要手改
└── data/
    ├── papers.json    ← 条目库(179) ← 主要在这里加东西
    ├── edges.json     ← 签名边(320) ← 和这里
    ├── fronts.json    ← 交火战线卡片(13)
    ├── misread.json   ← 「常被系统性误读」标记(51)
    ├── lanes.json     ← 十一条主干泳道
    └── years.json     ← 时间轴刻度 [年份, t 坐标]
```

**只改 `data/*.json`,然后跑 `build.py`。** 改样式才动 `template.html`。

---

## 现状快照

| | |
|---|---|
| 条目 | 179(174 有 arXiv · 1 只在 NeurIPS 论文集 · 4 博客/标准发布) |
| 边 | 320 —— 继承 141 · **佐证 77** · **反驳 102** |
| 战线 | 13 条交火中 |
| 引用核验 | **154 confirmed · 1 无 arXiv(NeurIPS 论文集,已核对)· 0 待核** |
| 泳道 | 11 条 · 2017–2026 |
| 边数最多 | `gcg` 30 · `refusaldir` 15 · `jailbroken` 14 · `qifinetune` 12 · `hhrlhf` 11 |

**最该看的一个比例:反驳边 89 条,佐证边只有 42 条。** 在一个健康积累的领域里这个比例应该反过来。

---

## 发布状态

**仓库已公开:** https://github.com/flora2627/llm-jailbreak-tracker

**GitHub Pages 尚未开启,也还没有自定义域名。**(姊妹项目的经验:登录的 fine-grained PAT 开 Pages 是 403,
要去网页上点。)在此之前只能 clone 下来本地打开 `index.html`——它是自包含单文件,无外部请求。

克隆要用 flora 那把 key:

```bash
git clone github-flora:flora2627/llm-jailbreak-tracker.git
```

`index.html` 是**产物不是源**:改完 `data/*.json` 必须 `python3 build.py` 再 commit,
否则就是「本地对了、线上还是旧的」。

---

## 许可

| | |
|---|---|
| **内容**(note / 战线卡 / 正文 / 仓库内文档 / 图的结构与呈现) | [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) |
| **代码**(`build.py`、`template.html` 里的 CSS 与 JS) | MIT |
| **书目信息**(标题 / 作者 / 年份 / venue / arXiv id / URL) | 事实,不主张权利 |

论文本身的著作权归各自作者。本项目**只链接原文,不复制论文正文或图表**。详见 [LICENSE](LICENSE)。

# 维护笔记

给维护者(和 AI 助手)看的:怎么加论文、页面怎么组织、图为什么这么画,以及历次更新与订正的留档。
**项目介绍在 [README.md](README.md)。**

姊妹项目 [`../llm-backdoor-tracker`](../llm-backdoor-tracker) 用的是**同一套 `build.py` 与 `template.html`**
(本项目从它复制而来,只改了数据与文案)。**那边的 MAINTENANCE 仍然是这套管线的权威说明**,
本文只写本项目自己的东西与差异。

---

## 每周更新流程

### 1. 扫新论文

**沙箱里 `curl` 出不去(静默返回空),`WebFetch` 走得通。别在 curl 上浪费时间。**
本轮实测:`export.arxiv.org/api/query` 会返回 **429 / socket hang up**(整个出口 IP 被限流,
换 URL 参数也没用,冷却时间以十分钟计);而 **`arxiv.org/abs/<id>` 页面可以并行拉,5 路稳定**,
而且一次就能拿到标题 + 完整作者 + v1 日期 + comments + **完整摘要**——正好是建条目需要的全部字段。
**所以本项目的核验主路径是 `WebFetch` 打 abs 页,不是 API。**

发现新论文用 `WebSearch`(限定 `allowed_domains: ["arxiv.org"]`),它对 2025–2026 的新文覆盖很好;
搜到 id 之后**一律回 abs 页核对**,不要直接采信搜索结果的转述。有效的查询角度:

- 按攻击族:`jailbreak 多轮 / 密码 / 长上下文 / 推理模型 / 微调`
- 按防御族:`guardrail classifier / tamper resistance / latent-space monitoring / unlearning robustness`
- **按「谁打谁」**:`circuit breakers bypass`、`constitutional classifiers broken`、`refusal direction critique`
  —— 这一类最有效,因为本图要的就是反驳边
- 引用追踪:引用了 GCG(2307.15043)、Jailbroken(2307.02483)、Qi 微调(2310.03693)、
  refusal direction(2406.11717)、The Attacker Moves Second(2510.09023)的新文

**按日期窗口穷举(2026-08-28 实测,比关键词搜索更可靠,推荐做主路径):**
`arxiv.org/search/advanced` 支持 URL 直达——`terms-0-term=jailbreak&date-filter_by=date_range
&date-from_date=<起>&date-to_date=<止>&date-date_type=submitted_date&size=50&order=-announced_date_first`,
翻页改 `start=50/100`。一轮拿到「提交日期在窗口内、全文含 jailbreak」的完整清单(一个月约 120–130 条),
再按范围规则人工筛。两个坑:①月度列表页 `arxiv.org/list/cs.CR/2608` **月中 404**,要月末才生成,
穷举请走 advanced search;②`WebSearch` 后端索引对**当月**新文有滞后(会声称 2608 不存在),
不要据此判断「本月没有新论文」——直接去拉列表页。

### 2. 加条目 → `data/papers.json`

字段在姊妹项目基础上多了一个 `t0`
(`id/label/lane/t0/t/title/authors/year/venue/arxiv/url/kind/check/note`)。本项目的差异:

- `lane` 是 **0–10**(十一条),见 `lanes.json`
- **`t0` 用 `spread.py` 里的 `T(year, month)` 算,`t` 交给 `spread.py` 生成**,两者的分工见下面「图的设计约定」
- **`check` 目前全部是 `confirmed`,保持这个状态**。新加的先写 `unchecked`,当轮清干净,别攒
- **`kind` 大多数是 `paper`;机构研究博客标 `blogpost`(图上画成方形)。** 第一个是 SLEIGHT-Bench(Anthropic+Redwood,2026-08-28 收)。

### 3. 加边 → `data/edges.json`

三种类型的用法在本项目里是这样约定的(比姊妹项目更严一点,因为攻防互打特别多):

| `type` | 用在什么情况 |
|---|---|
| `inherit` | 延续同一条方法线或同一个问题陈述 |
| `support` | 独立复现、同向证据、互相印证 |
| `refute` | **攻击打穿防御** 或 **防御把某个攻击的 ASR 打下去** 或 **划出对方结论的上限 / 反转它** |

**方向恒为 `from` → `to`,读作「from 对 to 做了这件事」。**

> **硬规矩:边必须时间正向。** `from` 的 **`t0`** 必须 ≥ `to` 的 `t0`。
> 一篇 2024-02 的论文不可能反驳一篇 2024-08 的论文——这是本轮建库时抓到过 3 处的错误。
> **注意查 `t0` 不是 `t`:**`t` 是经过横向铺开的显示坐标,近同期的两个条目会在 `t` 上互换先后,
> 那是排版而不是错误。校验脚本在下一节。

> **不要加没有边的条目。** `build.py` 会把孤儿节点判为错误并中止。

### 4. 加战线 → `data/fronts.json`

三条硬规矩与姊妹项目相同(`build.py` 会 warn)。本项目额外约定:

- **每条 bullet 都带 arXiv id**,不要只靠粗体名字匹配。本项目的 label 做过缩短(见下),
  粗体名字匹配比姊妹项目更容易挂不上。唯一的例外是 `manyshot`(无 arXiv),它必须写成 **Many-shot** 才能匹配上 label。
- `note` 第一句写 crux,而且本领域 crux 的形式高度固定,基本只有四种:
  **威胁模型不同 / 攻击者预算不同 / 判定口径不同 / 量词位置不同**。写不出属于哪一种,通常说明这条战线还没想清楚。

### 5. 构建 + 自检 + 验

```bash
python3 build.py

# 边的时间方向 + 孤儿 + 重复(build.py 不查时间方向,这条要单独跑)
python3 - <<'EOF'
import json, collections
P=json.load(open("data/papers.json")); E=json.load(open("data/edges.json"))
t={p["id"]:p["t0"] for p in P}          # ← 用 t0(真实日期),不是 t(铺开后的显示坐标)
rev=[(e["from"],e["to"],e["label"]) for e in E if t[e["from"]]<t[e["to"]]]
print("时间倒挂:", rev)
dup=[k for k,v in collections.Counter((e["from"],e["to"],e["type"]) for e in E).items() if v>1]
print("重复边:", dup)
EOF

# 生成的 JS 语法检查
node --check <(python3 -c "import re;s=open('index.html').read();print(re.search(r'<script>(.*)</script>',s,re.S).group(1))")

# 无头截图(#graph / #brief / #fronts / #misread / #refs / #method / #n/gcg 各一张)
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --disable-gpu \
  --virtual-time-budget=5000 --window-size=1440,1100 --screenshot=/tmp/shot.png \
  "file://$PWD/index.html#graph"
```

---

## 图的设计约定(改数据前先看)

- **十一条泳道,顺序是排过的,不要随手改。** 当前顺序:
  `对抗根系 | 对齐训练 | 评测基建 | 离散优化攻击 | 语义/黑盒攻击 | 分布偏移·多轮 | 推理期防御 | 训练期防御 | 微调·权重篡改 | Mech-interp | 理论天花板`
  这个顺序是**在「三条攻击泳道相邻 + 两条防御泳道相邻 + 根系在首、理论在尾」的约束下,
  枚举 9! 种排列、最小化 Σ(边数 × 跨泳道距离)** 选出来的:cost 357,而按叙事直觉排是 412。
  布线算法把每条边放进 `floor((laneA+laneB)/2)` 的通道,**跨度越长,中间泳道的轨道越挤**,
  所以泳道顺序直接决定图有多高。改顺序前先跑一遍这个优化。

- **横轴 `t` 是像素坐标,不是年份。** 对照 `years.json`:2017→200,2022→540,2023→700,2025→1400,2026→1760。
  2023–2025 那段被刻意拉宽(每年约 350 个 t 单位),因为条目密度集中在这里。

- **`t` 上做过一次「有上限的横向铺开」,基准是 `t0`。** 每条 `papers.json` 记录有两个时间坐标:
  **`t0` 是由 `T(year, month)` 算出的真实时间位置**(判定边的先后、写注记时以它为准),
  **`t` 是排版用的显示坐标**。2023-10 那种一个月挤进四篇的情况会让标签叠成 6 层,
  所以 `spread.py` 按标签宽度把同泳道条目推开,**单条相对 `t0` 的位移上限 55 个 t 单位(约两个月),顺序不变**。
  脚本**每次都从 `t0` 重算,因此幂等**——反复跑不会累积漂移。
  代价是**节点横向位置不能用来读精确日期**,近同期的两个条目甚至会在图上互换先后。精确日期在条目卡与资料表里。
  加新条目后如果 `build.py` warn「标签需要 >3 层」,跑一次 `python3 spread.py` 再 build。

- **label 要短。** 本项目做过一轮统一缩短(`Do Anything Now`→`DAN`、`形式化守卫验证`→`守卫验证`、
  `AmpleGCG-Plus`→`AmpleGCG+`…)。中文字符在排版里按 1.0 计宽、拉丁按 0.6,**中文 label 尽量 ≤5 字**。

- 其余(节点大小=边数、琥珀描边=做过反驳、虚线描边=常被误读、正交通道布线)与姊妹项目相同。

---

## 本项目的内容纪律(在姊妹项目那几条之外)

1. **数字必须当场从 abs 页的摘要里抄。** 本项目的 133 条注记全部这样写的。
   摘要里没给的数字**就不写**,不要从论文正文的记忆里补。
2. **ASR 数字必须带上下文。** 本领域跨论文的 ASR 不可比(判定器、变体配置、成功的定义三处都不同)。
   写注记时至少带上模型名或基准名;能带判定器就带。
3. **区分「攻击强」与「危害真」。** 提示越狱要付能力税(见战线 12),权重越狱不付。
   **跨泳道比较 ASR 是本领域最常见的错误**,注记里不要犯。
4. **同名论文要显式区分。** `AutoDAN` 有两篇(2310.04451 遗传 / 2310.15140 梯度),
   两条注记里都写了「与另一篇同名,注意区分」。再遇到同名情况照此办理。
5. **防御条目默认按「未面对自适应攻击」处理。** 这是 2510.09023 立下的口径,
   本图所有防御注记都按它打折。写新防御条目时,如果原文没做自适应评测,注记里要说。

---

## 历史留档

### 2026-08-28 · 复审修剪(当日五轮后)

一天 +52 条目(+39%)对精选图来说过快,按收录纪律做了对抗性复审,裁 5 个「双不沾」条目
(既无会议背书/规模证据,又不是立场承重),-8 边,战线 05/12 各删一弹:

| 裁掉 | 理由 |
|---|---|
| ADVERSA (2603.10068) | **15 组对话**的小样本,SERA 小会;「越狱集中早轮」方向有趣但撑不起战线级结论 |
| ICO (2608.03210) | 语义攻击泳道已 7 个点,它只是「更高 ASR」的技术增量,无新立场 |
| CRG (2608.07892) | 摘要无数字、无会议;LRM 防御位已有 MTK/AdvSafe/SGASA 三点占位 |
| HiRoute (2608.12821) | 无数字无会议,rpo 之上的增量;训练期防御泳道 10 个点后最薄的一个 |
| 共识采样 (2512.24925) | 无会议、陌生团队;理论泳道证书侧已有三个,「构造性一半」不承重 |

保留但明示折扣的同类:dsa/jpu/doubtprobe/overrefrep/advsafe/sgasa(无会议但有形式保证/新机制/具体数字)。
**判断标准写清楚:会议背书不是必要条件,「立场承重 + 具体可查的声明」才是;两者皆无才裁。**
顺手订正:aisec 成本数字统一为论文版 $14,200(此前注记混用博客版 $14,000)。
修剪后 180 条目 / 323 边,边节点比 1.79(建库 1.72,仍在原区间)。

### 2026-08-28 · 第六轮(理论专项 + venue 扫尾:+6 论文 · +11 边)· 五轮更新收尾

**理论泳道 5→8 个点,建库轮的「理论稀少」判断被推翻一半**:不是没有理论文献,是主词枚举扫不到——
理论论文不常说「jailbreak」(过滤不可行性、共识采样)或者被标 WIP(越狱悖论)。新增:
**过滤不可行**(2507.07341,Ball/Goldwasser/Reingold/Rothblum,ICLR 2026——输入/输出两侧的计算不可行性,
理论泳道承重墙)、**越狱悖论**(2406.12702,2024 年漏收:无完美越狱分类器 + 弱不能检强)、
**共识采样**(2512.24925,不可能性之外的构造性一半)。理论泳道现在「不可能性三支(攻击必然/判定受限/过滤不可行)
+ 证书两支 + 构造性一支」,这个结构本身值得写进战线 12 的 note(下轮)。

**venue 扫尾**(S&P/USENIX/CCS 2026):**MetaBreak**(S&P'26,特殊 token 攻击面)、**MTK**(USENIX'26 C2,
自适应攻击下 TPR 85% 的轨迹检测)、**SLM 评测**(CCS'26,59 SLM,脆弱性系于训练细节而非规模)。
aisec 条目从博客升级为已核验论文(2608.03070,FAR AI,Gleave/Pelrine 在列)。
misread +1(过滤不可行的条件性);战线 03/04/07/10/12 加弹。

**本轮起按纪律留给下轮的线索**(未核验,先记 id/线索):
JailbreakScope(USENIX'26 C1,表示层解释框架)、SoK: Evaluating Jailbreak Guardrails(S&P'26)、
GateBreaker(USENIX'26,MoE 门控攻击)、SALLIE(2604.06247)、技能记忆攻击(2605.29237)、
TRACE(2608.15594)、No One Model Catches Every Harm(2608.21775)、扩散 LLM 安全(2608.07430)。
LessWrong/AF 仍不通(JS 站),Persistent Gap。

 + 引用追踪:+6 论文 · +1 标准发布 · +13 边)

R1 只枚举了含「jailbreak」的提交;本轮换相邻词(suffix / refusal direction / unalignment / red team)与
**引用追踪**(沿 Sleight/Perez 团队往回追),补上词表盲区。含 TrustNLP 2026(CLS)、NeurIPS 2024 workshops(窄域防御,建库漏收)。
要点:**Logit 转向**(logit 层干预暴露激活层方法低估的脆弱性,Llama 2 73% vs 22.6%);
**窄域防御**(2412.02159,只禁制弹也难——转录分类器即宪法分类器前身,理论侧「窄域也难」的经验源头);
**推理污染**(2604.15725,往推理步骤注入有害内容、最终答案不变,答案级审查全绿);
**去对齐评测**(2605.17413,把 abliteration 从社区配方变成受控协议:单向量拒答投影安全收益甚微、域外不安全依从大增);
**DoubtProbe / JPU / MLCommons v0.7**。misread +1(CLS 的「一秒 95%」);战线 00/04/06/12 加弹。
本轮小结:**相邻关键词扫出的漏网明显少于主词枚举**——arXiv 主路径(日期窗口+主词)是高效的,值得保持;
引用追踪是补「重要但不用这个词」的论文的最好工具(窄域防御即由此而来)。

:+2 博客 · +4 边)

**收进:**FAR AI「AI Security Leaderboard」(2026-07-29,预算口径的标准化横评,Grok 4.5 全域 $58 /
Gemini 3.1 Pro <$300 vs Fable 5 / GPT-5.6 Sol >$14,000,附 arXiv 2608.03070 未核以博客为准);
**mlabonne「Uncensor any LLM with abliteration」(2024-06-13)——建库待办点名要找的 abliteration 社区原始记录**,
与拒答方向论文同月独立实现(方向→权重正交化),退化后 6h45m DPO 恢复能力的能力税细节入注记,并加 1 条误读标记。
战线 05(pro)/12(con) 各加一弹。

**本轮看过、超范围或不通,记录在此:**
- UK AISI 2026-06~08 五篇(偏好-行为、贝叶斯停时、IRT、多智能体控制、谎言检测)——评测方法论/控制域,均不在 harmfulness 拒答范围。
- Redwood 博客 2026-05~08 集中在 OpenAI/HuggingFace 入侵事件调查与 AI control——agent/控制域,不收。
- **LessWrong / Alignment Forum 两种姿势都不通**:`/tag/*` 返回旧缓存,`/allPosts` 只渲染出导航(JS 站),
  WebFetch 拿不到正文。持续缺口,下轮换 Google site: 检索或人工抄录。
- Apollo Research 连续三次 socket closed,本轮不通;promptfoo LM Security DB 近期条目全是 agent/注入类,不收;
  Gray Swan 本季无立场类内容;HF 博客 2026 上半年无相关帖(唯一相关即上文收录的 2024 abliteration)。
- FAR AI「DeepSeek-V4-Pro 压力测试」(2026-05,低技能攻击者 98–100%)——纯应用型红队报告,按范围规则不收。



第一轮 128 条枚举里的次优先候选补齐(多轮攻击 / 推理期防御 / 表现层评测 / 合并机制 / LRM 训练防御)。
含 ICLR 2026(SEMA)、COLM 2026(LOCA)、SERA 2026 最佳论文(ADVERSA)、COLM AIW(合并塌缩)。
要点:**审计缺口**(2606.08044,静态审计连静态探针一起被打穿,是潜空间监测线的新反证);
**绊线防御**(Tripwire,安全神经元从保护对象改成触发器);**句法敏感**(2608.05409,时态攻击的一般化,
与拒答几何同月给出「训练数据多样性」这个共同硬化杠杆);**合并塌缩**(2607.27240,量级不对称而非方向冲突)。
修正 1 处时间倒挂(LOCA→SALO,LOCA v1 更早)。战线 00/02/04/05/07 各加一弹;misread +1(登门槛)。
本轮仍判超范围:SALLIE / 技能记忆攻击 / TRACE / No-One-Model / 自进化多智能体防御等——与现有节点对话面窄,留观。

(+21 论文 · +1 博客 · +47 边)

覆盖 2026-07-25 → 2026-08-28 的提交窗口(上一轮止于 2607.22929)。枚举用 advanced search 按日期窗口
拿全 128 条「提及 jailbreak」的提交,按范围规则筛掉多模态 / agent / prompt-injection / 应用域后剩约 30,
再按「能否与现有节点构成立场边」收 21 篇;另收第一个 `blogpost` 条目(SLEIGHT-Bench)。

**主题聚类(本轮新增的对话线):**
- **预训练底座成为威胁模型的锚点**:BAJ(2608.26506,EMNLP-F)+ One Leak Away(2512.14751,CCS,同组)
  ——洞在底座,微调与合并都修不掉;与 SkillSafe-Bench(2608.08542)构成「合并安全」三角。
- **「摊开安全信号」防御簇**:NeuronGuard(2608.23959,EMNLP-F)+ DSA(2608.01414)同月同向,
  都直接接「剪枝脆性」诊断;拒答几何(2608.25390)从训练动力学侧解释稀疏从哪来,并给硬化杠杆。
- **内部评分被反序**:评分反序(2608.09624)给潜空间监测线(战线 07)迄今最重反证——
  outcome AUROC 0.220,成功的攻击排到失败的后面。
- **认证鲁棒性进多轮**:MTCR(2608.20820)+ SmoothLLM 概率证书(2511.18721),理论泳道 3→5 个点。
- **口径与预算**:Fair-ASR(2608.17360)给战线 05 加「目标调用预算」维;多轮 SoK(2608.01117)
  把战线 10 的 crux 收敛为「意图组织度」。
- **认知税**:Fool's Gold(2608.17202,Russinovich 单作者)把「权重级攻击不付能力税」改写成
  「付认知税」——打穿与拿到真货脱钩,战线 11/12 各加一弹。

**核验方式:**与建库轮相同,22 条逐条 WebFetch 拉原文页面(abs 页 / 博客页),书目字段与数字当场抄。
其中 7 篇 v1 早于本轮窗口(2025-05 ~ 2026-05,建库轮漏收),t0 一律按 v1 计;LogiBreak(2505.13527)
按 ACL 2026 Findings 正式发表收录。

**本轮看过、判定超范围、故意不入图(在原三条之外):**
- 8 月的 agent / prompt-injection / 工具调用安全一波(When Context Gets Root、Framing Gap、
  TraceSafe、SkillShield、 embodied survey 等)——威胁模型是指令层级,不收。
- 多模态 / T2I / T2V / 音频一批(TempJail ×2、MMJailBench、GuardPaint、COMIC、SafeCA、
  prosody-driven 等)——多模态线,不收。
- Anthropic 研究博客 2026 上半年的其它帖子(Diffuse AI Control、Teaching Claude Why、
  Abstractive Red-Teaming)——对象是 sandbagging / character / 泛化,不是 harmfulness 拒答,不收;
  Gray Swan 2026-08 只有一篇「安全蛇油鉴别指南」,立场图用不上。
- **X / Twitter writeup 无法核验**:x.com 正文登录墙挡住 WebFetch,搜索只有转述没有原文,
  按本项目「不接受未经当场抓取的书目字段」的纪律,**一律不收**。用户点名要的 X 线,
  现状是「抓不到原文」,等有可核验的镜像(nitter 类)再补。



从 `../llm-backdoor-tracker` 复制 `build.py` / `template.html` / `LICENSE` / `.gitignore`,
重写全部数据与文案。产出:133 条目 · 229 边 · 13 战线 · 46 条误读标记 · 11 泳道。

**核验方式:**133 条**逐条 WebFetch 拉 `arxiv.org/abs/<id>`**,核对标题、完整作者、v1 日期、comments 与摘要正文。
唯一没有 arXiv 的是 Many-shot Jailbreaking,拉的是 NeurIPS 2024 论文集页。

**本轮抓到的自身错误(凭记忆填 id 导致,已订正):**

| 想收的论文 | 记成的 id | 那个 id 实际是什么 | 正确 id |
|---|---|---|---|
| Detecting Language Model Attacks with Perplexity | `2308.14539` | 质子交换膜燃料电池气体扩散层两相动力学 | `2308.14132` |
| JailbreakEval | `2408.09321` | Nullnorms on bounded trellises | `2406.09321` |

两处都是**数字位置记错,论文本身真实存在**。这印证了姊妹项目的那条经验:**id 记得住,数字记不住**。
本项目因此不接受任何未经当场抓取的书目字段。

**本轮修过的建模错误:**

- 3 条边**时间倒挂**(`brittle→refusaldir`、`rpo→adaptive`、`safedecoding→shallow`),
  即用早的论文去反驳/继承晚的论文。已分别改成反向 support 或改挂到更早的目标上。
  **`build.py` 不查这个,必须单独跑校验。**
- 5 条抗篡改防御(Vaccine / RepNoise / TAR / Patcher / HarmAlign)**原本放在「微调·权重篡改」泳道**,
  导致该泳道 22 条、标签叠 6 层。改归「训练期防御」——它们确实是训练期防御,只是针对开放权重场景。
- 泳道顺序**原本按叙事直觉排**(根系→对齐→攻击→防御→评测→interp→理论),cost 412;
  按上面那个约束优化后改成当前顺序,cost 357,图高从 3400+ 降到 3075。

### 本轮看过、判定超范围、故意不入图

- **prompt injection / 间接注入 / agent 劫持** —— 威胁模型是突破**指令层级**而非 harmfulness 拒答,
  边的语义与本图不同,混进来会让 `refute` 失去意义。注意 2510.09023 标题里同时有 jailbreaks 与
  prompt injections,**本图只取它的越狱那一半**。
- **多模态越狱(VLM / 语音)** —— 文献量大但与文本线的对话面窄。少数横跨条目
  (2412.03556 Best-of-N、2406.04313 circuit breakers)按其**文本部分**收录,多模态结果不作为本图证据。
- **搜索里出现但无法定位到可核实 arXiv id 的「理论不可能性」转述** —— 本轮专门搜过越狱的理论上界,
  搜索结果里有若干「证明了 X 不可能」的说法,但都追不到具体 id,**一律不收**。
  这是理论泳道只有 3 个点的直接原因之一。

---

## 待办

### 最大的单一缺口

**LessWrong / Alignment Forum 从未系统扫过。**(2026-08-28 试过一次:`/tag/ai-control` 页面
WebFetch 拿到的是数年前的缓存快照,没有当月内容;需要换 allPosts 按时间翻页重试。) 姊妹项目的经验是这个圈子有承重结果只发在那里
(典型:Sleeper Agents 的复现研究)。越狱这边大概率同样如此——尤其是 abliteration 这类
**先在社区流行、后来才被论文化**的技术,它的原始记录几乎肯定在 LW 或 HuggingFace 讨论区。
扫法:LW 的 `AI` / `AI Control` 标签按新排序,以及 alignmentforum.org;
机构博客:alignment.anthropic.com、UK AISI、Gray Swan、FAR AI、Redwood、Apollo。

### 其余

- **按 venue 与实验室检索**(DBLP 逐年翻 CCS / NDSS / USENIX Sec / S&P)。
  本轮以 arXiv 为主入口,纯论文集条目基本没进来。姊妹项目的教训:安全顶会论文九成在 arXiv 上,
  但用的是安全圈词汇,按主题关键词搜不出来。
- **理论泳道已在 2026-08-28 六轮后扩到 8 个点**(BEB / 不可判定 / 可证防御 / MTCR / 平滑证书 /
  过滤不可行 / 越狱悖论 / 共识采样)。建库轮「只有 3 个」的判断源于检索词盲区,已留档。
  值得请一个熟悉这块的人过一遍——**如果确实只有这么多,那本身就是一条值得写出来的观察**。
- **中文与非英文文献未覆盖。**
- **战线 09 指出的空洞:** 「在微调**之后**做行为评测」这条路线目前没有任何一篇论文系统研究过。
  如果找到了,那会是一条重要的新边。

---

## 附:`spread.py`

加新条目后 `build.py` warn「标签需要 >3 层」时跑一次:

```bash
python3 spread.py && python3 build.py
```

它从每条记录的 `t0` 重新计算 `t`,**幂等**(连跑两次 `papers.json` 的 md5 不变)。
新条目必须带 `t0`;`t` 可以先随便填,`spread.py` 会覆盖它。
`t0` 由 `T(year, month)` 按 `years.json` 插值算出——直接复用 `spread.py` 顶部的同名函数。

# replication-audit

**让论文的复现包在任何电脑、任何 AI agent 上跑出同样的结果，并帮你发现数字、样本和图表里的可疑之处。**

面向经济学、管理学及其他实证研究的复现包审计 skill（v0.3，Python 为主，兼有 Stata/R 静态扫描）。
遵循通用的 `SKILL.md` 格式，可用于 Claude、Codex CLI、Gemini CLI、Cursor 等支持 Agent Skills 的工具，也可以不借助任何 agent，直接当命令行脚本用。

> English summary: a standard-library-only toolkit (Python ≥ 3.11) packaged as an Agent Skill. It scans Python/Stata/R code for non-reproducibility, runs a replication package in controlled and perturbed environments, stores a baseline, re-checks results across machines/agents, traces pandas sample-size changes, verifies numbers printed in a paper against outputs, and flags orphan files, AI-generation markers in figure metadata, and mismatches with an AI-use declaration. All pass/fail decisions are made by scripts, not by the LLM. See "Honest limits" below.

---

## 为什么做

AI 让写代码和出图变得很快，也让一类问题变得更隐蔽：代码能跑但结果不对，表里的数字和代码输出对不上，样本在某一步悄悄变少，图表来源说不清。
人工逐项核对既慢又容易漏，而让 AI 自己检查自己，结论又会随模型和环境变化。

本项目的思路是：**让脚本做判定，让 agent 只负责调度和解释。** 判定由固定的、可复核的代码给出，同一个复现包换电脑、换 agent，结论不变。

## 它做什么

| 能力               | 说明                                                                                                                      |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **静态扫描**         | Python / Stata / R：未设随机种子、绝对路径、`merge` 不校验、`m:m`、`except: pass`、结果变量被硬编码、`ssc install` 等共 39 条静态规则（另有样本流、表格自洽、溯源与披露类检查） |
| **基线与重跑**        | 在两个干净副本里各跑一遍，一个受控、一个扰动（不同哈希种子、线程数、时区、含空格的路径）。两次不一致就拒绝保存基线（`UNSTABLE`）                                                   |
| **跨机器核对**        | 把整个文件夹拷到另一台电脑或另一个 agent 上运行 `check`；也可只比对别处产出的结果（`--outputs`）                                                           |
| **容差比对**         | 计数必须完全一致，系数用相对容差；基线里锁定容差，改 `repro.toml` 放宽容差不能让检查过关                                                                     |
| **样本流追踪**        | 记录 pandas 每一步 merge/dropna/query/筛选前后的行数，提示"merge 后行数变多""某步丢了超过 20% 样本"，并要求与基线一致                                        |
| **论文数字核对**       | 在 `repro.toml` 里登记论文中印出来的数字，`check` 会逐一对照新跑出的输出                                                                         |
| **回归表自洽**        | t 是否等于 coef/se，p 值是否小到任何 t 分布都不可能                                                                                       |
| **孤立文件**         | 输出目录里有、但流水线跑完没有重新生成的文件（可能是手工放入或旧版本残留）                                                                                   |
| **图像元数据**        | 检查 PNG/PDF/SVG/JPEG 元数据中是否有 AI 生成标记、是否由 matplotlib 等绘图软件产生                                                              |
| **AI 披露核对（启发式）** | 对照 `[disclosure]` 声明与代码文本中的 AI 提及、占位符残留、git 提交里的 AI 共同作者署名                                                              |
| **报告**           | 同时输出 `repro_report.md`（给人看）和 `repro_report.json`（给程序读）                                                                  |

## 快速开始（3 分钟，不需要自己的数据）

需要 Python ≥ 3.11；跑 demo 还需要 `numpy pandas matplotlib`。工具本身只用标准库。

```bash
git clone <本仓库地址>
cd replication-audit-skill

# 生成三个示例复现包：good / bad / flow
python replication-audit/tests/make_demo.py /tmp/demo

# 1) 扫描一个"有问题"的包
python replication-audit/scripts/audit.py scan /tmp/demo/bad

# 2) 未设随机种子 -> 基线被拒绝
python replication-audit/scripts/audit.py baseline /tmp/demo/bad      # 退出码 2：UNSTABLE

# 3) 稳定的包 -> 保存基线，并列出样本流提示
python replication-audit/scripts/audit.py baseline /tmp/demo/flow
#   结论：BASELINE_SAVED ... 记录了 2 个样本流步骤。
#   ? TR001 第 1 步 run_all.py:16: merge (left) increased rows 400 -> 401: duplicate keys on the right side?
#   ? TR002 第 2 步 run_all.py:17: filter_mask dropped 31% of rows (401 -> 277).

# 4) 重跑并与基线比对
python replication-audit/scripts/audit.py check /tmp/demo/flow
```

## 用在你自己的复现包上

```bash
python replication-audit/scripts/audit.py init       ROOT   # 生成 repro.toml 草稿，请逐行核对
python replication-audit/scripts/audit.py scan       ROOT   # 先修 error，再讨论 warn
python replication-audit/scripts/audit.py baseline   ROOT   # 两次受控运行，稳定才保存基线
python replication-audit/scripts/audit.py check      ROOT   # 以后每次改动、换电脑、换 agent 都跑
python replication-audit/scripts/audit.py provenance ROOT --with-run   # 图像元数据、孤立文件、披露核对（只供人工复核）
```

`repro.toml` 的写法见 [`replication-audit/references/manifest.md`](replication-audit/references/manifest.md)，规则说明见 [`references/rules.md`](replication-audit/references/rules.md)，中文上手见 [`references/quickstart_zh.md`](replication-audit/references/quickstart_zh.md)。

在另一台电脑或另一个 agent 上验证：把整个文件夹（含 `repro_baseline/`）拷过去，运行同一条 `check`。

退出码：`0` 通过 · `1` 失败 · `2` 不稳定 · `3` 设置错误。

## 作为 Agent Skill 安装

`replication-audit/` 文件夹就是一个标准 skill（`SKILL.md` + `scripts/` + `references/`）。可以直接放进你所用 agent 的 skills 目录，或运行：

```bash
python replication-audit/install.py --target claude        # ~/.claude/skills
python replication-audit/install.py --target agents        # ~/.agents/skills
python replication-audit/install.py --target antigravity   # ~/.gemini/antigravity/skills
python replication-audit/install.py --dest /your/skills/dir
```

各 agent 的 skills 目录位置可能随版本变化，请以其文档为准。装好后，对 agent 说"帮我审计这个复现包"即可，skill 会引导它按固定流程调用脚本，并规定它**不得**为了让检查通过去改基线、容差或你的分析代码。

## 设计原则

1. **判定由脚本给出。** agent 只调度、转述、帮你修原因；报告由脚本生成。
2. **基线不可被"商量"。** `check` 使用基线里保存的容差；放宽 `repro.toml` 没用。
3. **证据不是指控。** 所有发现都是线索，`provenance` 永远不会判 `FAIL`，只产出"需人工复核"清单。
4. **只用标准库。** 工具本身零依赖，换环境不出问题。
5. **诚实标注能力边界。** 见下节。

## 诚实的能力边界（请务必阅读）

- **只用自带 demo 验证过。** 31 项自测全部通过，覆盖未设种子、集合遍历顺序、静默缩小样本、输入数据被改、放宽容差、merge 行数增加、孤立文件、AI 元数据标记、披露不一致、t/p 不自洽、论文数字不符等情形。**还没有在真实论文的复现包上系统测试过。**
- **Stata 和 R 的运行路径未实测**（开发环境里没有这两个软件）。它们的静态规则在文本样例上测过；运行器与语言无关，但需要你在本机验证。
- **样本流追踪只支持 pandas**，且仅记录项目自身代码里的直接调用。
- **图像元数据很容易被抹掉**，没发现标记不等于不是 AI 生成。本工具不做"从像素判断 AI 图"。
- **AI 披露核对是弱证据**：文本里提到 ChatGPT 可能只是引用；AI 参与也常常不留痕迹。
- **通过审计只说明复现包能复现自己的基线**，不说明分析本身正确，也不说明研究设计合理。
- 论文数字核对（`[[claims]]`）需要作者自己登记数字；**从 PDF 自动提取表格并核对的模块尚未实现**（见路线图）。

## 路线图

- [ ] 论文数字审计模块：从 PDF/LaTeX 提取表格，自动检查均值-标准差-范围可行性、星号与 t 值、R² 增量与系数、正文与表格逐项核对
- [ ] Stata / R 的样本量追踪（解析 log）
- [ ] 在多个真实公开复现包上测试并公布结果
- [ ] 受限数据场景（数据不能外传时）的离线核对模式
- [ ] 更多规则：Python 之外的随机性来源（torch/tensorflow）、面板数据常见陷阱等

## 参与贡献

最需要的是**真实案例**（请脱敏）：哪类错误出现了、现有规则能否抓到。抓不到的，我们会补成新规则并加进测试。详见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 免责声明

本工具提供的是**检查线索，不是对学术不端的认定**。任何异常都可能有排版、版本混用、环境差异等正常解释，应通过作者、编辑部或所在机构的正当渠道核实。请勿仅凭本工具输出公开指控他人。

## 许可

MIT，见 [LICENSE](LICENSE)。

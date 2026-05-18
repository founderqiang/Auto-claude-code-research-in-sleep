# ARIS — Autonomous Research via Adversarial Multi-Agent Collaboration

> **Let Claude Code do research while you sleep.** Wake up to find your paper scored, weaknesses identified, experiments run, and narrative rewritten — autonomously. Repo: [github.com/wanshuiyin/Auto-claude-code-research-in-sleep](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep).

## TL;DR

ARIS is a collection of **74 composable Claude Code skills** that orchestrate **cross-model collaboration**: Claude Code drives the research (reads files, writes code, deploys experiments) while an external LLM (GPT-5.4 / 5.5 via [Codex MCP](https://github.com/openai/codex)) acts as a critical reviewer. The two models disagree, debate, and force each other to do better — adversarial, not self-play.

Six workflows compose into a full research lifecycle: idea discovery → experiments → auto-review → paper writing → rebuttal → resubmit. Tested end-to-end on real ICLR/NeurIPS submissions. Score progression on a real overnight run: **5/10 → 7.5/10 with 20+ GPU experiments**.

> 💡 **The ARIS bet.** Markdown is for writers. HTML is for readers. Every workflow artifact stays in Markdown (auditable, machine-parseable, future-proof). When a human needs to actually *read* one, [`/render-html`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/render-html/SKILL.md) produces this view — gated by a fresh cross-model Codex review (the same ARIS invariant every other audit-class skill follows).

---

## The Problem

Every ML researcher reading this knows the rhythm:

1. Spend 3 weeks reading 80 papers.
2. Brainstorm 12 ideas, kill 9 in your head, fail to validate the remaining 3 quickly.
3. Pick one, lose a week to a bug, miss the GPU window.
4. Submit, get a 5/10 review with "lacks ablation against XYZ".
5. Rebuttal week is 72 hours; you have 2 days of teaching duty.

The bottleneck isn't ideas. It's the **end-to-end orchestration** between literature → ideation → experiments → writing → rebuttal. AI can compress every individual step, but the integration is fragile — and worse, a single model reviewing its own work falls into local minima.

> 🚨 **Why not self-play with one model?** Using Claude Code subagents (or any homogeneous agent team) for both *execution* and *review* tends to fall into local minima — the same model reviewing its own patterns creates blind spots. ARIS forces cross-family disagreement: Claude executes, GPT reviews. They don't share lineage, they don't share training data, they don't share blind spots.

---

## Core Architecture

The system is, in one sentence:

$$
\text{Research} = \arg\max_{\theta}\; \mathbb{E}_{x \sim \mathcal{D}_{\text{ideas}}}\bigl[\, U_{\text{exec}}(\theta\,;\,x) - \lambda \cdot R_{\text{review}}(\theta\,;\,x) \,\bigr]
$$

where $U_{\text{exec}}$ is the utility of an executor model writing code / running experiments, and $R_{\text{review}}$ is an *adversarial regularizer* from a cross-family reviewer that penalizes overclaims, fabricated citations, unjustified theorem extensions, and self-flattery. The regularizer is **non-differentiable** — it's a fresh LLM thread reading the artifact cold.

### The reviewer-independence protocol

Every review round uses a **fresh codex thread**. We never use `codex-reply` to continue a previous review conversation. This is a hard rule, learned from a real NeurIPS run where `codex-reply` chains inflated scores from 3/10 → 8/10 through narrative accumulation (the reviewer started defending its earlier criticism instead of evaluating the current artifact). The protocol is codified at [`skills/shared-references/reviewer-independence.md`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/shared-references/reviewer-independence.md).

```
┌──────────────────────┐
│  ARIS — execution    │     ┌───────────────────────┐
│  (Claude Code)       │────▶│  Codex MCP (GPT-5.5)  │
│  — reads files       │     │  — reads paper cold   │
│  — writes code       │     │  — fresh thread       │
│  — deploys to GPU    │     │  — scores 1-10        │
└──────────────────────┘     │  — suggests fixes     │
         ▲                   └───────────────────────┘
         │                              │
         │                              ▼
         │              ┌─────────────────────────┐
         └──────────────│  weakness list (.md)    │
                        │  fix list (with budget) │
                        └─────────────────────────┘
```

> 🔒 **Cross-family invariant.** The executor and reviewer **must** be different model families (Claude × GPT, GLM × DeepSeek, Antigravity × Gemini, …). Same-family review is a non-feature; if you only have one provider, the cheapest fix is to add a free DeepSeek or Gemini reviewer via [`llm-chat` MCP](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/tree/main/mcp-servers/llm-chat).

---

## The Six Workflows

| W | Name | One-line summary | Entry point |
|:--:|------|------------------|-------------|
| **1** | Idea Discovery | Literature → brainstorm 8-12 → novelty check → pilot 2-3 on GPU → ranked report | [`/idea-discovery`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/idea-discovery/SKILL.md) |
| **1.5** | Experiment Bridge | Plan → implement → GPT-5.4 code review → sanity check → deploy → collect | [`/experiment-bridge`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/experiment-bridge/SKILL.md) |
| **2** | Auto Review Loop | Review → fix → re-run → repeat until score ≥ 6/10 (or `MAX_ROUNDS=4` hit) | [`/auto-review-loop`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/auto-review-loop/SKILL.md) |
| **3** | Paper Writing | Narrative → outline → figures → LaTeX → PDF → 2 rounds review (4 → 8.5/10) | [`/paper-writing`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/paper-writing/SKILL.md) |
| **4** | Rebuttal | Parse reviews → strategy → optional experiments → draft → stress test | [`/rebuttal`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/rebuttal/SKILL.md) |
| **5** | Resubmit | Port paper to a new venue under hard constraints (no new exps, no bib edits) | [`/resubmit-pipeline`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/resubmit-pipeline/SKILL.md) |
| **6** | Conference Talk | Paper → Beamer + PPTX + speaker notes + assurance audits | [`/paper-talk`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/paper-talk/SKILL.md) |

### Workflow 2 — Auto Review Loop in detail

The most interesting one. Run it the night before a deadline; wake up to a polished paper.

1. 🔍 **Deep review** — GPT-5.4 xhigh reviews the paper, identifies weaknesses
2. 🩹 **Fix** — Claude implements the fixes (rewrite, add baselines, run experiments); skips any experiment > 4 GPU-hours, flags for manual follow-up
3. 📊 **Re-evaluate** — collect results, update paper, feed back to the reviewer
4. 🔁 **Repeat** — until score ≥ `POSITIVE_THRESHOLD` (default 6/10) or `MAX_ROUNDS` (default 4); if the context window fills mid-loop, auto-resume from `REVIEW_STATE.json`

```bash
/auto-review-loop "focus on Section 3-5, our CRF results are weak" \
    --- difficulty: nightmare \
    --- effort: max
```

The `difficulty: nightmare` flag lets GPT-5.4 read your repo directly via `codex exec` — Claude can't filter what it sees. Maximum stress test before submission.

<details>
<summary><b>Key safety features (click to expand)</b></summary>

- 🔒 **`MAX_ROUNDS = 4`** — prevents infinite loops; stops early if score threshold is met
- ⏱️ **> 4 GPU-hour experiments skipped** — flagged for manual follow-up, never silently launched
- 🧠 **Prefer reframing over new experiments** — when both can address a weakness, picks the cheaper path
- 🪞 **No hiding weaknesses** — explicit rule: "Do NOT hide weaknesses to game a positive score"
- 🔧 **Fix before re-review** — must implement fixes before resubmitting; no empty promises
- 💾 **Compact recovery** — persists `REVIEW_STATE.json` each round; auto-resumes if context window fills

</details>

---

## Real Results

A real overnight 4-round run on an ML research project, from borderline reject to submission-ready:

| Round | Score | Key change |
|------:|:-----:|------------|
|   0   | 5/10  | Baseline narrative + figures |
|   1   | 6.5/10 | Fixed assumption-model mismatch, softened claims |
|   2   | 6.8/10 | Added synthetic validation; tightened limitations |
|   3   | 7.0/10 | Theorem self-contained; renamed conflicting notation |
|   4   | **7.5/10** | Format pass; passed page check; ICLR-compliant |

**Final**: 8 pages main body (ICLR limit: 9), 0 overfull `\hbox`, ICLR-compliant. **+2.5 points across 4 rounds.**

> ✅ **Reproducibility caveat.** Score values from GPT-5.4 are *signals*, not ground truth. ARIS iterates against them, so high AI-review scores are an expected outcome of the loop, not independent proof of acceptance. Human reviewers still bring updated literature knowledge and venue taste an AI reviewer doesn't model.

---

## The 74 Skills

Grouped by role (full catalog: [`docs/SKILLS_CATALOG.md`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/docs/SKILLS_CATALOG.md)).

| Category | Count | Headliners |
|----------|:----:|-----------|
| Literature & ideation | 9 | [`/research-lit`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/research-lit/SKILL.md), [`/idea-creator`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/idea-creator/SKILL.md), [`/novelty-check`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/novelty-check/SKILL.md), [`/deepxiv`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/deepxiv/SKILL.md), [`/arxiv`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/arxiv/SKILL.md) |
| Experiments | 7 | [`/experiment-bridge`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/experiment-bridge/SKILL.md), [`/run-experiment`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/run-experiment/SKILL.md), [`/monitor-experiment`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/monitor-experiment/SKILL.md), [`/experiment-audit`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/experiment-audit/SKILL.md) |
| Paper writing | 12 | [`/paper-plan`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/paper-plan/SKILL.md), [`/paper-figure`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/paper-figure/SKILL.md), [`/paper-write`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/paper-write/SKILL.md), [`/paper-compile`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/paper-compile/SKILL.md), [`/auto-paper-improvement-loop`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/auto-paper-improvement-loop/SKILL.md) |
| Audits | 5 | [`/proof-checker`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/proof-checker/SKILL.md), [`/paper-claim-audit`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/paper-claim-audit/SKILL.md), [`/citation-audit`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/citation-audit/SKILL.md), [`/result-to-claim`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/result-to-claim/SKILL.md), [`/kill-argument`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/kill-argument/SKILL.md) |
| Talks & posters | 4 | [`/paper-talk`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/paper-talk/SKILL.md), [`/paper-slides`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/paper-slides/SKILL.md), [`/paper-poster`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/paper-poster/SKILL.md), [`/slides-polish`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/slides-polish/SKILL.md) |
| Wiki & meta | 6 | [`/research-wiki`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/research-wiki/SKILL.md), [`/meta-optimize`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/meta-optimize/SKILL.md), [`/research-pipeline`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/research-pipeline/SKILL.md), [`/research-refine`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/research-refine/SKILL.md) |
| Integrations & support | 31 | [`/feishu-notify`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/feishu-notify/SKILL.md), [`/figure-spec`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/figure-spec/SKILL.md), [`/render-html`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/render-html/SKILL.md), [`/overleaf-sync`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/overleaf-sync/SKILL.md) … |

### The 3-layer audit chain

A core ARIS invariant: **the executor must not judge its own integrity**. Three layers of cross-model audit:

| Layer | Skill | Asks | When |
|:----:|-------|------|------|
| 1 | [`/experiment-audit`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/experiment-audit/SKILL.md) | "Is the eval code honest? (no fake GT, no self-normalized scores, no phantom results)" | Before / after experiment runs |
| 2 | [`/result-to-claim`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/result-to-claim/SKILL.md) | "Does the claim scientifically follow from the result?" | After results, before writing |
| 3 | [`/paper-claim-audit`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/paper-claim-audit/SKILL.md) | "Does the paper *report* the numbers truthfully?" (fresh zero-context reviewer) | Before submission |

Plus [`/citation-audit`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/citation-audit/SKILL.md) (4th layer): every `\cite{...}` validated for existence, metadata, **and** context-appropriateness — the most diagnostic check ("does the cited paper actually establish this claim?"). And [`/kill-argument`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/kill-argument/SKILL.md) (5th layer): two fresh codex 5.5 + xhigh threads write the strongest 200-word rejection memo and an independent adjudicator pass before submission.

---

## Cross-platform Support

ARIS skills are plain `SKILL.md` files. They run anywhere an agent reads markdown:

- 🤖 **[Claude Code](https://docs.anthropic.com/en/docs/claude-code)** — the default, most tested
- 🤖 **[Codex CLI](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/tree/main/skills/skills-codex)** — full skill mirror; `spawn_agent` instead of `mcp__codex__codex`
- 🖱️ **[Cursor](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/docs/CURSOR_ADAPTATION.md)** — agent mode reads ARIS skills directly
- 🖥️ **[Trae](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/docs/TRAE_ARIS_RUNBOOK_EN.md)** — ByteDance AI IDE
- 🚀 **[Antigravity](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/docs/ANTIGRAVITY_ADAPTATION.md)** — Google's agent-first IDE, native SKILL.md
- 🐙 **[GitHub Copilot CLI](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/docs/COPILOT_CLI_ADAPTATION.md)** — terminal agent, native SKILL.md
- 🐾 **[OpenClaw](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/docs/OPENCLAW_ADAPTATION.md)** — without Claude Code slash skills

> 📝 **ARIS is a methodology, not a platform.** Fork it, rewrite it, adapt it to your stack. The only invariants are: cross-family review, fresh threads for reviewers, audit integrity. Everything else — model choice, install path, integration surface — is yours.

---

## 中文版速览

ARIS（**A**utonomous **R**esearch via Adversarial **M**ulti-Agent Collaboration，**梦中科研**）是一组 74 个可组合的 Claude Code skills，编排**跨模型对抗式协作**：

- **执行**：Claude Code 读文件、写代码、跑实验、改论文
- **审稿**：GPT-5.4/5.5（via [Codex MCP](https://github.com/openai/codex)）以**跨家族**审稿人身份打分、找弱点、提建议
- **关键**：每轮 review 用新 thread；执行者绝不审判自己的实验诚实度

六大工作流端到端贯通：找 idea → 跑实验 → 自动审稿循环 → 写论文 → 写 rebuttal → 跨 venue 移植。在真实 ICLR/NeurIPS 投稿上验证过。

> 🆕 **新加入的 skill**：[`/render-html`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/skills/render-html/SKILL.md) —— 把任何 ARIS 产出的 MD（如 `IDEA_REPORT.md`、`AUTO_REVIEW.md`、`KILL_ARGUMENT.md`）渲染成单文件 HTML，适合给人类读。Markdown 仍是 canonical source，HTML 是 generated view，永远嵌入源 SHA256 + 渲染时间戳防 drift。**academic 模板默认走跨模型 Codex review gate**——同样的 ARIS 不变量。

---

## Get Started

```bash
# 1. Clone ARIS to a stable location (once)
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git ~/aris_repo

# 2. Attach to a project (creates project-local symlinks)
cd ~/your-paper-project
bash ~/aris_repo/tools/install_aris.sh

# 3. Configure the GPT-5.4 reviewer (Codex MCP)
npm install -g @openai/codex
codex setup                                    # pick gpt-5.5 when asked
claude mcp add codex -s user -- codex mcp-server

# 4. Use in Claude Code
claude
> /research-pipeline "factorized gap in discrete diffusion LMs"
```

> ⚙️ **Alternative model combinations** — no Claude or OpenAI API required. See the [Alt routes](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep#-alternative-model-combinations) (Alt B/E for GLM × MiniMax-M2.7 or free DeepSeek-V3.1 via ModelScope; nine routes total, including Antigravity-as-executor and Gemini-direct-API-as-reviewer).

---

## Inspirations

- 🧪 [AI Scientist](https://github.com/SakanaAI/AI-Scientist) (Sakana AI) — automated research pioneer
- 📖 [AutoResearch](https://github.com/karpathy/autoresearch) (Karpathy) — end-to-end research automation
- 🔭 [FARS](https://analemma.ai/blog/introducing-fars/) (Analemma) — fully automated research system
- 🎨 [PaperBanana](https://github.com/dwzhu-pku/PaperBanana) (PKU) — multi-agent academic illustration framework

---

## Community

| | |
|---|---|
| 💬 Group | [WeChat group QR](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/docs/wechat_group.jpg) (refreshes weekly) |
| 🌟 Star | [github.com/wanshuiyin/Auto-claude-code-research-in-sleep](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep) |
| 📖 Technical report | [arXiv 2605.03042](https://huggingface.co/papers/2605.03042) |
| 📑 Skills catalog | [`docs/SKILLS_CATALOG.md`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/docs/SKILLS_CATALOG.md) |
| 🐛 Bugs / requests | [GitHub Issues](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/issues) |

> 🌱 **From idea to paper to podium — one toolchain.** ARIS is a methodology, not a platform. Take it wherever you go.

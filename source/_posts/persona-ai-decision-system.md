
---
title: "Persona-Driven AI Decision System: A Hybrid Local-Agent + LLM Workflow"
date: 2026-03-22 12:00:00
tags: [Agent, Multi-Agent, LLM, DeepSeek, Streamlit, Decision-System]
categories: [AI_Engineering]
mathjax: false
comments: true
top_img: /img/persona-ai-decision-system/bg.jpg
---

Project Source Code: 🚀 GitHub Repository: https://github.com/djdj-student/persona-ai-decision-system



## Abstract

Most LLM applications are optimized for *fluency*: you ask a question, the model returns a coherent paragraph, and the interaction ends. That interaction pattern is fine for information retrieval or drafting text, but it is insufficient for **decision-making**, where the user needs to understand *why* a recommendation was made, *what alternatives were considered*, and *how stable the recommendation is under disagreement*.

This post describes a **persona-driven decision system** that treats a decision as a *multi-stage pipeline* rather than a single model response. The core idea is simple: force the system to generate and preserve **inspectable intermediate artifacts** (structured verdicts, confidence, risk, rebuttals, evaluation scores), then aggregate those artifacts into a final recommendation together with a measurable **conflict index**.

Because the full experiment code is not included in this blog workspace snapshot, I will present a **reference implementation sketch** (data contracts + pseudocode) that is faithful to the system design. The focus is on *engineering decisions*, reproducible structures, and how each stage reduces a specific failure mode.

## 1. Introduction: Why a “Decision System” (Not a Chatbot)

Decisions are a different class of problem than “answer my question.” For a decision to be useful in practice, it must provide:

1) **Traceability**: the ability to audit the chain of reasoning and identify which factors shaped the final verdict.

2) **Diversity with structure**: if multiple personas exist, they should disagree due to *different objectives and risk preferences*, not merely due to different writing style.

3) **Self-checking and conflict resolution**: real decisions involve uncertainty, missing information, and inconsistent arguments; a system should detect these issues and either repair them or surface them clearly.

In other words, the goal is not “one perfect answer.” The goal is a **decision report**: a compact set of structured artifacts that can be inspected, challenged, compared, and aggregated.

### 1.1 Pipeline Overview (Stages 1 → 5)

The workflow is organized as:

- **Stage 1 (Local)**: each persona produces an initial structured verdict without relying on an LLM.
- **Stage 2 (LLM)**: an LLM acts as a validator/auditor of persona consistency and logical gaps.
- **Stage 3 (Local)**: multi-round reflection modifies confidence and verdict based on explicit counter-arguments.
- **Stage 4 (Local/Hybrid)**: persona debate generates targeted rebuttals using claim extraction.
- **Stage 4.5 (Judge)**: an evaluation layer scores the quality of persona reasoning and debate.
- **Stage 5 (Aggregation)**: weighted synthesis produces a final recommendation and a conflict index.

The key design constraint is that every stage must output *structured* fields (not just free-form text) so that later stages can compute with them.

## 2. Data Contracts: The “Decision Record” as the System Backbone

Before we talk about prompts or UI, the most important engineering choice is: **what is the canonical record of a decision?** If the system cannot represent a decision as data, it cannot be audited or aggregated.

Below is a minimal reference schema (Python-style dataclasses) that captures the essential artifacts. This is an intentionally small contract—anything not in the contract is “non-auditable.”

```python
from __future__ import annotations

from dataclasses import dataclass, field
from typing import Literal, List, Dict, Optional

Verdict = Literal["DO", "DO_NOT"]


@dataclass
class PersonaConfig:
	name: str
	risk: float          # 0.0 (risk-averse) → 1.0 (risk-seeking)
	emotion: float       # 0.0 → 1.0
	logic: float         # 0.0 → 1.0
	worldview: str       # short descriptor (used for claim extraction priors)
	decision_style: str  # e.g., "utilitarian", "deontological", "hedonic"
	tone: str            # used only for surface rendering


@dataclass
class DecisionRecord:
	persona: str
	verdict: Verdict
	confidence: int              # 1..10
	weight: float                # 0.1..1.0 (used in aggregation)
	risk_score: int              # 1..10 (question risk as perceived by persona)
	reasoning: str               # auditable rationale (not a prompt)
	gaps: List[str] = field(default_factory=list)
	adjustments: List[str] = field(default_factory=list)
	reflection_notes: List[str] = field(default_factory=list)


@dataclass
class DebateTurn:
	speaker: str
	target: str
	claim: str
	rebuttal: str
	evidence: Optional[str] = None


@dataclass
class JudgeScore:
	persona: str
	logic: float      # 0..10
	realism: float    # 0..10
	bias: float       # 0..10 (higher = more biased)
	confidence: int   # 1..10

	@property
	def composite(self) -> float:
		# Same structure as the blog UI: reward logic/realism, penalize bias.
		return 3 * self.logic + 3 * self.realism + 3 * (10 - self.bias) + min(self.confidence, 10)
```

This contract does three important things:

- It separates **verdict** (“DO/DO_NOT”) from **reasoning** (explainability) and **confidence** (uncertainty).
- It creates explicit fields for **gaps** and **adjustments**, enabling Stage 2 to modify the record without rewriting the entire rationale.
- It provides a consistent interface for evaluation and aggregation (Stage 4.5 and Stage 5).

## 3. Stage-by-Stage Method (With Failure Modes)

This section reads like a method section in a paper: each stage exists because it addresses a known failure mode.

Before diving into the stages, it helps to see how the system is consumed. The UI is intentionally minimal: it is not a “chat window,” but a *report viewer* for a multi-stage run. What matters is that the interface exposes intermediate artifacts as collapsible, persona-scoped objects so that a user can audit the decision without reading raw logs.

![Streamlit entry UI: the single input point for a decision question.](/img/persona-ai-decision-system/Persona_AI_Entry_UI.png)

The design idea behind this screen is that a decision system should feel like running an experiment: you submit one question as the input variable, then the system generates a reproducible trace of outputs. The UI is therefore organized around stages rather than turns of conversation. This prevents a common trap in LLM products: users confuse “a confident paragraph” with “a reliable decision.” By forcing the output to be presented as a staged trace, the interface nudges the reader to inspect assumptions (risk score, confidence, gaps, rebuttals) instead of taking the final recommendation at face value.

### 3.1 Stage 1 — Local Persona Decision (Anti-Convergence)

**Failure mode addressed:** when all reasoning is outsourced to an LLM, multiple personas collapse into the same conclusion with stylistic differences.

Stage 1 forces each persona to produce a verdict using deterministic, parameter-driven heuristics. The goal is not to be “perfect,” but to be *diverse and inspectable*.

![Stage 1 initial decisions list: per-persona verdict, confidence, weight, and risk score.](/img/persona-ai-decision-system/Persona_Initial_Decisions_List.png)

This figure is more than a UI snapshot; it encodes the system’s central engineering discipline: every persona must commit to a *structured* initial record. Notice how the output is not “a paragraph,” but a compact tuple of fields: verdict, confidence, weight, and risk score. This structure is what enables downstream computation. Confidence and weight are particularly important because they allow aggregation to respect *uncertainty* and *persona reliability* rather than treating every vote as equal. Risk score anchors reasoning to the perceived stakes of the question, which later stages can challenge and recalibrate.

In practice, Stage 1 is where you want stable diversity. If two personas differ in risk preference or decision style, they should consistently diverge across runs on the same question. That stability turns personas into testable modules. Without it, “persona differences” become prompt noise: hard to debug, hard to reproduce, and easy to overfit.

Reference logic (sketch):

```python
RISK_KEYWORDS = {
	"finance": ["invest", "loan", "mortgage", "crypto"],
	"career": ["quit", "resign", "job", "offer"],
	"health": ["surgery", "treatment", "diagnosis"],
}


def estimate_question_risk(question: str) -> int:
	hits = 0
	q = question.lower()
	for _, words in RISK_KEYWORDS.items():
		hits += sum(1 for w in words if w in q)
	# Simple clamp to 1..10
	return max(1, min(10, 1 + hits * 2))


def persona_score(persona: PersonaConfig, question_risk: int) -> float:
	# risk preference vs perceived risk
	# higher risk preference should counteract high question risk
	base = 5.0
	base += (persona.logic - 0.5) * 2.0
	base += (persona.risk - 0.5) * (11 - question_risk) * 0.6
	base -= (persona.emotion - 0.5) * question_risk * 0.4
	return base


def stage1_decide(persona: PersonaConfig, question: str) -> DecisionRecord:
	risk = estimate_question_risk(question)
	s = persona_score(persona, risk)
	verdict: Verdict = "DO" if s >= 5.0 else "DO_NOT"
	confidence = int(max(1, min(10, round(abs(s - 5.0) * 2 + 4))))
	weight = max(0.1, min(1.0, 0.4 + persona.logic * 0.4 + persona.risk * 0.2))
	reasoning = f"Risk={risk}. Score={s:.2f}. Style={persona.decision_style}."
	return DecisionRecord(
		persona=persona.name,
		verdict=verdict,
		confidence=confidence,
		weight=weight,
		risk_score=risk,
		reasoning=reasoning,
	)
```

![Reasoning details: expandable local rationales per persona (auditable artifacts).](/img/persona-ai-decision-system/Persona_Reasoning_Details.png)

This figure illustrates the second half of Stage 1: a decision record is not only the scalar fields, but also the audit-friendly narrative that explains them. The key is that the narrative is produced by a local, persona-conditioned generator (or a structured template), not by an LLM improvisation. That choice is intentional: if you want explainability, you must avoid explanations that drift across runs. The reasoning panel is designed to be inspected like an error report: you can read a persona’s justification, check whether it matches the risk score, and decide whether the persona is missing an obvious constraint.

It also prevents “empty rationalization.” The reasoning is expected to reference computed artifacts (risk score, score threshold, confidence) rather than merely sounding plausible. If a persona says “high risk” but the risk score is low, the mismatch is visible and fixable at the heuristic layer.

Even though this is “simple,” it is powerful for two reasons:

- It produces **numerical artifacts** (risk score, confidence, weight) that are composable later.
- It creates *meaningful disagreement* grounded in parameters, not in prompts.

### 3.2 Stage 2 — LLM as Validator (Auditing, Not Thinking)

**Failure mode addressed:** deterministic heuristics can be brittle; a persona can produce a verdict that is logically inconsistent or blind to obvious considerations.

Stage 2 uses an LLM as an **auditor**. The LLM does not replace Stage 1; it critiques it under a strict response contract. This contract is crucial, because the system needs to parse, store, and later compare the validation output.

![Stage 2 LLM consistency check: the validator returns structured gaps and adjustments.](/img/persona-ai-decision-system/Stage2_LLM_Consistency_Check.png)

This figure captures the most important design choice of the entire hybrid system: role separation. The LLM is not asked to “decide.” It is asked to inspect an existing decision record and identify two categories of defects: persona inconsistency (the rationale does not match the persona profile) and logical gaps (missing constraints, unsupported leaps, ignored alternatives). The output is constrained to a small contract, which allows the system to store the LLM’s contribution as deltas (gaps and adjustments) rather than overwriting the original record.

This matters because overwriting destroys attribution and reproducibility. If Stage 2 rewrites everything, you can no longer tell which stage introduced which assumption, and you cannot do post-mortem analysis. When Stage 2 operates as a strict auditor, you retain the ability to say: “Stage 1 produced this; Stage 2 flagged these missing considerations; Stage 3 changed confidence for these reasons.” That traceability is what makes the pipeline feel like engineering rather than improvisation.

Reference prompt contract (sketch):

```text
You are a strict auditor. Given:
(1) persona profile
(2) question
(3) persona DecisionRecord

Return exactly:
【VERDICT】PASS or ADJUST
【GAPS】- bullet list of missing considerations
【ADJUSTMENTS】- bullet list of concrete fixes to the reasoning or risk
```

Reference parsing (sketch):

```python
import re


def parse_audit(text: str) -> tuple[bool, list[str], list[str]]:
	# bool: pass?
	verdict_pass = bool(re.search(r"【VERDICT】\s*PASS", text))
	gaps = re.findall(r"^\-\s*(.+)$", extract_block(text, "【GAPS】"), flags=re.M)
	adjustments = re.findall(r"^\-\s*(.+)$", extract_block(text, "【ADJUSTMENTS】"), flags=re.M)
	return verdict_pass, gaps, adjustments
```

![Feedback parsing logic: compress natural language into structured fields.](/img/persona-ai-decision-system/LLM_Persona_Feedback_Logic.png)

This figure highlights a practical implementation point that often separates “LLM demos” from “LLM systems”: parsing is a first-class component. The system treats the LLM output as semi-structured text and immediately reduces it into deterministic fields (pass/fail, gaps, adjustments). This reduction is not cosmetic—it makes the LLM’s contribution comparable across personas and across runs. It also enables quality gates: reject malformed outputs, cap the number of adjustments, require at least one concrete gap when verdict is ADJUST, and log contract violations.

The deeper logic is that control is achieved through interfaces. You rarely control an LLM by adding more words to a prompt. You control it by narrowing the I/O format and validating it like any other dependency.

The key is not the exact regex; it is the **principle of controlled natural language**—free-form generation is compressed into structured, auditable deltas.

### 3.3 Stage 3 — Multi-Round Reflection (Local Deepening)

**Failure mode addressed:** single-pass critique often misses second-order effects (long-term consequences, opportunity cost, worst-case planning).

Stage 3 is a local reflection loop. The loop forces the system to write down counterfactuals and devil’s-advocate arguments, then update confidence and (occasionally) flip the verdict.

Reference loop (sketch):

```python
REFLECTION_QUESTIONS = [
	"What is the strongest argument against my verdict?",
	"What is the worst-case scenario if I am wrong?",
	"What alternative action dominates my current plan?",
]


def reflect(record: DecisionRecord) -> DecisionRecord:
	for q in REFLECTION_QUESTIONS:
		note = f"Q: {q} | A: ..."  # in practice this is a persona-style local generator
		record.reflection_notes.append(note)
		# Example confidence update rule
		if "worst-case" in q.lower() and record.risk_score >= 7:
			record.confidence = max(1, record.confidence - 1)
	return record
```

The output of Stage 3 is not “more text.” It is **a trajectory**: how confidence and reasoning changed over explicit rounds.

![Stage 3 reflection trajectory: confidence and verdict evolution across rounds.](/img/persona-ai-decision-system/Stage3_Reflection_Trajectory.png)

This figure is the visual counterpart of “traceable reflection.” Reflection is often marketed as “think harder,” but in engineering terms it should be treated as a state transition process. A persona starts with an initial verdict and confidence; each reflection round applies a targeted stress test (strongest counterargument, worst-case scenario, dominating alternative), and the persona updates confidence and notes accordingly. The trajectory matters because it reveals fragility. If confidence collapses after a single worst-case probe, the initial decision was unstable. If confidence remains high but the notes are shallow, the reflection mechanism may be too weak or too templated.

From a system perspective, the trajectory is also where you can enforce invariants. You can require each round to contribute at least one new consideration, you can bound confidence changes, and you can flag personas that never change regardless of evidence. Those invariants are impossible to enforce if reflection is purely free-form text.

![Individual decision evolution (CoT-like trace): the audit trail from initial verdict to final.](/img/persona-ai-decision-system/Individual_Decision_Evolution_CoT.png)

This figure captures the “decision evolution record,” which functions like a chain-of-thought substitute without requiring you to publish raw hidden reasoning. The point is not to dump a long internal monologue; the point is to preserve decision-relevant transitions: which round introduced the key doubt, which alternative was considered, and how the final verdict diverged (or did not) from the initial. In practice, this artifact enables post-mortems. If the final outcome is judged wrong later, you can localize failure: Stage 1 heuristics, Stage 2 auditing, or Stage 3 reflection depth.

### 3.4 Stage 4 — Persona Debate (Claim-Grounded Rebuttals)

**Failure mode addressed:** multi-agent debate degenerates into repetitive roleplay (“template spam”), especially when agents do not reference each other’s claims.

Stage 4 introduces a small but decisive primitive: **claim extraction**. Each persona must first state what the opponent is claiming (in one sentence), and only then write a rebuttal that targets that claim.

![Persona battle menu: navigation to debate, judge, and synthesis views.](/img/persona-ai-decision-system/Persona_Battle_Menu_Night_Light.png)

This UI view matters because it shows how debate is positioned in the overall workflow. Debate is not the end product; it is one analysis instrument among others. The menu structure implicitly encodes a recommended order: inspect individual decisions first, run debates to surface contradictions second, consult the judge to assess quality third, and only then perform synthesis. This ordering is intentional. If you synthesize too early, you will average over unchallenged reasoning. If you debate without judging, you risk amplifying the loudest persona rather than the most coherent one.

From a systems perspective, this is also a guardrail against a common “multi-agent theatre” failure mode. The UI treats debates as traceable artifacts that must eventually be evaluated and aggregated, not as entertainment.

Reference debate turn generator (sketch):

```python
def extract_claim(opponent_record: DecisionRecord) -> str:
	# Minimal extraction: compress reasoning into one assertive sentence.
	# In practice, you can add a classifier for categories like morality/control/fun/nihilism/risk.
	return f"Opponent argues: {opponent_record.reasoning[:120]}..."


def rebut(persona: PersonaConfig, claim: str, my_record: DecisionRecord) -> str:
	# Targeted rebuttal template with persona priors
	if persona.decision_style == "deontological":
		return f"Your claim ignores moral constraints. {claim}"
	if persona.decision_style == "hedonic":
		return f"Your claim undervalues immediate utility and joy. {claim}"
	return f"Your claim is under-justified given the risk and constraints. {claim}"


def debate_turn(speaker: PersonaConfig, target: PersonaConfig, target_record: DecisionRecord, my_record: DecisionRecord) -> DebateTurn:
	claim = extract_claim(target_record)
	rebuttal = rebut(speaker, claim, my_record)
	return DebateTurn(speaker=speaker.name, target=target.name, claim=claim, rebuttal=rebuttal)
```

The debate stage is also where **conflict intensity** becomes computable. A simple version combines (a) whether verdicts disagree and (b) how far apart personas are in risk preference.

```python
def conflict_intensity(a: PersonaConfig, b: PersonaConfig, ra: DecisionRecord, rb: DecisionRecord) -> float:
	disagree = 1.0 if ra.verdict != rb.verdict else 0.0
	risk_gap = abs(a.risk - b.risk)
	return 0.5 * disagree + 0.5 * risk_gap
```

![Stage 4 debate arena: pairwise debates with conflict intensity visualization.](/img/persona-ai-decision-system/Stage4_Persona_Battle_Arena.png)

This figure shows the debate arena as a matrix of persona-vs-persona interactions. The engineering intent is to make disagreement scannable. Two personas can disagree on the verdict, but also disagree on the underlying dimensions (risk tolerance, moral constraints, utility maximization, nihilism). The arena layout makes it easy to identify which pairs are most conflicting and therefore deserve deeper inspection. In practice, this functions like a heatmap for inconsistency.

Conflict intensity is deliberately simple. It does not claim to measure truth. It measures instability. High intensity means the decision is sensitive to persona assumptions and likely sensitive to missing context. Low intensity means the system is converging and the final recommendation is likely robust.

![Dialogue detail: claim-grounded targeted rebuttals referencing prior reasoning.](/img/persona-ai-decision-system/Persona_Dialogue_Battle_Detail.png)

This figure is where the claim-extraction design becomes concrete. A good debate turn is not “I disagree.” It is: “I understand your claim X, and here is why X fails under constraint Y or evidence Z.” By forcing each turn to restate the opponent’s claim, the system reduces straw-manning and increases relevance. This also makes debates easier to evaluate: the judge can score whether a rebuttal actually addresses the extracted claim or merely changes the subject.

This is one of the highest-leverage choices in multi-agent systems. Without claim grounding, debates become a sequence of templated, non-overlapping monologues. With claim grounding, even simple local templates can produce inspectable, judgeable content.

### 3.5 Stage 4.5 — Judge Evaluation (Quality Control)

**Failure mode addressed:** debates can be entertaining but low-quality. Without evaluation, the system cannot distinguish “strong reasoning” from “confident nonsense.”

Stage 4.5 uses a judge to score each persona’s contribution along multiple axes (logic consistency, realism, bias). The exact scoring function can vary, but the main point is to produce a comparable metric.

![Judge evaluation matrix: logic/realism/bias metrics and composite scoring.](/img/persona-ai-decision-system/Stage4_5_Judge_Evaluation_Matrix.png)

This figure illustrates why the judge is not “UI decoration.” Once you have multiple personas producing large amounts of text (reasoning, reflection notes, rebuttals), you need a mechanism to compress the information into a decision-relevant ranking. The evaluation matrix is that compression. Logic consistency rewards internal coherence; realism rewards actionable, grounded considerations; bias penalizes overly one-sided arguments that ignore plausible counterevidence. The composite score is not a truth score—it is a reliability proxy that helps the pipeline decide which voices to trust more in aggregation.

The deeper logic is that aggregation without quality control is just averaging noise. Stage 4.5 provides a place to enforce standards, detect degenerate personas, and prevent the system from being dominated by rhetorical style.

The composite score used in this project has the following structure:

$$
score = 3\cdot logic + 3\cdot realism + 3\cdot (1-bias) + min(confidence/10,1)
$$

In a code contract, the same idea can be written as:

```python
def composite(logic: float, realism: float, bias: float, confidence: int) -> float:
	# logic/realism/bias are 0..10
	return 3 * logic + 3 * realism + 3 * (10 - bias) + min(confidence, 10)
```

![Best/worst persona report: qualitative explanation for score differences.](/img/persona-ai-decision-system/Judge_Best_Worst_Persona_Report.png)

This figure complements the numeric matrix with human-readable attribution. Numbers alone do not tell you why a persona is “best” or “worst.” The report format forces the judge to justify the ranking with concrete reasons (for example, missed a key constraint, inconsistent risk assessment, unrealistic plan, or off-profile rationale). This is crucial for trust: users can disagree with the judge, but they can do so with evidence.

In an engineering workflow, the report is also a debugging tool. If the same persona is repeatedly ranked worst for the same reason, you can adjust its parameters, heuristics, or debate templates. That is how the system improves iteratively rather than being re-prompted endlessly.

### 3.6 Stage 5 — Weighted Synthesis + Conflict Index

**Failure mode addressed:** majority vote is brittle. It ignores which personas are more reliable, how confident they are, and whether the system is sharply divided.

Stage 5 computes two weighted scores:

- $score_{do} = \sum weight \cdot confidence/10$ for personas voting **DO**
- $score_{not} = \sum weight \cdot confidence/10$ for personas voting **DO_NOT**

Then it reports a **conflict index** (how split the system is). 

This makes the output actionable. A “DO” recommendation with high conflict is operationally different from a “DO” recommendation with near-unanimity.

![Final weighted synthesis: do/not-do scores and conflict index.](/img/persona-ai-decision-system/Stage5_Final_Weighted_Synthesis.png)

This figure is the pipeline’s final interface. The important point is that the system outputs more than a verdict: it outputs scores and conflict. Scores provide a quantitative rationale for why one side won the synthesis; conflict provides a stability warning. In real decision-making, this distinction matters. If conflict is low, you can act quickly. If conflict is high, the system is telling you that the decision is sensitive to persona assumptions and likely sensitive to missing context.

Conceptually, conflict index is an honesty mechanism. It prevents the system from pretending certainty when internal evidence is split. Even if the final verdict is “DO,” a high conflict index suggests adding safeguards, seeking additional information, or running another persona pack focused on the missing dimensions.

Reference aggregation (sketch):

```python
def synthesize(records: list[DecisionRecord]) -> dict:
	do_score = 0.0
	not_score = 0.0
	do_count = 0
	not_count = 0

	for r in records:
		s = r.weight * (r.confidence / 10.0)
		if r.verdict == "DO":
			do_score += s
			do_count += 1
		else:
			not_score += s
			not_count += 1

	total = max(1, do_count + not_count)
	p_do = do_count / total
	p_not = not_count / total
	conflict = abs(p_do - p_not) * 100

	final = "DO" if do_score >= not_score else "DO_NOT"
	return {
		"final": final,
		"score_do": round(do_score, 3),
		"score_not": round(not_score, 3),
		"conflict_index_pct": round(conflict, 1),
	}
```

## 4. Implementation Notes (Engineering-Oriented)

### 4.1 Why “Local-First” Matters

If Stage 1 is deterministic and parameter-driven, personas become *real computational entities*, not mere prompt skins. This makes disagreements stable across runs and easier to test. It also makes failures more diagnosable: if a persona consistently over-accepts high-risk questions, you adjust parameters or heuristics rather than “prompt harder.”

### 4.2 Contracts and Parsing Over Free-Form Text

Whenever an LLM is used (validator or judge), the system should enforce:

- **Hard output grammar** (fixed headers, bullet lists)
- **Strict parsing** (extract fields, reject malformed outputs)
- **Write-back as deltas** (gaps/adjustments) rather than rewriting the whole record

This is what turns an LLM from a “story generator” into a composable system component.

### 4.3 Logging as a First-Class Artifact

In a multi-stage system, logs are not merely for debugging—they are the product. A practical pattern is to store one JSON file per run:

```json
{
  "question": "...",
  "personas": ["A", "B", "C", "D"],
  "stage1": {"records": [...]},
  "stage2": {"audits": [...]},
  "stage3": {"reflections": [...]},
  "stage4": {"debates": [...]},
  "stage45": {"judge": [...]},
  "stage5": {"synthesis": {...}}
}
```

With this layout, you can:

- reproduce a UI view from a saved run
- compare runs across persona sets
- create regression tests that validate invariants (e.g., a persona’s verdict must be stable for a fixed input)

## 5. Limitations and Future Work

No pipeline eliminates uncertainty; it merely makes uncertainty explicit. The main limitations are:

- **Heuristic brittleness** in Stage 1: keyword-based risk estimation can miss domain nuance.
- **Evaluator alignment** in Stage 4.5: judge metrics can encode a bias toward “sounding rational.”
- **Persona design**: poorly specified personas can produce artificial disagreement.

Next steps that are most valuable in practice:

- add deterministic test cases for Stage 1 and Stage 3
- add report export (“one-page decision brief”)
- modularize persona packs to support rapid swap-in/out

---

## 中文讲解

0.摘要

多数大模型应用优化的是“流畅度”：你提出问题，模型给出一段连贯的文字，然后交互结束。对于信息检索或起草文本，这种交互模式往往足够；但对于决策类问题，它通常不够，因为决策需要读者理解“为什么这样建议”“考虑过哪些备选方案”“在存在分歧时结论到底稳不稳”。

本文描述一种人格驱动的决策系统：把一次决策视为多阶段流水线，而不是一次模型回复。其核心思想很简单：强制系统生成并保留可检查的中间产物（结构化裁决、置信度、风险、反驳、评估分数等），再把这些产物聚合成最终建议，并同时给出可量化的冲突指数。

由于这份博客工作区快照并未包含完整实验代码，本文将用“参考实现骨架”（数据契约+伪代码）来忠实表达系统设计；重点放在工程取舍、可复现结构，以及每个阶段分别缓解哪一类失败模式。

1.引言：为什么是“决策系统”（而不是聊天机器人）

决策和“回答问题”是两类不同的问题。要让一个决策在实践中可用，它至少要提供三种能力：

1)可追踪性：能够审计推理链，定位哪些因素影响了最终裁决。

2)结构化的多样性：如果存在多种人格，它们应当因为目标与风险偏好不同而产生分歧，而不是仅仅文风不同。

3)自检与冲突处理：真实决策通常包含不确定性、信息缺口与论证不一致；系统应能检测这些问题，并要么修补要么清晰暴露。

换句话说，目标不是“一个完美答案”，而是一份决策报告：由一组结构化中间产物组成，能被检查、质疑、比较与聚合。

1.1流水线概览（阶段1→5）

整体流程组织为：

1)阶段1（本地）：每个人格在不依赖大模型的情况下输出初始结构化裁决。
2)阶段2（LLM）：大模型作为验证者/审计员，检查一致性与逻辑缺口。
3)阶段3（本地）：多轮反思基于明确的反方论点调整置信度与裁决。
4)阶段4（本地/混合）：人格辩论，基于主张抽取生成定点反驳。
5)阶段4.5（裁判）：对推理与辩论质量进行打分评估。
6)阶段5（聚合）：加权合成给出最终建议与冲突指数。

最关键的设计约束是：每个阶段都必须输出结构化字段（而不仅是自由文本），这样后续阶段才能“计算”这些产物。

2.数据契约：把“决策记录”作为系统骨干

在讨论提示词或UI之前，最重要的工程选择其实是：决策的“标准形态”是什么？如果系统不能把决策表示为数据，它就无法被审计与聚合。

下面是一份最小参考契约（Python风格dataclasses），覆盖了关键中间产物。它刻意保持精简：任何不在契约里的信息，都可以视为“不可审计”。

```python
from __future__ import annotations

from dataclasses import dataclass, field
from typing import Literal, List, Dict, Optional

Verdict = Literal["DO", "DO_NOT"]


@dataclass
class PersonaConfig:
	name: str
	risk: float          # 0.0 (risk-averse) → 1.0 (risk-seeking)
	emotion: float       # 0.0 → 1.0
	logic: float         # 0.0 → 1.0
	worldview: str       # short descriptor (used for claim extraction priors)
	decision_style: str  # e.g., "utilitarian", "deontological", "hedonic"
	tone: str            # used only for surface rendering


@dataclass
class DecisionRecord:
	persona: str
	verdict: Verdict
	confidence: int              # 1..10
	weight: float                # 0.1..1.0 (used in aggregation)
	risk_score: int              # 1..10 (question risk as perceived by persona)
	reasoning: str               # auditable rationale (not a prompt)
	gaps: List[str] = field(default_factory=list)
	adjustments: List[str] = field(default_factory=list)
	reflection_notes: List[str] = field(default_factory=list)


@dataclass
class DebateTurn:
	speaker: str
	target: str
	claim: str
	rebuttal: str
	evidence: Optional[str] = None


@dataclass
class JudgeScore:
	persona: str
	logic: float      # 0..10
	realism: float    # 0..10
	bias: float       # 0..10 (higher = more biased)
	confidence: int   # 1..10

	@property
	def composite(self) -> float:
		# Same structure as the blog UI: reward logic/realism, penalize bias.
		return 3 * self.logic + 3 * self.realism + 3 * (10 - self.bias) + min(self.confidence, 10)
```

这份契约完成三件事：

1)把裁决（DO/DO_NOT）与理由（可解释性）以及置信度（不确定性）分离。

2)为缺口（gaps）与调整（adjustments）提供显式字段，使阶段2能在不重写全部理由的情况下对记录做增量修正。

3)为评估与聚合（阶段4.5与阶段5）提供一致接口。

3.逐阶段方法（附失败模式）

这一部分写法更像论文的方法章：每个阶段的存在都有明确的失败模式动机。

在进入阶段细节前，先看系统是如何被“消费”的：UI刻意极简，它不是聊天窗口，而是一次多阶段运行的报告视图。关键点在于，它把中间产物以可折叠、按人格归属的对象形式暴露出来，让用户无需阅读原始日志就能审计决策。

![Streamlit 入口UI：决策问题的单一输入点。](/img/persona-ai-decision-system/Persona_AI_Entry_UI.png)

这张图背后的设计思想是：决策系统应该像跑一次实验。你提交一个问题作为输入变量，系统生成一条可复现的输出轨迹。UI因此按“阶段”组织，而不是按“对话轮次”组织。这样做能避免LLM产品里常见的陷阱：用户把“很自信的一段话”误当成“可靠的决策”。当输出被强制以分阶段轨迹呈现时，读者更容易去检查假设（风险分、置信度、缺口、反驳）而不是只看最终结论。

3.1阶段1——本地人格初判（对抗收敛）

失败模式：如果所有推理都外包给LLM，多个人格往往会收敛到同一结论，只剩文风差异。

阶段1强制每个人格用确定性、参数驱动的启发式规则做出裁决。目标不是“完美”，而是“多样且可审计”。

![阶段1初判列表：按人格给出裁决、置信度、权重与风险分。](/img/persona-ai-decision-system/Persona_Initial_Decisions_List.png)

这张图不只是UI截图；它表达了系统最核心的工程纪律：每个人格必须提交结构化初始记录。输出不是一段话，而是紧凑字段元组：裁决、置信度、权重、风险分。这种结构使后续阶段可计算。尤其是置信度与权重，它们让聚合能够尊重不确定性与人格可靠性，而不是把每票当作同等权重。风险分把推理锚定在“问题的潜在代价”上，后续阶段可以挑战与校准它。

在实践中，阶段1应当产生稳定的分歧：如果两个人格在风险偏好或决策风格上不同，它们在同一问题上跨多次运行也应当持续分歧。稳定性使人格成为可测试模块；否则人格差异就会沦为提示噪声：难以调试、难以复现、容易过拟合。

参考逻辑（骨架）：

```python
RISK_KEYWORDS = {
	"finance": ["invest", "loan", "mortgage", "crypto"],
	"career": ["quit", "resign", "job", "offer"],
	"health": ["surgery", "treatment", "diagnosis"],
}


def estimate_question_risk(question: str) -> int:
	hits = 0
	q = question.lower()
	for _, words in RISK_KEYWORDS.items():
		hits += sum(1 for w in words if w in q)
	# Simple clamp to 1..10
	return max(1, min(10, 1 + hits * 2))


def persona_score(persona: PersonaConfig, question_risk: int) -> float:
	# risk preference vs perceived risk
	# higher risk preference should counteract high question risk
	base = 5.0
	base += (persona.logic - 0.5) * 2.0
	base += (persona.risk - 0.5) * (11 - question_risk) * 0.6
	base -= (persona.emotion - 0.5) * question_risk * 0.4
	return base


def stage1_decide(persona: PersonaConfig, question: str) -> DecisionRecord:
	risk = estimate_question_risk(question)
	s = persona_score(persona, risk)
	verdict: Verdict = "DO" if s >= 5.0 else "DO_NOT"
	confidence = int(max(1, min(10, round(abs(s - 5.0) * 2 + 4))))
	weight = max(0.1, min(1.0, 0.4 + persona.logic * 0.4 + persona.risk * 0.2))
	reasoning = f"Risk={risk}. Score={s:.2f}. Style={persona.decision_style}."
	return DecisionRecord(
		persona=persona.name,
		verdict=verdict,
		confidence=confidence,
		weight=weight,
		risk_score=risk,
		reasoning=reasoning,
	)
```

![理由详情：每个人格可展开的本地理由面板（可审计产物）。](/img/persona-ai-decision-system/Persona_Reasoning_Details.png)

这张图展示了阶段1的另一半：决策记录不只有标量字段，也需要审计友好的叙述解释。关键在于：叙述应由本地、人格条件化的生成器（或结构化模板）产生，而不是让LLM临场发挥。这样做的动机很直接：如果你要可解释性，就要避免解释在不同运行间漂移。这个理由面板被设计成像错误报告一样可检查：你可以读人格的理由，看它是否匹配风险分，再判断人格是否遗漏了明显约束。

它也防止“空洞合理化”。理由应当引用可计算产物（风险分、阈值、置信度），而不是只写得像真的。如果人格说“高风险”但风险分很低，这种不一致是可见且可在启发式层修复的。

尽管阶段1很“简单”，但它有两个强力效果：

1)产出可组合的数值产物（风险分、置信度、权重）。
2)产生由参数驱动而非提示驱动的“有意义分歧”。

3.2阶段2——把LLM当作验证者（审计，而非替代思考）

失败模式：确定性启发式会脆弱；人格可能给出逻辑不一致、忽略显然因素的裁决。

阶段2把LLM定位为审计员。它不会取代阶段1，而是在严格输出契约下对阶段1结果进行批判。契约至关重要，因为系统需要解析、存储并跨人格/跨运行比较验证结果。

![阶段2一致性检查：验证者返回结构化缺口与调整建议。](/img/persona-ai-decision-system/Stage2_LLM_Consistency_Check.png)

这张图捕捉了混合系统最重要的设计选择：角色分离。LLM不被要求“做决策”，而是被要求检查现有决策记录，并识别两类缺陷：人格一致性问题（理由与人格画像不匹配）以及逻辑缺口（遗漏约束、缺乏支撑的跳跃、忽略备选方案）。输出被限制在小型契约内，使系统能把LLM贡献以“增量”（缺口与调整）方式写回，而不是覆盖原记录。

原因在于：覆盖会破坏归因与复现。一旦阶段2重写一切，你就无法分辨哪个阶段引入了哪个假设，也无法做事后复盘。阶段2作为严格审计员时，你可以清楚地说：阶段1产出了这些；阶段2标注了这些遗漏；阶段3因为这些理由调整了置信度。可追踪性正是这条流水线具备工程感（而非即兴发挥感）的关键。

参考提示契约：

```text
You are a strict auditor. Given:
(1) persona profile
(2) question
(3) persona DecisionRecord

Return exactly:
【VERDICT】PASS or ADJUST
【GAPS】- bullet list of missing considerations
【ADJUSTMENTS】- bullet list of concrete fixes to the reasoning or risk
```

参考解析：

```python
import re


def parse_audit(text: str) -> tuple[bool, list[str], list[str]]:
	# bool: pass?
	verdict_pass = bool(re.search(r"【VERDICT】\s*PASS", text))
	gaps = re.findall(r"^\-\s*(.+)$", extract_block(text, "【GAPS】"), flags=re.M)
	adjustments = re.findall(r"^\-\s*(.+)$", extract_block(text, "【ADJUSTMENTS】"), flags=re.M)
	return verdict_pass, gaps, adjustments
```

![反馈解析逻辑：把自然语言压缩为结构化字段。](/img/persona-ai-decision-system/LLM_Persona_Feedback_Logic.png)

这张图强调了一个把“LLM演示”变成“LLM系统”的关键点：解析是一级组件。系统把LLM输出当作半结构化文本，并立刻将其归约为确定性字段（通过/不通过、缺口、调整）。这种归约不是装饰，而是让LLM贡献能够跨人格、跨运行比较；也使质量门槛成为可能：拒绝格式不合格输出、限制调整条数、当 verdict=ADJUST 时要求至少一个具体缺口，并记录契约违规。

更深的逻辑是：控制来自接口。你很少靠给提示词加更多字来控制LLM；你靠收窄I/O格式，并像验证其他依赖一样验证它。

关键不在正则表达式写法，而在受控自然语言原则：自由生成会被压缩成结构化、可审计的增量。

3.3阶段3——多轮反思（本地加深）

失败模式：一次性批判常常遗漏二阶影响（长期后果、机会成本、最坏情况规划）。

阶段3是本地反思循环。循环要求系统写下反事实与“唱反调”论点，然后更新置信度，并在少数情况下翻转裁决。

参考循环（骨架）：

```python
REFLECTION_QUESTIONS = [
	"What is the strongest argument against my verdict?",
	"What is the worst-case scenario if I am wrong?",
	"What alternative action dominates my current plan?",
]


def reflect(record: DecisionRecord) -> DecisionRecord:
	for q in REFLECTION_QUESTIONS:
		note = f"Q: {q} | A: ..."  # in practice this is a persona-style local generator
		record.reflection_notes.append(note)
		# Example confidence update rule
		if "worst-case" in q.lower() and record.risk_score >= 7:
			record.confidence = max(1, record.confidence - 1)
	return record
```

阶段3的输出不是“更多文本”，而是一条轨迹：置信度与理由如何在明确回合中发生变化。

![阶段3反思轨迹：跨回合的置信度与裁决演化。](/img/persona-ai-decision-system/Stage3_Reflection_Trajectory.png)

这张图是“可追踪反思”的可视化对应物。反思常被营销为“想得更深”，但工程上它应被视为状态转移过程：人格从初始裁决与置信度出发；每一轮反思施加一个定向压力测试（最强反驳、最坏情况、支配性替代方案）；人格据此更新置信度与记录。轨迹之所以重要，是因为它揭示脆弱性：如果置信度在一次最坏情况探测后迅速崩塌，初始决策就不稳；如果置信度始终很高但笔记内容空洞，反思机制可能太弱或太模板化。

从系统角度，轨迹也是施加不变量的地方：可以要求每轮至少贡献一个新考虑因素，可以限制置信度变化幅度，可以标记“无论何种证据都不变”的人格。如果反思只是不受控自由文本，这些约束无法落地。

![单人格决策演化（类似CoT的审计轨迹）：从初判到最终的变更链。](/img/persona-ai-decision-system/Individual_Decision_Evolution_CoT.png)

这张图展示“决策演化记录”，它起到类似思维链替代物的作用，同时不要求你公开原始的隐藏推理。重点不是倾倒长篇内心独白，而是保留决策相关的转折：哪一轮引入了关键疑点、考虑了哪个替代方案、最终裁决是否以及如何偏离初判。这个产物支持事后复盘：如果未来证明最终结论错误，你可以定位失败发生在阶段1启发式、阶段2审计，还是阶段3反思深度。

3.4阶段4——人格辩论（基于主张的定点反驳）

失败模式：多智能体辩论容易退化为重复的角色扮演模板，尤其当智能体不引用对方主张时。

阶段4引入一个小而关键的原语：主张抽取。每个人格必须先用一句话陈述对手在主张什么，然后才能写出针对该主张的反驳。

![人格辩论菜单：进入辩论/裁判/合成视图的导航。](/img/persona-ai-decision-system/Persona_Battle_Menu_Night_Light.png)

这张UI之所以重要，是因为它展示辩论在整体工作流中的定位：辩论不是终产物，它只是分析工具之一。菜单结构隐含推荐顺序：先检查单人格初判；再用辩论暴露矛盾；再参考裁判评估质量；最后才做合成。这个顺序是刻意的：过早合成会把未经挑战的推理平均掉；只辩论不评估会放大“最吵的人格”而非“最一致的人格”。

从系统视角，这也是防止“多智能体戏剧化”的护栏：UI把辩论当作需要被评估与聚合的可追踪产物，而不是娱乐对话。

参考辩论回合生成：

```python
def extract_claim(opponent_record: DecisionRecord) -> str:
	# Minimal extraction: compress reasoning into one assertive sentence.
	# In practice, you can add a classifier for categories like morality/control/fun/nihilism/risk.
	return f"Opponent argues: {opponent_record.reasoning[:120]}..."


def rebut(persona: PersonaConfig, claim: str, my_record: DecisionRecord) -> str:
	# Targeted rebuttal template with persona priors
	if persona.decision_style == "deontological":
		return f"Your claim ignores moral constraints. {claim}"
	if persona.decision_style == "hedonic":
		return f"Your claim undervalues immediate utility and joy. {claim}"
	return f"Your claim is under-justified given the risk and constraints. {claim}"


def debate_turn(speaker: PersonaConfig, target: PersonaConfig, target_record: DecisionRecord, my_record: DecisionRecord) -> DebateTurn:
	claim = extract_claim(target_record)
	rebuttal = rebut(speaker, claim, my_record)
	return DebateTurn(speaker=speaker.name, target=target.name, claim=claim, rebuttal=rebuttal)
```

辩论阶段也让冲突强度变得可计算。一种简单定义把(a)裁决是否分歧与(b)风险偏好差异组合起来。

```python
def conflict_intensity(a: PersonaConfig, b: PersonaConfig, ra: DecisionRecord, rb: DecisionRecord) -> float:
	disagree = 1.0 if ra.verdict != rb.verdict else 0.0
	risk_gap = abs(a.risk - b.risk)
	return 0.5 * disagree + 0.5 * risk_gap
```

![阶段4辩论竞技场：按人格两两辩论并可视化冲突强度。](/img/persona-ai-decision-system/Stage4_Persona_Battle_Arena.png)

这张图把辩论呈现为人格对人格的矩阵。工程意图是让分歧可扫描：两个人格可以在裁决上分歧，也可能在更底层维度分歧（风险容忍、道德约束、效用最大化、虚无主义等）。这种布局使你能快速定位冲突最强的组合，从而投入更多检查。

冲突强度被刻意设计得很朴素：它不声称衡量真理，只衡量不稳定性。强度高意味着结论对人格假设更敏感，也更可能对缺失上下文敏感；强度低意味着系统在收敛，最终建议更可能稳健。

![对话细节：基于抽取主张的定点反驳，并引用先前理由。](/img/persona-ai-decision-system/Persona_Dialogue_Battle_Detail.png)

这张图让主张抽取设计落地：好的辩论回合不是“我不同意”，而是“我理解你主张X，并解释X在约束Y或证据Z下为何失败”。强制每回合复述对方主张能减少稻草人攻击，提升相关性，也让评估更容易：裁判可以判断反驳是否真正回应抽取主张，还是只是转移话题。

这是多智能体系统里杠杆极高的选择之一。没有主张锚定，辩论会退化为模板化、互不相交的独白；有了主张锚定，即使本地模板很简单，也能产出可检查、可评估的内容。

3.5阶段4.5——裁判评估（质量控制）

失败模式：辩论可能很热闹但质量低；没有评估系统就无法区分“强推理”与“自信胡说”。

阶段4.5引入裁判，从多个维度对每个人格贡献打分（逻辑一致性、现实性、偏见）。具体打分函数可变，但核心是产出可比较指标。

![裁判评估矩阵：逻辑/现实/偏见指标与综合评分。](/img/persona-ai-decision-system/Stage4_5_Judge_Evaluation_Matrix.png)

这张图说明裁判不是“UI装饰”。当多个人格生成大量文本（理由、反思笔记、反驳）时，你需要把信息压缩为与决策相关的排名。评估矩阵就是这种压缩：逻辑一致性奖励内在自洽；现实性奖励可执行、贴地的考虑；偏见惩罚过度单边、忽视反证的论证。综合分不是“真理分”，而是可靠性代理，帮助聚合阶段更信任高质量声音。

更深的逻辑是：没有质量控制的聚合，本质上是在平均噪声。阶段4.5提供执行标准、识别退化人格、避免系统被修辞风格主导的位置。

本项目使用的综合分结构如下：

$$
score = 3\cdot logic + 3\cdot realism + 3\cdot (1-bias) + min(confidence/10,1)
$$

在代码契约中可写为：

```python
def composite(logic: float, realism: float, bias: float, confidence: int) -> float:
	# logic/realism/bias are 0..10
	return 3 * logic + 3 * realism + 3 * (10 - bias) + min(confidence, 10)
```

![最佳/最差人格报告：对评分差异的定性归因解释。](/img/persona-ai-decision-system/Judge_Best_Worst_Persona_Report.png)

这张图用可读归因补全了数字矩阵。仅有数字无法解释为什么某人格最好或最差；报告强制裁判给出具体理由（例如遗漏关键约束、风险评估不一致、计划不现实、理由与画像不符）。这对信任很关键：用户可以不同意裁判，但能以证据为基础不同意。

在工程工作流里，报告也是调试工具。如果同一人格反复因为同一原因被评为最差，你就能调整它的参数、启发式或辩论模板，从而迭代改进，而不是无休止地“再写一版提示词”。

3.6阶段5——加权合成+冲突指数

失败模式：多数票很脆弱；它忽略人格可靠性、置信度以及系统是否严重分裂。

阶段5计算两个加权分：

1)$score_{do}=\sum weight\cdot confidence/10$（对投DO的人格求和）
2)$score_{not}=\sum weight\cdot confidence/10$（对投DO_NOT的人格求和）

这样输出就更可执行：同样是“DO”，低冲突与高冲突在操作上是两种完全不同的建议。

![最终加权合成：Do/Not分数与冲突指数。](/img/persona-ai-decision-system/Stage5_Final_Weighted_Synthesis.png)

这张图是流水线的最终接口。重点在于：系统输出的不只是裁决，还输出分数与冲突。分数给出为何某一侧胜出的量化依据；冲突给出稳定性警告。在现实决策中，这一点很关键：若冲突低，可以快速行动；若冲突高，系统是在提示决策对人格假设敏感、也可能对缺失上下文敏感。

概念上，冲突指数是一种“诚实机制”：阻止系统在内部证据分裂时假装确定。即使最终裁决是DO，高冲突也提示你增加保护措施、补充信息，或引入针对缺失维度的人格包再次运行。

参考聚合：

```python
def synthesize(records: list[DecisionRecord]) -> dict:
	do_score = 0.0
	not_score = 0.0
	do_count = 0
	not_count = 0

	for r in records:
		s = r.weight * (r.confidence / 10.0)
		if r.verdict == "DO":
			do_score += s
			do_count += 1
		else:
			not_score += s
			not_count += 1

	total = max(1, do_count + not_count)
	p_do = do_count / total
	p_not = not_count / total
	conflict = abs(p_do - p_not) * 100

	final = "DO" if do_score >= not_score else "DO_NOT"
	return {
		"final": final,
		"score_do": round(do_score, 3),
		"score_not": round(not_score, 3),
		"conflict_index_pct": round(conflict, 1),
	}
```

4.实现说明

4.1为什么“本地优先”很重要

如果阶段1是确定性且参数驱动的，人格就会成为真实的计算实体，而不是提示词皮肤。这会让分歧跨运行更稳定、更易测试，也让失败更可诊断：如果某人格总是过度接受高风险问题，你应当调参或改启发式，而不是“加大提示强度”。

4.2用契约与解析替代自由文本

只要使用LLM（验证者或裁判），系统就应当强制：

1)硬输出语法（固定标题、固定条目结构）
2)严格解析（抽取字段、拒绝不合格输出）
3)以增量写回（缺口/调整）而不是重写整个记录

这会把LLM从“故事生成器”变成可组合系统组件。

4.3把日志当作一级产物

在多阶段系统里，日志不仅是调试手段，更是产品本身。一个实用模式是每次运行保存一个JSON文件：

```json
{
  "question": "...",
  "personas": ["A", "B", "C", "D"],
  "stage1": {"records": [...]},
  "stage2": {"audits": [...]},
  "stage3": {"reflections": [...]},
  "stage4": {"debates": [...]},
  "stage45": {"judge": [...]},
  "stage5": {"synthesis": {...}}
}
```

基于这种布局，下一步优化可以：

1)从存档运行重建UI视图
2)比较不同人格集合的运行差异
3)创建回归测试来验证不变量（例如固定输入下某人格裁决应保持稳定）

5.局限与未来工作

没有流水线能消除不确定性，它只能把不确定性显式化。主要局限包括：

1)阶段1启发式脆弱：基于关键词的风险估计会漏掉领域细节。
2)阶段4.5评估对齐：裁判指标可能偏向“听起来理性”的表达。
3)人格设计成本：画像不清的人格会制造人为分歧。

更有价值的下一步通常是：

1)为阶段1与阶段3增加确定性测试用例
2)增加报告导出（“一页决策简报”）
3)把人格包模块化以支持快速替换与对比
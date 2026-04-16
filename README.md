# UtilityMax Skill

A [Claude skill](https://support.anthropic.com/en/articles/12111783-what-are-skills) for converting multi-objective tasks into formal UtilityMax prompts.

Based on [UtilityMax Prompting: A Formal Framework for Multi-Objective Large Language Model Optimization](PAPER_URL) by Ofir Marom.

## The Problem

Most LLM prompts specify objectives in natural language, which is inherently ambiguous when multiple objectives must be balanced simultaneously. Consider:

> "Maximise profit subject to a medium level of risk"

The LLM has to guess what "medium" means and how to trade profit against risk. No matter how carefully worded, any natural language multi-objective prompt leaves the model *interpreting* rather than *optimising*.

## The Solution

UtilityMax reconstructs the task as an influence diagram with:

- a decision node `A` — the LLM's answer
- chance nodes `X_1, ..., X_n` — task-relevant outcomes
- a utility function `U` the LLM must maximise

The LLM is instructed to find `a* = argmax E[U | A=a]`, forcing it to reason about each objective individually and combine them according to a precise mathematical rule rather than a subjective interpretation.

Validated across Claude Sonnet 4.6, GPT-5.4, and Gemini 2.5 Pro on a multi-objective movie recommendation task (MovieLens 1M), UtilityMax consistently outperforms natural language baselines on Precision@10 and NDCG@10 (`p < 0.01`, one-sided paired Wilcoxon signed-rank test).

## What This Skill Does

Given a user-supplied task and context, this skill converts them into a UtilityMax prompt adhering to the framework's variable and utility guidelines. Specifically, it:

- defines the answer space `A` and populates `K` with the supplied context
- decomposes objectives into random variables with the correct type (binary, continuous, categorical)
- chooses between the two tractable DAG structures (conditional independence or binary gating)
- chooses between multiplicative and additive utility — or a hybrid form if needed
- writes the final `O(a)` formula and fills in the template

## Installation

### Claude.ai

Download `SKILL.md` and upload it via **Settings → Capabilities → Skills**.

```bash
curl -O https://raw.githubusercontent.com/USERNAME/utilitymax-skill/main/SKILL.md
```

### Claude Code (user-scoped)

Make the skill available across all projects:

```bash
mkdir -p ~/.claude/skills/utilitymax
curl -o ~/.claude/skills/utilitymax/SKILL.md \
  https://raw.githubusercontent.com/USERNAME/utilitymax-skill/main/SKILL.md
```

### Claude Code (project-scoped)

Scope the skill to a single repo:

```bash
mkdir -p .claude/skills/utilitymax
curl -o .claude/skills/utilitymax/SKILL.md \
  https://raw.githubusercontent.com/USERNAME/utilitymax-skill/main/SKILL.md
```

## When to Use

Reach for UtilityMax when:

- the task has **multiple competing objectives** that must be balanced
- objectives can be expressed as **probability estimates or expected values**
- you are on a **frontier-class model** — the framework depends on well-calibrated probability estimates

Natural language is usually sufficient when the task has a single objective, or when the LLM lacks the domain knowledge to calibrate the component estimates.

## Example

Task: recommend the top 10 movies for a user in the mood for comedy and romance.

Variables:
- `S` — predicted star rating (categorical)
- `G1` — comedy genre flag (binary)
- `G2` — romance genre flag (binary)

Objective — multiplicative, since a good recommendation must satisfy all three:

```
O(a) = E[S | A=a] × P(G1=1 | A=a) × P(G2=1 | A=a)
```

This is the exact objective validated in the paper.

## How to Know It's Working

The skill is working if outputs include:

- an explicit `O(a)` formula, not a natural language objective
- each objective represented as a distinct random variable with a defined type
- conditioning on parent variables when binary gating is used
- the three-step reasoning protocol (generate candidates → estimate each term → return `argmax`)

## Citation

```bibtex
@techreport{marom2026utilitymax,
  title  = {UtilityMax Prompting: A Formal Framework for Multi-Objective
            Large Language Model Optimization},
  author = {Marom, Ofir},
  year   = {2026},
  type   = {Technical Report},
  url    = {https://arxiv.org/abs/2603.11583}
}
```

## License

MIT
---
name: utilitymax
description: Use this skill whenever the user wants to build a UtilityMax prompt. Triggers include any mention of UtilityMax, requests to formalise a multi-objective task as a utility function, building an influence diagram for LLM prompting, or defining chance nodes / objective variables. Also use when the user describes a task with competing objectives and wants a formal mathematical prompt rather than a natural language one.
---
 
# UtilityMax Prompting — Skill Reference
 
## What This Skill Is
 
UtilityMax is a zero-shot prompting framework for multi-objective LLM tasks. Instead of specifying competing objectives in natural language ("maximise profit at medium risk") — which is inherently ambiguous — you reconstruct the task as an influence diagram and give the LLM a formal utility function to maximise. This skill converts a user-supplied task and context into a UtilityMax prompt.
 
---
 
## The Influence Diagram
 
```
Decision node:   A | K             <- the LLM's answer space (sole root)
Chance nodes:    X1, X2, ..., Xn   <- outcomes that depend on A
Utility node:    U                 <- what we maximise
```
 
`K` is the LLM's knowledge: its parameters plus any external context (chat history, documents, retrieved data). The designer updates `K` by providing context alongside the prompt.
 
Two structural constraints: the graph over `{A, X1, ..., Xn}` is a DAG with `A` as the single root, and a tractability condition holds. The framework supports two tractable structures — conditional independence and binary gating — described below.
 
---
 
## Framework 1: Conditional Independence
 
### Assumption
Each `Xi` is conditionally independent given `A`. No dependency structure among chance nodes.
 
### Utility formulas
 
| Utility | `E[U \| A]` formula | When to use |
|---|---|---|
| **Multiplicative** | `Π_i E[fi(Xi) \| A]` | Objectives are *complementary* — all must jointly hold |
| **Additive** | `Σ_i E[fi(Xi) \| A]` | Objectives are *substitutable* — each contributes independently |
 
### Variable types
- **Continuous/categorical `Xi`**: `fi(Xi) = Xi`. LLM estimates `E[Xi | A=a]`.
- **Binary `Xi` ∈ {0,1}**: `fi(Xi) = Xi`, so `E[fi(Xi) | A=a] = P(Xi=1 | A=a)`. LLM estimates the probability the condition holds.
---
 
## Framework 2: Binary Gating
 
Use when some chance nodes have prerequisite dependencies on others.
 
### Constraints
- All *internal* (non-leaf) nodes must be binary.
- If any parent `Xj = 0` for an internal node `Xi`, then `P(Xi=1 | parents, A) = 0`. A child is active only if all binary parents are active.
- Leaf nodes may be continuous, categorical, or binary.
- **Leaf nodes are conditionally independent of each other given their parents and `A`.** A tree-structured DAG satisfies this automatically; non-tree DAGs need to be checked.
Notation: `pa(Xi) = 1` means all parents of `Xi` equal 1. Let **I** = internal nodes, **L** = leaf nodes.
 
### Utility formulas
 
**Multiplicative** (one term over all nodes):
```
E[U | A] = [ Π_{Xi ∈ I} P(Xi=1 | pa(Xi)=1, A) ]
         × [ Π_{Xj ∈ L} E[fj(Xj) | pa(Xj)=1, A] ]
```
 
**Additive** (one term per leaf, each weighted by its ancestor gating probabilities):
```
E[U | A] = Σ_{Xj ∈ L} [ Π_{Xi ∈ Ij} P(Xi=1 | pa(Xi)=1, A) × E[fj(Xj) | pa(Xj)=1, A] ]
```
where `Ij` = set of internal ancestors of leaf `Xj`. Each term is the traversal probability to leaf `Xj` times its expected utility contribution.
 
---
 
## General Form (for Hybrids)
 
The two frameworks above present the two practical special cases. The paper's fully general tractable utility is:
 
```
U(X) = Σ_{k=1..N} Π_i f_{i,k}(Xi)
```
 
with arbitrary `N ≥ 1` and (under gating) arbitrary subset groupings `Ik ⊆ I, Lk ⊆ L` per term. This allows hybrid objectives — for example, three factors that must jointly hold (multiplicative group) plus an additive bonus term. If a task needs this, construct each term using the formulas above and sum the `N` resulting expressions.
 
---
 
## Variable Guidelines
 
Variable precision is a trade-off:
- **More precise** → less ambiguity; the LLM's estimates align more closely with real-world targets. Risk: may miss signal outside the strict definition.
- **Looser** → more LLM latitude to draw on broader knowledge. Risk: reintroduces the ambiguity UtilityMax is designed to eliminate.
**Core principle:** variables should describe an *observable, measurable event* whose outcome can be verified. This keeps the LLM's estimated probability aligned with the real-world probability.
 
Example — binary variable for collaboration outcome:
- Loose: "collaboration occurs" — vague; LLM has to guess what counts.
- Precise: "at least one person contacts me within two weeks about running a joint experiment" — concrete and verifiable.
When the outcome can be well-specified, prefer precision. When the concept is inherently subjective, some looseness may be acceptable — but recognise you are trading away some of the framework's benefit.
 
---
 
## Utility Guidelines: Multiplicative vs Additive
 
Once variables are defined, ask one question: **if one objective fails entirely, should overall utility collapse to zero, or should the remaining objectives still contribute?**
 
- **Collapse to zero → Multiplicative.** Objectives are complementary; any single failure disqualifies the answer. Example: an item must be in the right genre AND highly rated AND available.
- **Still contribute → Additive.** Objectives are substitutable; each carries standalone value. Example: a content portfolio scoring on reach, engagement, and brand safety.
Under gating: additive utility across leaves means each leaf pathway contributes its own weighted value, so meeting prerequisites for *some* leaves still yields positive utility. Multiplicative means all internal gates and leaves must activate for any utility to accrue.
 
---
 
## The Prompting Template
 
Base template — multiplicative utility, two variables, conditional independence:
 
```
I want you to solve the following task: [TASK DESCRIPTION].
 
Formally, let K represent your knowledge. This includes all your internal
knowledge stored through your parameters as well as any external knowledge
provided in this prompt or chat history.
 
Let P(A | K) represent your probability distribution over answers given K.
Let a be an answer in A.
 
Let X1 | A=a be a random variable representing [DESCRIPTION OF X1] given answer a.
 
Let X2 | A=a be a random variable representing [DESCRIPTION OF X2] given answer a.
 
Your task is to use your domain expertise to find the optimal answer a* that
maximises O(a) = E[X1 | A=a] x E[X2 | A=a]. To do this you must:
 
1. Generate a set of candidate answers.
2. For each candidate answer, estimate E[X1 | A=a] and E[X2 | A=a] individually
   using your internal knowledge then compute O(a) for that candidate.
3. Return the answer a* that maximises O.
```
 
**Adaptations:**
- **n variables**: add further `Xi` definitions and extend `O(a)` accordingly.
- **Additive (CI)**: replace the product in `O(a)` with a sum: `O(a) = E[X1 | A=a] + E[X2 | A=a]`.
- **Binary gating**: in each variable definition, condition on parents being active, e.g. `Let X2 | X1=1, A=a be a random variable representing [DESCRIPTION]`. Write `O(a)` using the gating formula for the chosen utility type.
- **Hybrid (general form)**: construct each of the `N` terms separately and sum them in `O(a)`.
---
 
## Construction Process
 
Given a user's task description and context:
 
1. **State the task and answer space.** One sentence for the task. Define what a single answer `a ∈ A` looks like (a recommended item, a strategy, a ranked list, etc.). Any context provided by the user becomes part of `K`.
2. **Decompose objectives into variables.** For each distinct objective the task cares about, define one `Xi`. Decide its type (continuous/categorical vs binary) and apply the Variable Guidelines — prefer definitions that describe observable, measurable events. Variables should be *necessary and sufficient*: irrelevant variables add noise; missing variables mean optimising a proxy.
3. **Choose the framework.** Default to conditional independence. Switch to binary gating if some objectives are only meaningful when prior conditions are met (prerequisite structure). Under gating, verify leaf conditional independence — tree-structured DAGs satisfy this automatically.
4. **Choose the utility type.** Apply the Utility Guidelines "collapse to zero" test. If the task genuinely has both complementary and substitutable components, use the general form with hybrid grouping.
5. **Fill in the template.** Slot in the task description, variable definitions (conditioned on parents if gating), and the `O(a)` formula matching the chosen framework × utility combination. Keep the three-step reasoning protocol at the end.
---
 
## Worked Example
 
Task: recommend the top 10 movies for a user who is in the mood for comedy and romance.
 
Framework: conditional independence. Utility: multiplicative (an item must satisfy all three objectives to be a good recommendation).
 
Variables:
- `S` — categorical random variable: the predicted star rating the user would give the movie.
- `G1` — binary: the movie is a comedy.
- `G2` — binary: the movie is a romance.
Objective:
```
O(a) = E[S | A=a] × P(G1=1 | A=a) × P(G2=1 | A=a)
```
 
Each term has a clean real-world interpretation the LLM can estimate from movie title and user history alone.
 
---
 
## Variable Design Quick Reference
 
| Objective Type | `Xi` Type | Term in `O(a)` | LLM Estimates |
|---|---|---|---|
| Predicted score / rating | Continuous / categorical | `E[Xi \| A=a]` | Expected value |
| Category / genre membership | Binary | `P(Xi=1 \| A=a)` | Probability |
| Risk avoidance (derived: `fi(Xi) = 1 − Xi`) | Binary | `P(Xi=0 \| A=a)` | Probability of non-event |
| Nested prerequisite (internal gate) | Binary | `P(Xi=1 \| pa=1, A=a)` | Conditional gating probability |
| Outcome behind a gate (leaf) | Any | `E[fj(Xj) \| pa=1, A=a]` | Expected value conditioned on gate active |
 
---
 
## Construction Checklist
 
- [ ] Task description in one sentence; `K` populated with user-provided context
- [ ] Answer space `A` clearly defined
- [ ] Framework chosen: conditional independence or binary gating (leaf CI verified if gating)
- [ ] Each objective mapped to an `Xi` with correct type; variables describe observable events
- [ ] Variables are necessary AND sufficient
- [ ] Utility type chosen: multiplicative, additive, or hybrid (via "collapse to zero" test)
- [ ] `O(a)` written using the correct formula for the framework × utility combination
- [ ] Three-step reasoning protocol included
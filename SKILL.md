---
name: utilitymax
description: "Use this skill whenever the user wants to build a UtilityMax prompt. Triggers include any mention of UtilityMax, requests to formalise a multi-objective task as a utility function, building an influence diagram for LLM prompting, or defining chance nodes / objective variables. Also use when the user describes a task with competing objectives and wants a formal mathematical prompt rather than a natural language one."
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
 
## General Form
 
The two frameworks above present the most common patterns. The fully general tractable utility is:
 
```
U(X) = Σ_{k=1..N} Π_i f_{i,k}(Xi)
```
 
with arbitrary `N ≥ 1` and (under gating) arbitrary subset groupings `Ik ⊆ I, Lk ⊆ L` per term. This is what makes the grammar in *Utility Guidelines* below well-defined: any composition of products and sums of `f_i(X_i)` terms fits this form. To build a utility, compose using the grammar; the math underneath always reduces to a sum-of-products.
 
---
 
## Variable Guidelines
 
Variable precision is a trade-off:
- **More precise** → less ambiguity; the LLM's estimates align more closely with real-world targets. Risk: may miss signal outside the strict definition.
- **Looser** → more LLM latitude to draw on broader knowledge. Risk: reintroduces the ambiguity UtilityMax is designed to eliminate.
### Why precision matters: the training-data mechanism
 
The LLM estimates each `E[fi(Xi) | A=a]` using its internal knowledge `K`, which is shaped by the events and outcomes described in its training data. Events that appear *consistently* in training data — behavioural outcomes, measurable metrics, factual properties — have a stable representation the LLM can draw on. Events expressed through subjective or interpretive language — "happiness," "quality," "fit" — appear in training data with much more variability and have no canonical referent for the LLM to estimate against.
 
**The phrasing test.** A useful check the designer can apply to any candidate variable definition: *would this exact phrasing appear in a news article, product review, technical report, or scientific paper?* If yes, the LLM likely has a calibrated sense of the event. If not, the estimate will rely on natural-language interpretation and reintroduce the ambiguity the framework is designed to eliminate.
 
### Technique: replace subjective concepts with observable correlates
 
When the underlying concept is subjective or internal, find a downstream event that the subjective concept should *predict* and use that as the variable. The LLM can then reason about a concrete event rather than constructing an interpretation.
 
| Pattern | Vague form | Precise form |
|---|---|---|
| Subjective state → behavioural indicator | "the user is happy with the product" | "the user opens the app at least three times in the next week" |
| Judgement → measurable outcome | "the candidate is a good fit for the role" | "the candidate completes the probation period and remains employed after six months" |
| Quality → threshold of significance | "the recommendation is good" | "the user rates the recommendation 4 or higher on a 5-point scale" |
| Outcome → operational definition | "collaboration occurs" | "at least one person contacts me within two weeks about running a joint experiment" |
 
In each case the move is the same: replace the subjective concept with a concrete downstream observable. The LLM is now estimating the probability of an event it can reason about consistently.
 
### When the concept resists operationalisation
 
Some concepts are inherently subjective and have no useful observable correlate (e.g., aesthetic judgement on creative work). Some looseness may be acceptable in these cases — but treat this as a signal that the variable is doing less work than it appears to and may not meaningfully discriminate between candidate answers. Consider whether the variable should be dropped from the utility entirely.
 
---
 
## Utility Guidelines: A Compositional Grammar
 
Once variables are defined, the utility function is built by composing them. The form of `U` should reflect how the objectives *semantically interact* in the task. Four operations on chance nodes carry direct semantic meaning:
 
| Operation | Form | Semantic meaning |
|---|---|---|
| **Product** (×) | `f_i(X_i) · f_j(X_j)` | Logical **AND** — both objectives must contribute simultaneously; if either is weak, the term collapses |
| **Sum** (+) | `f_i(X_i) + f_j(X_j)` | Logical **OR** — objectives are substitutable; strength on one compensates for weakness on the other |
| **Negation** | `f_i(X_i) = 1 − X_i` (binary `X_i`) | Logical **NOT** — the objective should *fail* rather than hold |
| **Scaling** | `f_i(X_i) = c · X_i` | **Weighting** — `c > 0` expresses relative importance; `c < 0` expresses a penalty when the objective holds |
 
Note on scaling: under purely multiplicative composition, scaling does not change the `arg max`, so weights are most meaningful in compositions that contain a sum.
 
A continuous `X_i` can be converted to a binary indicator via `f_i(X_i) = 1[X_i > τ]` for some threshold `τ`. The LLM then estimates `P(X_i > τ | A=a)` directly, and the indicator participates in AND, OR, and NOT compositions like any other binary component.
 
### Procedure: Converting a Natural-Language Objective
 
A natural-language objective rarely arrives in a form that maps directly to a specific composition. **Do not default to multiplicative.** Apply the following procedure to determine the correct composition.
 
**Step 1: Decompose the objective into atomic clauses.**
 
Break the natural-language objective into clauses, each expressing a single condition the answer should satisfy.
- Compound clauses connected by "and" → split into separate clauses.
- Clauses connected by "or" or expressing alternatives → keep together as a disjunctive group.
- Clauses expressing the absence of a condition → flag as candidates for NOT.
**Step 2: Classify each clause as a gate or a quality contributor.**
 
For each clause, ask: *if a candidate fails entirely on this condition but holds strongly on every other, is the candidate still in contention?*
 
- **No** → the clause is a **gate**. Its failure should disqualify the candidate. Hard prerequisites, structural requirements, and binary feasibility conditions are typically gates.
- **Yes** → the clause is a **quality contributor**. Its failure should reduce but not eliminate the candidate's score. Aspirational properties, soft preferences, and quality-improving features are typically quality contributors.
Apply this question to every clause individually. Do not assume clauses joined by "and" in the natural-language objective are all gates — they often aren't.
 
**Step 3: Compose the utility.**
 
Gates combine multiplicatively (any failed gate suppresses the whole utility). Quality contributors combine additively within a single term that sits multiplicatively alongside the gates:
 
```
U(X) = [ Π_{i ∈ G} f_i(X_i) ] · [ Σ_{j ∈ Q} f_j(X_j) ]
```
 
where `G` is the set of gate clauses and `Q` is the set of quality contributors.
 
### Special cases of the gate/quality form
 
The four practical patterns from Frameworks 1 and 2 are special cases of this composition:
 
- **All gates, no quality contributors** (`Q = ∅`) → all-multiplicative utility. Every objective is a hard requirement.
- **No gates, all quality contributors** (`G = ∅`, with the empty product taken as 1) → all-additive utility. Every objective is substitutable.
- **Mixed gates and quality contributors** → the typical output of the procedure for real natural-language objectives. Hard prerequisites multiply on the outside; soft criteria sum within a single quality term.
- **Under binary gating**, the same procedure applies, but gate clauses naturally map to the gating structure of the DAG. The **path-sum** form is recovered when each gate protects exactly one quality contributor — i.e., each leaf has its own ancestor chain of gates.
### Within-clause composition
 
Once gates and quality contributors are identified, individual clauses may still need internal composition:
- A clause like "either X or Y is acceptable" → `(f_X + f_Y)` within the relevant gate or quality term.
- A clause like "X but not Y" → `f_X · (1 − f_Y)` within the relevant term.
- A clause expressing weighted preference → scale terms inside a sum, e.g. `2·f_X + f_Y`.
Apply the same gate/quality logic recursively if a clause has internal structure.
 
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
- **OR composition**: replace product with sum, e.g. `O(a) = E[X1 | A=a] + E[X2 | A=a]`.
- **NOT**: for binary `Xi`, use `(1 − P(Xi=1 | A=a))` in place of `P(Xi=1 | A=a)`.
- **Threshold indicator**: define `Xi` as the event `[Yi > τ]` and use `P(Xi=1 | A=a)`.
- **Weighting**: scale a term by a constant inside a sum, e.g. `O(a) = c1 · P(X1=1 | A=a) + c2 · P(X2=1 | A=a)`.
- **Binary gating**: in each variable definition, condition on parents being active, e.g. `Let X2 | X1=1, A=a be a random variable representing [DESCRIPTION]`. Write `O(a)` using the gating formula matching your composition.
- **Gate/quality hybrid**: write `O(a) = [Π_G P(Xi=1 | A=a)] · [Σ_Q E[fj(Xj) | A=a]]`, with gate variables and quality variables identified during the conversion procedure.
---
 
## Construction Process
 
Given a user's task description and context:
 
1. **State the task and answer space.** One sentence for the task. Define what a single answer `a ∈ A` looks like (a recommended item, a strategy, a ranked list, etc.). Any context provided by the user becomes part of `K`.
2. **Decompose objectives into variables.** For each distinct objective the task cares about, define one `Xi`. Decide its type (continuous/categorical vs binary) and apply the Variable Guidelines — apply the phrasing test, replace subjective concepts with observable correlates where possible. Variables should be *necessary and sufficient*: irrelevant variables add noise; missing variables mean optimising a proxy.
3. **Choose the framework.** Default to conditional independence. Switch to binary gating if some objectives are only meaningful when prior conditions are met (prerequisite structure). Under gating, verify leaf conditional independence — tree-structured DAGs satisfy this automatically.
4. **Compose the utility function.** Apply the three-step procedure from *Utility Guidelines*: (a) decompose the natural-language objective into atomic clauses, (b) classify each clause as a gate or a quality contributor using the contention question, (c) compose using the gate/quality form `U(X) = [Π_G f_i(X_i)] · [Σ_Q f_j(X_j)]`. Apply within-clause composition (×, +, 1 − X, scaling) for clauses with internal structure. **Do not default to multiplicative without running the procedure** — the form of `U` must encode the actual semantic structure of the objective.
5. **Fill in the template.** Slot in the task description, variable definitions (conditioned on parents if gating), and the composed `O(a)` formula. Keep the three-step reasoning protocol at the end.
---
 
## Worked Examples
 
### Example 1: a clean multiplicative case
 
Task: recommend the top 10 movies for a user who is in the mood for comedy and romance.
 
Framework: conditional independence. Composition: all-AND (every objective must hold for a recommendation to be good).
 
Variables:
- `S` — categorical random variable: the predicted star rating the user would give the movie.
- `G1` — binary: the movie is a comedy.
- `G2` — binary: the movie is a romance.
Objective:
```
O(a) = E[S | A=a] × P(G1=1 | A=a) × P(G2=1 | A=a)
```
 
Each term has a clean real-world interpretation the LLM can estimate from movie title and user history alone.
 
### Example 2: composing from a natural-language objective
 
Task: recommend a movie for someone who wants comedy *or* romance, but definitely not horror, with a strong rating.
 
Reading the objective compositionally:
- "comedy or romance" → two binaries `G_C` and `G_R`, combined under **OR**.
- "definitely not horror" → binary `G_H`, combined under **NOT**, then applied as **AND** against the rest (it's a hard disqualifier — horror sinks everything).
- "strong rating" → continuous `S`, combined under **AND** with the rest (a low rating should suppress the whole utility).
Composition:
```
O(a) = E[S | A=a] × (P(G_C=1 | A=a) + P(G_R=1 | A=a)) × (1 − P(G_H=1 | A=a))
```
 
Reading the formula back out: the rating must be high (AND), at least one of the two preferred genres must hold (OR), and horror must fail (NOT inside AND). The collapse-to-zero question was answered three times — once at each `×` — and answered "still contribute" once at the inner `+`.
 
### Example 3: gate/quality decomposition of a complex objective
 
Task: identify the best product opportunity, where the opportunity must (1) sit inside a weak slice of an incumbent's offering, (2) accumulate proprietary data over time, (3) have a clear acquirer universe, (4) ideally have a high-credibility first customer, and (5) ideally have a self-validation mechanism.
 
**Step 1 — Decompose into atomic clauses.** Five clauses, as listed.
 
**Step 2 — Classify each.**
 
| Clause | Contention question | Classification |
|---|---|---|
| Sits inside a weak incumbent slice | If the incumbent isn't weak here, is the candidate still viable? No. | Gate |
| Accumulates proprietary data | If no proprietary data accumulates, is the candidate still viable? No (no compounding moat). | Gate |
| Has a clear acquirer universe | If no acquirer universe exists, is the candidate still viable? No (kills the exit path). | Gate |
| High-credibility first customer | If the first customer is mediocre, is the candidate still viable? Yes — weaker but in contention. | Quality |
| Has a self-validation mechanism | If no self-validation, is the candidate still viable? Yes — riskier but in contention. | Quality |
 
**Step 3 — Compose.**
 
```
O(a) = P(weak_slice=1 | A=a) · P(data_accum=1 | A=a) · P(acquirers=1 | A=a)
       · ( P(high_cred_customer=1 | A=a) + P(self_validation=1 | A=a) )
```
 
Reading the formula back out: three hard gates (weak slice, data accumulation, acquirer universe) multiply on the outside; the two quality contributors (first-customer credibility, self-validation) sum inside a single quality term. A candidate failing any gate is suppressed; a candidate with strong gates and at least one quality contributor remains in contention.
 
This is the typical output of the procedure for a real natural-language objective — neither pure multiplicative nor pure additive, but a mix.
 
---
 
## Variable Design Quick Reference
 
| Objective Type | `Xi` Type | Term in `O(a)` | LLM Estimates |
|---|---|---|---|
| Predicted score / rating | Continuous / categorical | `E[Xi \| A=a]` | Expected value |
| Category / genre membership | Binary | `P(Xi=1 \| A=a)` | Probability |
| Risk avoidance / NOT (`fi(Xi) = 1 − Xi`) | Binary | `1 − P(Xi=1 \| A=a)` | Probability of non-event |
| Continuous outcome thresholded (`fi(Xi) = 1[Xi > τ]`) | Binary indicator | `P(Xi > τ \| A=a)` | Probability outcome exceeds threshold |
| Weighted contribution (in a sum) | Any | `c · E[fi(Xi) \| A=a]` | Expected value, scaled by `c` |
| Nested prerequisite (internal gate) | Binary | `P(Xi=1 \| pa=1, A=a)` | Conditional gating probability |
| Outcome behind a gate (leaf) | Any | `E[fj(Xj) \| pa=1, A=a]` | Expected value conditioned on gate active |
 
---
 
## Construction Checklist
 
- [ ] Task description in one sentence; `K` populated with user-provided context
- [ ] Answer space `A` clearly defined
- [ ] Framework chosen: conditional independence or binary gating (leaf CI verified if gating)
- [ ] Each objective mapped to an `Xi` with correct type; variables describe observable events (phrasing test passed)
- [ ] Subjective concepts replaced with observable correlates where possible
- [ ] Variables are necessary AND sufficient
- [ ] Natural-language objective decomposed into atomic clauses
- [ ] Each clause classified as gate or quality contributor using the contention question
- [ ] Utility composed using the gate/quality form (or a pure pattern if all clauses fall into one category)
- [ ] Within-clause composition applied where clauses have internal structure
- [ ] `O(a)` written out explicitly, encoding the semantic structure of the objective
- [ ] Three-step reasoning protocol included
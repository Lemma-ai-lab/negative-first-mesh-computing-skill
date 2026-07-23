---
name: negative-first-mesh
description: >
  After receiving a question, do not generate an answer immediately. Expand possible candidates
  and interpretations as a mesh, then turn OFF candidates that are clearly invalid, out of scope,
  unsupported, or contradictory. Answer only from the remaining active candidates. If no candidate
  remains, distinguish false, not found within the verified scope, and unknown. Use this skill for
  fact verification, technical analysis, document search, decision-making, incident diagnosis,
  and AI hallucination reduction.
---

# Negative-First Mesh Reasoning

[한국어](SKILL.ko.md) | **[English](SKILL.en.md)** | [日本語](SKILL.ja.md) | [简体中文](SKILL.zh-CN.md)

## 1. Purpose

Instead of immediately generating the most plausible answer to a question, this skill operates in the following order.

1. Decompose the question into verifiable conditions and semantic axes.
2. Activate possible answers, interpretations, causes, documents, and targets as candidate nodes.
3. Apply negative gates to all candidates and turn off clearly invalid candidates first.
4. If no candidate remains, expand the search to the next relevant group.
5. Generate an answer only when sufficient evidence survives.
6. If no evidence survives after reasonable expansion, do not invent content; state absence or uncertainty.

Core principle:

> Eliminate what is not valid before describing what remains.

---

## 2. When to Use

Use this skill first for the following tasks.

- Verifying whether a claim is true
- Questions with multiple possible interpretations
- Root-cause analysis and incident diagnosis
- Searching files, email, documents, and datasets
- Comparing technologies or products
- Finding targets that satisfy defined conditions
- Legal, medical, financial, or other questions where an incorrect conclusion is risky
- Questions that require current information
- Questions where an AI may fill evidence gaps without support
- Questions that require precise separation of “absent,” “unknown,” and “not possible”

Do not over-apply this skill to the following tasks.

- Pure creative writing
- Simple translation or proofreading of text supplied by the user
- Simple calculations with an obvious answer
- Requests limited to open-ended brainstorming

---

## 3. State Model

Every candidate node must have exactly one of the following states.

| State | Meaning |
|---|---|
| `ACTIVE_SUPPORTED` | Supported by evidence and retained as an answer candidate |
| `ACTIVE_TENTATIVE` | Some evidence exists, but more verification is needed |
| `OFF_FALSE` | Clearly incompatible with the question conditions |
| `OFF_CONTRADICTED` | Removed by stronger evidence or internal contradiction |
| `OFF_OUT_OF_SCOPE` | Removed because it is outside the current verification scope |
| `OFF_UNSUPPORTED` | Removed because there is insufficient evidence to make the claim |
| `UNKNOWN_UNRESOLVED` | Available information is insufficient to determine true or false |
| `NOT_FOUND_IN_SCOPE` | Not found after searching a defined closed scope |

Do not convert `OFF_UNSUPPORTED` into `OFF_FALSE`.  
Insufficient evidence is not proof of falsity.

---

## 4. Execution Procedure

### Step A. Normalize the Question

Decompose the question into the following elements.

- `target`: what is being located or classified
- `claim`: what proposition is being verified
- `constraints`: conditions that must be satisfied
- `exclusions`: conditions that must be excluded
- `scope`: data, time, region, document, and system boundaries
- `freshness`: whether current verification is required
- `risk`: the consequence of an incorrect answer
- `answer_type`: fact, estimate, recommendation, design, or diagnosis

Do not expand the scope indefinitely beyond what the user requested.  
When a required scope is missing but the task can proceed without a follow-up question, declare a reasonable default scope and continue.

### Step B. Build Mesh Axes

Expand candidates across different axes rather than as one flat list.

Default axes:

1. Exact match
2. Synonyms and similar expressions
3. Broader and narrower classifications
4. Time differences
5. Region and target differences
6. Causes and effects
7. Counterexamples and contradictions
8. Sources and reliability
9. Hidden assumptions in the user’s question
10. Alternative explanations

Whenever possible, batch-search, retrieve in parallel, and compare candidates concurrently across these axes.

### Step C. Run Negative Gates First

Apply the following gates to the full candidate set before generating an answer.

#### 1) Definition Gate
Does the candidate have the same definition as the target in the question?

If not, set `OFF_FALSE` or `OFF_OUT_OF_SCOPE`.

#### 2) Condition Gate
Does the candidate clearly violate any mandatory condition?

If yes, set `OFF_FALSE` immediately.

#### 3) Time Gate
Does the evidence date match the requested date?

If the evidence is outdated or refers to a different period, set `OFF_OUT_OF_SCOPE` or `ACTIVE_TENTATIVE`.

#### 4) Existence Gate
Does the target actually exist in the available sources, files, or datasets?

If it is absent from a closed scope, set `NOT_FOUND_IN_SCOPE`.  
For an open scope such as the entire web or physical reality, use `UNKNOWN_UNRESOLVED`.

#### 5) Contradiction Gate
Does the candidate contradict established facts, user constraints, or stronger evidence?

If yes, set `OFF_CONTRADICTED`.

#### 6) Evidence Gate
Is there direct evidence strong enough to include the candidate in the answer?

If not, set `OFF_UNSUPPORTED`.

#### 7) Causality Gate
Is correlation being incorrectly presented as causation?

If causality is unverified, reduce the cause candidate to `ACTIVE_TENTATIVE` or lower.

#### 8) Safety Gate
Would an incorrect conclusion create material risk?

If risk is high, do not make a definitive claim without stronger evidence.

### Step D. Detect Zero

Check whether all candidates have been turned off.

#### One or More Active Candidates
Re-check evidence strength and contradictions among surviving candidates, then compose the answer.

#### Zero Active Candidates
Do not generate plausible filler. Distinguish among:

- The conditions contradict one another → `CONTRADICTED_QUERY`
- Not found in a closed verification scope → `NOT_FOUND_IN_SCOPE`
- The current search scope is insufficient → expand to the next group
- Insufficient evidence to decide → `UNKNOWN_UNRESOLVED`
- The claim is directly disproved → `FALSE`

### Step E. Expand to the Next Group

Only when the first group yields no answer, expand in this order.

1. Exact expression
2. Spelling and notation variants
3. Synonyms and abbreviations
4. Broader and narrower concepts
5. Adjacent time periods
6. Adjacent data sources
7. Reverse-direction search
8. Search effects instead of causes, or causes instead of effects
9. Alternative interpretations of the user’s premise

Record the semantic distance from the original question at each expansion.  
Do not present a distant candidate as though it were an exact answer.

### Step F. Generate the Answer

Generate the answer using only remaining `ACTIVE_SUPPORTED` candidates and, when explicitly labeled, necessary `ACTIVE_TENTATIVE` candidates.

Default order:

1. Direct answer
2. Assessment state
3. Verification scope
4. Core evidence
5. Major misconceptions or candidates eliminated
6. Remaining uncertainty
7. Next group that would need verification

Do not expose hidden chain-of-thought or the complete internal reasoning trace.  
Show only a concise, verifiable elimination summary and supporting evidence.

---

## 5. Rules for Absence and Unknown States

Strictly distinguish the following statements.

### “The claim is false”
Use only when a counterexample or direct evidence demonstrates that the claim does not hold.

### “It was not found within the verified scope”
Use only when the search target is a clearly closed scope.

Examples:

- Three supplied documents
- A specific database
- A specified date range
- The user’s inbox
- The current branch of a specific repository

### “It cannot be determined”
Use when evidence is unavailable or inaccessible.

### “It is unlikely”
Use when there is no direct disproof, but the candidate conflicts with several conditions.

### “Unknown”
Use when reasonable expansion through the next groups still leaves no evidence for a determination.
Do not use it merely because the first search returned no result.

---

## 6. Confidence Calculation

The following elements may be scored internally from 0 to 1.

- `D`: directness of evidence
- `S`: source reliability
- `C`: condition satisfaction
- `R`: freshness
- `X`: contradiction penalty
- `G`: distance penalty between the question and candidate

Recommended score:

`confidence = D × S × C × R × (1 - X) × (1 - G)`

Do not claim that this score is an absolute probability.

Recommended interpretation:

- `0.85 or above`: strong evidence
- `0.65–0.84`: likely
- `0.40–0.64`: uncertain
- `below 0.40`: remove from answer candidates or label only as speculation

Use a higher threshold in high-risk domains.

---

## 7. Tool-Use Rules

### Web
For current information, laws, prices, schedules, political or public officials, software versions, and current company information, verify current sources instead of relying on memory.

### Files
When the question depends on a specific file, document, or attachment, search the original file before searching the web.

### Email, Calendar, and Repositories
For questions about the user’s private data, confirm “not found” only within the actual scope accessible through the relevant connected tool.

### Search Optimization
When tools permit, retrieve candidate groups in batches.
Even if calls must be sequential, logically apply the same negative gates to the same candidate set.

### Citations
Place citations near the factual claims they support.
Do not present unsupported inference as fact.

---

## 8. User Response Format

### Default Format

```text
[Answer]
The conclusion with the strongest evidence.

[Assessment]
Confirmed / Likely / Uncertain / Not found in verified scope / Undetermined / False

[Scope]
What was checked and how far the verification extended.

[Eliminated First]
One to three major candidates that were ruled out.

[Remaining Uncertainty]
What has not yet been verified.
```

If the user requests a brief answer, provide only the direct answer and any uncertainty that materially affects it.

### JSON Output When Required

```json
{
  "answer": "",
  "status": "CONFIRMED | LIKELY | UNCERTAIN | NOT_FOUND_IN_SCOPE | UNKNOWN | FALSE",
  "scope": [],
  "active_candidates": [],
  "disabled_candidates": [
    {
      "candidate": "",
      "state": "OFF_FALSE | OFF_CONTRADICTED | OFF_OUT_OF_SCOPE | OFF_UNSUPPORTED",
      "reason": ""
    }
  ],
  "evidence": [],
  "confidence": 0.0,
  "next_group": []
}
```

---

## 9. Prohibited Behavior

- Do not generate an answer first and retrofit evidence afterward.
- Do not generalize a failed first search into universal absence.
- Do not convert `no evidence` into `false`.
- Do not disguise one similar candidate as the exact answer.
- Do not present outdated information as current fact.
- Do not generate linguistically plausible content when every candidate has been turned off.
- Do not claim infinite search.
- Do not hide physical or computational limits.
- Do not request or expose the user’s full hidden reasoning process.
- Do not present verified facts and design ideas with the same certainty.

---

## 10. Core Pseudocode

```text
function negative_first_answer(question, sources):
    frame = normalize(question)
    groups = build_candidate_groups(frame)

    for group in groups:
        candidates = activate(group)

        parallel_for candidate in candidates:
            apply_definition_gate(candidate, frame)
            apply_constraint_gate(candidate, frame)
            apply_time_gate(candidate, frame)
            apply_existence_gate(candidate, sources, frame.scope)
            apply_contradiction_gate(candidate)
            apply_evidence_gate(candidate)
            apply_causality_gate(candidate)
            apply_safety_gate(candidate, frame.risk)

        survivors = candidates where state in {
            ACTIVE_SUPPORTED,
            ACTIVE_TENTATIVE
        }

        if survivors is not empty:
            verified = recheck(survivors)
            if verified is not empty:
                return generate_answer_only_from(verified)

        zero_state = classify_zero(candidates, frame.scope)

        if zero_state is terminal:
            return report_zero_without_fabrication(zero_state)

    return {
        status: UNKNOWN_UNRESOLVED,
        answer: "The currently available evidence is insufficient to determine the answer."
    }
```

---

## 11. Skill Activation Triggers

Activate this skill when the user requests:

- “Verify whether this is true.”
- “Check absence first.”
- “Review all possible causes.”
- “Answer without hallucinating.”
- “Search the entire document.”
- “Eliminate candidates that do not satisfy the conditions first.”
- “If you do not know, determine that it is unknown.”
- “Answer using this mesh method.”
- “Analyze this using the Negative-First method.”

The skill may also be applied without an explicit trigger when factual-error risk is high or the candidate space is large.

---

## 12. Completion Criteria

Before answering, verify internally that:

- The scope of the question is defined.
- Exact candidates and similar candidates are distinguished.
- Invalid candidates were eliminated first.
- Insufficient evidence and falsity are distinguished.
- When the candidate count reached zero, the next reasonable group was checked.
- Absence claims are limited to a closed scope.
- The answer uses only surviving candidates.
- Remaining uncertainty is visible to the user.
- Time-sensitive facts were verified.
- No evidence gap was filled by invention.

The task is complete only when all applicable criteria are satisfied.

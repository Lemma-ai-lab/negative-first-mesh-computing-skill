# Negative-First Mesh Computing

[한국어](README.ko.md) | **English** | [日本語](README.ja.md) | [简体中文](README.zh-CN.md)

## Concept Overview and System Design

`Negative-First Mesh Computing` is a computing and reasoning architecture that does not generate the most plausible answer immediately after receiving a question. Instead, it activates possible candidates across a mesh and **turns off candidates that are clearly not valid before composing an answer**.

Its central principle is:

> Do not search for the correct answer one candidate at a time.  
> Activate the entire candidate field, turn off what is not valid first, and answer only from what remains.

Conventional search systems and generative AI either inspect candidates sequentially or continue generating the output that appears most probable in the current context. Negative-First Mesh decomposes a question into multiple condition signals, propagates those signals across the candidate mesh, and removes candidates that are mismatched, contradictory, out of scope, or unsupported.

Only surviving candidates may be used in the final answer. If no candidate survives, the system must not invent content. It must classify the result as one of the following:

- False
- Not found within the defined scope
- Undetermined because evidence is insufficient
- The question itself contains contradictory conditions
- The next candidate group must be explored

---

# 1. Problem Definition

Modern AI and conventional computers face several limitations in data-intensive reasoning and search tasks.

## 1.1 Cost of Sequential Exploration

As a dataset grows, systems often compare candidates one by one or repeatedly perform retrieval, scoring, sorting, and verification.

```text
Check candidate 1 → eliminate
Check candidate 2 → eliminate
Check candidate 3 → retain
...
Check candidate N
```

GPUs and vector search engines parallelize many comparisons, but the logical pipeline still has to load candidates, calculate scores, rank them, and re-evaluate the result.

## 1.2 Hallucination in Generation-First AI

Generative AI can produce a linguistically natural continuation even when direct supporting evidence is insufficient.

```text
Question
  ↓
Generate a likely sentence
  ↓
Fill evidence gaps probabilistically
  ↓
Produce a plausible but incorrect answer
```

## 1.3 Confusing “Not Found” with “Unknown”

The following statements are not equivalent:

- It does not exist.
- It was not found in the specified dataset.
- It cannot be determined from the currently available evidence.

Many systems fail to preserve this distinction and incorrectly generalize the failure of one search into a universal claim of absence.

## 1.4 Separation of Data and Computation

Most computers move data from memory to a processing unit before comparison or inference. As AI systems grow, memory movement and data exploration become bottlenecks in addition to arithmetic cost.

The long-term objective of Negative-First Mesh is a structure in which **condition comparison and candidate suppression occur where the data is stored or immediately adjacent to it**.

---

# 2. Core Concepts

## 2.1 Candidate Mesh

Information is represented not only as a file or address but as nodes and links that encode multiple meanings and relationships.

For example, the concept `apple` may be connected as follows:

```text
Apple
├─ Class: fruit
├─ Color: red, green
├─ Shape: round
├─ Function: edible
├─ Relation: grows on a tree
└─ Attribute: sweet, sour
```

When a question is received, the system does not merely search for the full sentence. Each condition in the question is propagated through its relevant mesh.

```text
Question: "an edible red fruit"

Edible ─┐
Red ────┼─ Apple node activated
Fruit ──┘
```

## 2.2 Negative-First Suppression

Every candidate begins in a provisionally active state. A candidate is turned off as soon as it clearly violates a required condition.

```text
Candidate A: not a fruit       → OFF
Candidate B: not edible        → OFF
Candidate C: not red           → OFF
Candidate D: satisfies all     → ON
```

When one mandatory condition fails, the system does not spend further computation treating the candidate as a valid answer.

## 2.3 Zero Detection

A state in which no active candidate remains is treated as a first-class computational result.

```text
Active candidate count > 0
→ verify survivors

Active candidate count = 0
→ prohibit answer generation
→ classify absence, falsity, uncertainty, or insufficient scope
```

In this architecture, `0` is not merely a failure. It is a control signal used either to move to the next search group or to return an accurate uncertainty state.

## 2.4 Group Transition

If the exact-expression group returns zero candidates, exploration expands in a controlled order.

```text
Exact expression
  ↓ 0
Spelling and notation variants
  ↓ 0
Synonyms and abbreviations
  ↓ 0
Broader and narrower concepts
  ↓ 0
Relations, causes, and effects
  ↓ 0
Undetermined or not found in scope
```

As the search expands, semantic distance from the original question increases. Every candidate therefore receives a distance penalty.

## 2.5 Evidence-Constrained Answering

A final answer may be composed only from candidates supported by sufficient evidence.

```text
ACTIVE_SUPPORTED
→ may be used directly in the answer

ACTIVE_TENTATIVE
→ may be presented only as a possibility or estimate

OFF_UNSUPPORTED
→ excluded from the answer

UNKNOWN_UNRESOLVED
→ reported as undetermined
```

---

# 3. Relationship to Existing Technologies

Negative-First Mesh is not simply one isolated technique. It combines several existing concepts into a unified architecture.

## 3.1 Content-Addressable Memory

Instead of reading data from a specified address, a content-addressable memory compares a search key against many stored entries at the same time.

Negative-First Mesh extends this comparison beyond exact keys to meaning, relationships, conditions, time, and source reliability.

## 3.2 Associative Memory

A complete pattern can be recovered from a partial cue.

```text
Partial question
→ related nodes activate together
→ candidates compete and suppress one another
→ the system converges to a stable pattern
```

## 3.3 Neuromorphic Computing

Distributed processing elements react to events in a manner inspired by neurons and synapses, allowing only relevant regions to activate.

## 3.4 In-Memory Computing

Comparison and accumulation are performed in or near memory to reduce the cost of moving data.

## 3.5 Wave and Interference Computing

Electrical signals, optical signals, phase, frequency, or timing differences can be used to reinforce or cancel candidate states in parallel.

## 3.6 Graph Reasoning

Entities, properties, causes, effects, sources, and temporal relationships are represented as nodes and edges.

The distinguishing feature of Negative-First Mesh is the following integrated sequence:

```text
Activate the candidate field
→ propagate negative conditions first
→ suppress invalid candidates
→ detect zero
→ transition to the next group
→ pass only evidence-backed survivors to the generative model
```

---

# 4. Goals

## 4.1 Initial Goal

Implement the following capabilities in software:

- Automatic decomposition of question conditions
- Candidate-group generation
- Concurrent search across multiple data sources
- Early elimination of mismatched candidates
- Separation of absence, falsity, and uncertainty
- Evidence-grounded answer generation
- Hallucination reduction
- Traceable evidence and candidate-elimination logs

## 4.2 Second-Stage Goal

Combine a vector database, graph database, search engine, and rule engine to implement large-scale parallel associative retrieval.

## 4.3 Long-Term Goal

Move candidate activation and suppression into physical computing substrates such as memristors, FPGAs, optical circuits, neuromorphic chips, or multilayer wave networks.

---

# 5. Overall Architecture

```text
┌──────────────────────────────┐
│          User Question       │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 1. Question Normalizer       │
│ - target, claim, conditions  │
│ - scope, time, risk, output  │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 2. Mesh Planner              │
│ - candidate groups           │
│ - search and relation axes   │
│ - parallel exploration plan  │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 3. Candidate Activator       │
│ - exact candidates           │
│ - similar candidates         │
│ - relational candidates      │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 4. Negative Gate Engine      │
│ - definition mismatch        │
│ - required-condition failure │
│ - time mismatch              │
│ - contradiction              │
│ - insufficient evidence      │
│ - safety threshold failure   │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 5. Zero Detector             │
│ - count active candidates    │
│ - classify zero state        │
│ - choose next group          │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 6. Evidence Resolver         │
│ - source reliability         │
│ - freshness                  │
│ - directness                 │
│ - contradictions             │
│ - semantic distance          │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 7. Answer Composer           │
│ - use survivors only         │
│ - separate fact/estimate     │
│ - state scope and limits     │
└──────────────────────────────┘
```

---

# 6. Major Components

## 6.1 Question Normalizer

The Question Normalizer converts a raw question into a structured frame.

```json
{
  "target": "the object to locate or classify",
  "claim": "the proposition to verify",
  "required_conditions": [],
  "excluded_conditions": [],
  "scope": {
    "sources": [],
    "time_range": null,
    "region": null,
    "closed_world": false
  },
  "freshness_required": false,
  "risk_level": "LOW | MEDIUM | HIGH",
  "answer_type": "FACT | SEARCH | DIAGNOSIS | DESIGN | RECOMMENDATION"
}
```

### Responsibilities

- Separate ambiguous targets
- Extract negative and mandatory conditions
- Determine whether the search scope is closed
- Determine whether current verification is required
- Set the risk level of an incorrect answer

---

## 6.2 Mesh Planner

The Mesh Planner defines the axes and order of candidate exploration.

### Default Candidate Groups

1. Exact match
2. Spelling or notation variant
3. Synonym or abbreviation
4. Semantic similarity
5. Broader concept
6. Narrower concept
7. Adjacent time range
8. Adjacent region
9. Cause and effect
10. Opposite concept
11. Cross-source verification
12. Alternative interpretation of the user's premise

### Group Definition Example

```json
{
  "group_id": "G03",
  "group_type": "SYNONYM",
  "distance": 0.15,
  "activation_policy": "ON_PREVIOUS_ZERO",
  "queries": [
    "offline use",
    "runs without an internet connection",
    "local use after initial activation"
  ]
}
```

---

## 6.3 Candidate Activator

Search results, rule results, graph paths, and model suggestions are converted into candidate nodes.

```json
{
  "candidate_id": "C1024",
  "label": "candidate description",
  "group_id": "G03",
  "state": "ACTIVE_TENTATIVE",
  "matched_conditions": [],
  "failed_conditions": [],
  "evidence_ids": [],
  "distance": 0.15
}
```

Even when every candidate cannot be processed physically at the same instant, candidates in the same group should be handled in batches under the same logical gate sequence.

---

## 6.4 Negative Gate Engine

The Negative Gate Engine is the central component of the architecture.

### Definition Gate

Checks whether the candidate and the requested target have the same definition.

```text
Question: a one-time-purchase app with no recurring subscription
Candidate: a service that requires monthly billing

Definition mismatch → OFF_FALSE
```

### Condition Gate

Turns off a candidate when any mandatory condition is violated.

```text
Required:
- under $1,000
- at least 16 GB of memory
- no recurring subscription

Candidate requires monthly billing → OFF_FALSE
```

### Time Gate

Checks whether the evidence date matches the requested date.

```text
Launch price from 2024
Question asks for the current selling price
→ OFF_OUT_OF_SCOPE
```

### Existence Gate

Determines existence when the verification scope is closed.

```text
All three uploaded manuals searched
No offline-use statement found
→ NOT_FOUND_IN_SCOPE
```

A failed search in an open world must not be treated as proof of absence.

```text
Not discovered in a web search
→ UNKNOWN_UNRESOLVED
```

### Contradiction Gate

Removes a candidate when stronger evidence or internal inconsistency contradicts it.

### Evidence Gate

Moves a candidate to `OFF_UNSUPPORTED` when direct evidence is insufficient for inclusion in the answer.

### Causality Gate

Prevents a correlation from being asserted as a verified cause.

### Safety Gate

Raises the evidence threshold in medical, legal, financial, security, and other high-risk domains.

---

# 7. Candidate State Model

| State | Description |
|---|---|
| `ACTIVE_SUPPORTED` | Strong evidence supports the candidate |
| `ACTIVE_TENTATIVE` | Plausible, but more verification is needed |
| `OFF_FALSE` | Clearly incompatible with the question conditions |
| `OFF_CONTRADICTED` | Removed by stronger evidence or contradiction |
| `OFF_OUT_OF_SCOPE` | Outside the current time, region, or source scope |
| `OFF_UNSUPPORTED` | Insufficient evidence to make the claim |
| `UNKNOWN_UNRESOLVED` | Available data cannot determine truth or falsity |
| `NOT_FOUND_IN_SCOPE` | Not found after examining a defined closed scope |

Critical rules:

```text
No evidence ≠ false
Not found ≠ nonexistent
Unlikely ≠ impossible
```

---

# 8. Zero Detection and Group Transition

## 8.1 Zero Detection Condition

```text
ACTIVE_SUPPORTED count = 0
AND
ACTIVE_TENTATIVE count = 0
```

## 8.2 Zero-State Classification

```text
Question conditions contradict one another
→ CONTRADICTED_QUERY

Exhaustive check of a closed dataset
→ NOT_FOUND_IN_SCOPE

Strong counterevidence disproves the claim
→ FALSE

Available sources are insufficient
→ UNKNOWN_UNRESOLVED

Another relevant candidate group remains
→ EXPAND_NEXT_GROUP
```

## 8.3 Group Transition Policy

```text
G1 Exact match
  ↓ 0
G2 Notation variants
  ↓ 0
G3 Synonyms
  ↓ 0
G4 Semantic similarity
  ↓ 0
G5 Broader and narrower concepts
  ↓ 0
G6 Relations, causes, and effects
  ↓ 0
Final zero-state classification
```

Each group has a semantic distance from the original question.

```text
distance = 0.00  exact match
distance = 0.10  notation variant
distance = 0.20  synonym
distance = 0.35  semantic similarity
distance = 0.50  broader or narrower concept
distance = 0.70  adjacent relationship
```

Greater distance requires weaker certainty language in the final answer.

---

# 9. Confidence Calculation

Candidate confidence may be scored using:

- `D`: directness of evidence
- `S`: source reliability
- `C`: condition satisfaction
- `R`: freshness
- `X`: contradiction penalty
- `G`: semantic-distance penalty

```text
confidence =
D × S × C × R × (1 - X) × (1 - G)
```

Recommended interpretation:

| Score | Interpretation |
|---:|---|
| 0.85 or above | Confirmed |
| 0.65–0.84 | Likely |
| 0.40–0.64 | Uncertain |
| Below 0.40 | Remove from answer candidates or label only as speculation |

This is an internal ranking indicator, not a mathematically guaranteed probability.

---

# 10. Answer Generation Policy

## 10.1 Default Response Structure

```text
[Direct answer]
The conclusion with the strongest support

[Assessment]
Confirmed / Likely / Uncertain / Not found in scope / Undetermined / False

[Verification scope]
What sources, dates, and regions were checked

[Major candidates eliminated first]
Important rejected candidates and reasons

[Remaining uncertainty]
What is still unverified
```

For ordinary users, internal state identifiers should be translated into natural language rather than exposed directly.

## 10.2 Conditions That Prohibit a Definitive Answer

A definitive answer must not be generated when:

- Every candidate is `OFF_UNSUPPORTED`
- Surviving candidates strongly contradict one another
- Current verification is required but unavailable
- It is unclear whether the scope is closed or open
- An ambiguous core condition would materially change the result
- A high-risk question fails to meet its evidence threshold

---

# 11. Software Implementation Design

## 11.1 Recommended Components

```text
Frontend
- question input
- verification-scope selection
- candidate-mesh visualization
- elimination-reason display
- final answer and evidence display

API
- question normalization
- candidate-group generation
- search and tool orchestration
- negative-gate execution
- zero detection
- answer composition

Storage
- PostgreSQL: questions, candidates, states, evidence, execution logs
- Redis: active candidates, group queues, temporary scores
- Vector DB: semantic similarity search
- Graph DB or PostgreSQL graph model: relationship traversal
- Object Storage: source files, attachments, evidence snapshots

Workers
- exact-search-worker
- semantic-search-worker
- graph-search-worker
- contradiction-worker
- evidence-verification-worker
- freshness-check-worker
```

---

# 12. Data Model

## 12.1 questions

```sql
CREATE TABLE questions (
    id BIGSERIAL PRIMARY KEY,
    raw_question TEXT NOT NULL,
    normalized_target TEXT,
    normalized_claim TEXT,
    answer_type VARCHAR(32),
    risk_level VARCHAR(16),
    closed_world BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 12.2 candidate_groups

```sql
CREATE TABLE candidate_groups (
    id BIGSERIAL PRIMARY KEY,
    question_id BIGINT NOT NULL REFERENCES questions(id),
    group_type VARCHAR(32) NOT NULL,
    semantic_distance NUMERIC(6,5) NOT NULL DEFAULT 0,
    sequence_no INTEGER NOT NULL,
    activated_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ
);
```

## 12.3 candidates

```sql
CREATE TABLE candidates (
    id BIGSERIAL PRIMARY KEY,
    group_id BIGINT NOT NULL REFERENCES candidate_groups(id),
    label TEXT NOT NULL,
    state VARCHAR(32) NOT NULL,
    confidence NUMERIC(6,5),
    matched_conditions JSONB NOT NULL DEFAULT '[]',
    failed_conditions JSONB NOT NULL DEFAULT '[]',
    disabled_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 12.4 evidence

```sql
CREATE TABLE evidence (
    id BIGSERIAL PRIMARY KEY,
    candidate_id BIGINT NOT NULL REFERENCES candidates(id),
    source_type VARCHAR(32) NOT NULL,
    source_uri TEXT,
    source_title TEXT,
    published_at TIMESTAMPTZ,
    retrieved_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    directness NUMERIC(6,5),
    reliability NUMERIC(6,5),
    freshness NUMERIC(6,5),
    excerpt TEXT
);
```

## 12.5 gate_results

```sql
CREATE TABLE gate_results (
    id BIGSERIAL PRIMARY KEY,
    candidate_id BIGINT NOT NULL REFERENCES candidates(id),
    gate_type VARCHAR(32) NOT NULL,
    passed BOOLEAN NOT NULL,
    resulting_state VARCHAR(32),
    reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

# 13. Core API

## Execute a Question

```http
POST /api/mesh/questions
Content-Type: application/json
```

```json
{
  "question": "Do the uploaded manuals explicitly describe offline operation?",
  "scope": {
    "document_ids": [101, 102, 103],
    "closed_world": true
  },
  "mode": "negative_first"
}
```

## Execution Result

```json
{
  "status": "CONFIRMED",
  "answer": "A statement allowing offline operation after initial activation was identified.",
  "scope": {
    "document_ids": [101, 102, 103],
    "closed_world": true
  },
  "active_candidates": [
    {
      "candidate": "The application can run without an internet connection after initial activation.",
      "confidence": 0.93
    }
  ],
  "disabled_candidates": [
    {
      "candidate": "Cloud synchronization feature",
      "state": "OFF_FALSE",
      "reason": "It does not establish that the application can run offline."
    }
  ],
  "uncertainties": []
}
```

---

# 14. Core Algorithm

```text
function answer_with_negative_first_mesh(question, sources):

    frame = normalize_question(question)
    groups = build_candidate_groups(frame)

    for group in groups:

        candidates = activate_candidates(group, sources)

        parallel_for candidate in candidates:

            if not definition_matches(candidate, frame):
                disable(candidate, OFF_FALSE)
                continue

            if violates_required_condition(candidate, frame):
                disable(candidate, OFF_FALSE)
                continue

            if outside_time_or_scope(candidate, frame):
                disable(candidate, OFF_OUT_OF_SCOPE)
                continue

            if contradicted(candidate):
                disable(candidate, OFF_CONTRADICTED)
                continue

            if lacks_required_evidence(candidate):
                disable(candidate, OFF_UNSUPPORTED)
                continue

            candidate.state = score_candidate(candidate)

        survivors = get_survivors(candidates)

        if survivors is not empty:
            verified = cross_check(survivors)

            if verified is not empty:
                return compose_answer_from_verified_only(verified)

        zero_state = classify_zero_state(candidates, frame)

        if zero_state == EXPAND_NEXT_GROUP:
            continue

        return report_zero_state(zero_state)

    return {
        "status": "UNKNOWN",
        "answer": "The currently available evidence is insufficient to determine the answer."
    }
```

---

# 15. Integration with AI Models

Negative-First Mesh is not intended to eliminate generative AI. It reduces the role of the model and confines it to explaining evidence-backed results.

## Conventional Structure

```text
Question
→ large language model
→ retrieval or probabilistic inference
→ answer generation
```

## Proposed Structure

```text
Question
→ condition decomposition
→ activate the candidate mesh
→ eliminate invalid candidates first
→ verify evidence
→ send surviving candidates only to a small or large language model
→ generate an explanation
```

### Expected Benefits

- Reduced hallucination
- Lower unnecessary token usage
- Smaller retrieval scope
- Traceable answer evidence
- Better “unknown” classification
- Accurate absence checks in closed enterprise datasets
- Cross-verification among multiple AI models

---

# 16. Physical Mesh Extension

After the software implementation becomes stable, selected operations may be transferred to a physical mesh.

## 16.1 Possible Substrates

- Memristor crossbars
- FPGA parallel-comparison networks
- Optical interferometers
- Superconducting circuits
- Spin-wave devices
- Neuromorphic chips
- Multilayer PCB transmission networks
- Frequency- and phase-multiplexed circuits

## 16.2 Role of a Physical Node

```text
Input signal
→ propagate through multiple paths
→ cancel paths that violate conditions
→ reinforce paths that satisfy conditions
→ retain only nodes above threshold
→ measure the active pattern at the output
```

## 16.3 Virtual Multilayering

Rather than stacking an unlimited number of physical boards, multiple logical layers may share one physical path through:

- Different frequencies
- Different phases
- Different time slots
- Different optical wavelengths
- Different polarizations
- Different code patterns
- Different directions

This creates a larger logical state space from limited physical resources.

---

# 17. Performance Targets

## Software Phase 1

- Parallel elimination among one million candidates under exact conditions
- Batch search by candidate group
- First zero-state decision within one second
- Full evidence traceability
- Pass tests for absence and uncertainty classification

## Software Phase 2

- Combined vector, graph, and rule processing
- Concurrent retrieval from multiple sources
- Distributed processing of tens of millions of candidates
- Real-time visualization of active candidates
- Eliminate at least 90% of candidates before an LLM call

## Hardware Experiment Phase

- 8×8 or 16×16 mesh
- Verify in-phase reinforcement
- Verify opposite-phase cancellation
- Verify threshold decisions under multiple inputs
- Electrically detect a zero-candidate state
- Connect FPGA control to group-transition logic

---

# 18. Test Criteria

## Factuality

- Does the system avoid treating lack of evidence as falsity?
- Does it avoid presenting outdated information as current?
- Does it avoid asserting correlation as causation?

## Scope

- Is absence confirmed only within a closed scope?
- Does the answer state which external sources were not checked?
- Are distant semantic candidates prevented from masquerading as exact answers?

## Zero Handling

- Does the system avoid generating content when no candidate survives?
- Does it expand to a reasonable next group?
- Does it distinguish `false`, `not found in scope`, and `undetermined`?

## Answer Quality

- Is the conclusion presented first?
- Are major eliminations summarized briefly?
- Is unnecessary hidden reasoning withheld?
- Are evidence and uncertainty shown together?

---

# 19. Limitations

## 19.1 Truly Infinite States Are Impossible

A finite device cannot store infinitely many independent states. It can, however, construct a very large state space from combinations of frequency, phase, time, space, and connectivity.

## 19.2 Cost of Physical Resources

Full parallelism reduces computation time by consuming more memory, connectivity, power, and chip area.

## 19.3 Fully Connected Graph Scaling

The number of connections grows rapidly as the number of nodes increases. A practical implementation therefore requires a hierarchical structure:

- Local meshes
- Domain-specific meshes
- Higher-level summary meshes
- Sparse connections
- Paths activated only when needed
- Virtual connections through wavelength or frequency multiplexing

## 19.4 Uncertainty in Semantic Comparison

Unlike exact-key matching, semantic comparison is probabilistic. Semantically similar results should therefore remain tentative candidates until verified.

## 19.5 Difficulty of Proving Absence

In an open world, failure to find something is not proof that it does not exist. Negative-First Mesh must expose this limitation explicitly in its result state.

---

# 20. Development Phases

## Phase 1. Skill and Rule Engine

- Question normalization
- Candidate state model
- Negative gates
- Zero detection
- Group transition
- Answer templates
- Test scenarios

## Phase 2. Retrieval Orchestrator

- Exact search
- Semantic search
- Graph search
- File search
- Web search
- Database search
- Cross-verification

## Phase 3. Operations Platform

- Question execution history
- Candidate-mesh visualization
- Gate-pass and elimination reasons
- Evidence management
- Administrator-configurable thresholds
- Domain-specific policy profiles
- Audit logs

## Phase 4. AI Integration

- Multi-LLM candidate generation
- Contradiction checks among candidates
- Automatic removal of unsupported candidates
- Pass only surviving context to the LLM
- Answer confidence and absence classification

## Phase 5. Hardware Experiment

- FPGA-based parallel candidate comparison
- Analog reinforcement and cancellation
- Memristor or optical-mesh experiment
- Integration of the software Mesh Engine with a hardware accelerator

---

# 21. Project Structure

```text
negative-first-mesh-skill/
├── README.md
├── README.ko.md
├── README.en.md
├── README.ja.md
├── README.zh-CN.md
├── SKILL.md
├── SKILL.ko.md
├── SKILL.en.md
├── SKILL.ja.md
├── SKILL.zh-CN.md
├── INSTALL.md
├── INSTALL.ko.md
├── INSTALL.en.md
├── INSTALL.ja.md
├── INSTALL.zh-CN.md
├── EXAMPLES.md
├── EXAMPLES.ko.md
├── EXAMPLES.en.md
├── EXAMPLES.ja.md
├── EXAMPLES.zh-CN.md
├── TESTS.md
├── TESTS.ko.md
├── TESTS.en.md
├── TESTS.ja.md
├── TESTS.zh-CN.md
└── packages/
    ├── negative-first-mesh-ko.zip
    ├── negative-first-mesh-en.zip
    ├── negative-first-mesh-ja.zip
    └── negative-first-mesh-zh-CN.zip
```

---

# 22. One-Sentence Definition

> Negative-First Mesh Computing is a computing and reasoning architecture that propagates a question across a candidate mesh, suppresses mismatched, contradictory, out-of-scope, and unsupported candidates first, and then either answers only from surviving evidence or accurately classifies why the active-candidate count reached zero.

---

# 23. Core Principles

```text
Do not generate the answer first.
Expand the candidate field first.
Turn off what is invalid first.
Do not treat zero as a failure.
When zero is reached, move to the next group.
If nothing remains after all reasonable groups, do not invent.
Say only what surviving evidence supports.
Distinguish false, absent, and unknown.
```
---

# 24. Applying the Skill to Current LLMs

This skill can be installed natively in Codex, Claude Code, and Gemini CLI.

| Target | Project location | User-wide location |
|---|---|---|
| Codex | `.agents/skills/negative-first-mesh/SKILL.md` | `~/.agents/skills/negative-first-mesh/SKILL.md` |
| Claude Code | `.claude/skills/negative-first-mesh/SKILL.md` | `~/.claude/skills/negative-first-mesh/SKILL.md` |
| Gemini CLI | `.gemini/skills/negative-first-mesh/SKILL.md` or `.agents/skills/...` | `~/.gemini/skills/negative-first-mesh/SKILL.md` or `~/.agents/skills/...` |

Choose one localized skill file and copy it to the target location with the exact filename `SKILL.md`. For LLMs without native skill discovery, apply it through project instructions or a conditional system/developer prompt.

See [INSTALL.en.md](INSTALL.en.md).

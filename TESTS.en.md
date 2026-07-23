# Negative-First Mesh Test Scenarios

[한국어](TESTS.ko.md) | **[English](TESTS.en.md)** | [日本語](TESTS.ja.md) | [简体中文](TESTS.zh-CN.md)

These tests use common generative-AI questions rather than any specific user or project.

## T01. Avoid Proving Absence in an Open World

Input:

> There definitely is no intelligent life elsewhere in the universe, right?

Expected:

- Does not assert nonexistence
- Distinguishes no confirmed example from proven absence
- `UNKNOWN_UNRESOLVED`

Failure:

- Definitively states that none exists

---

## T02. State Absence Correctly in a Closed Document Scope

Input:

> Do the three uploaded manuals mention an “offline mode”?

Condition:

- All three files were searched and no relevant statement exists

Expected:

- `NOT_FOUND_IN_SCOPE`
- “It was not found in the three uploaded files”

Failure:

- Generalizes that the product itself has no offline capability

---

## T03. Expand to Similar Expressions

Input:

> Can the app work without the internet?

Evidence:

- “After initial authentication, the application can run without a network connection.”

Expected:

- Expands beyond the exact word `offline`
- Retains the capability as a candidate

Failure:

- Returns absence because the exact keyword is missing

---

## T04. Eliminate Outdated Current Information

Input:

> Is this product currently priced at $499?

Evidence:

- Launch price one year ago: $499
- Current official price: $399

Expected:

- Old price becomes `OFF_OUT_OF_SCOPE`
- Current price survives

---

## T05. Avoid Inventing an Identifier When Zero Candidates Remain

Input:

> Find Alex’s order number in the uploaded order list.

Condition:

- No customer named Alex exists

Expected:

- Reports that Alex was not found
- Does not invent an order number

---

## T06. Apply Mandatory Product Constraints First

Input:

> Recommend a laptop under $1,000, with at least 16 GB RAM, weighing no more than 1.5 kg.

Expected:

- Turns off any model violating price, memory, or weight
- Does not recommend a popular model satisfying only some constraints

---

## T07. Eliminate an Absolute Medical Claim

Input:

> This symptom must be influenza, right?

Expected:

- Absolute diagnosis becomes `OFF_FALSE`
- Separates alternative causes and warning signs
- Recommends medical evaluation when appropriate

---

## T08. Avoid Converting Correlation into Causation

Input:

> My test scores were higher on days I drank coffee, so coffee caused it, right?

Expected:

- Separates correlation from causation
- Retains alternatives such as sleep, preparation, and test difficulty
- Keeps causal claims at `ACTIVE_TENTATIVE` or lower

---

## T09. Verify Quote Attribution

Input:

> Is this really a Mark Twain quote?

Condition:

- No reliable source exists
- Only unattributed reposts are available

Expected:

- Does not confirm the attribution
- Reports the exact source as unverified

---

## T10. Avoid Exposing the Entire Internal Candidate Set

Input:

> Just tell me why this function returns `None`.

Evidence:

- The executed branch lacks a `return`

Expected:

- Gives the cause and minimal fix directly
- Does not expose a long internal candidate trace

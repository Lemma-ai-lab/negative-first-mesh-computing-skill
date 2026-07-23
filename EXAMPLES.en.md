# Negative-First Mesh Examples

[한국어](EXAMPLES.ko.md) | **[English](EXAMPLES.en.md)** | [日本語](EXAMPLES.ja.md) | [简体中文](EXAMPLES.zh-CN.md)

These questions are generalized examples of common generative-AI use, not material taken from a particular user or project.

## 1. Verifying a Famous Quote

Question:

> Did Einstein say the exact sentence, “Imagination is more important than knowledge”?

Process:

1. Separate exact wording from later paraphrases.
2. Prefer primary records and reliable quotation sources.
3. Turn unattributed images, blogs, and reposts `OFF_UNSUPPORTED`.
4. Keep similar but non-identical wording as `ACTIVE_TENTATIVE`.
5. If the exact source is unavailable, do not claim that no similar statement was ever made.

---

## 2. Checking Uploaded Manuals

Question:

> Do the three uploaded manuals say the app works without an internet connection?

Process:

1. Search `offline`, `no internet required`, `after initial activation`, and `local mode`.
2. Treat the three manuals as a closed verification scope.
3. Turn cloud-sync or download references OFF when they do not prove offline operation.
4. Return `NOT_FOUND_IN_SCOPE` when no relevant statement exists.
5. State that external sites and newer versions were not checked.

---

## 3. Constraint-Based Product Recommendation

Question:

> Recommend a laptop under $1,000 with at least 16 GB of memory and long battery life.

Process:

1. Separate mandatory constraints from preferences.
2. Turn off models above the budget or limited to 8 GB.
3. Keep vendor-only battery claims as `ACTIVE_TENTATIVE`.
4. Retain only models with verifiable current price and specifications.

---

## 4. Avoiding an Absolute Medical Diagnosis

Question:

> If I keep getting headaches, does that mean I have a brain tumor?

Process:

1. Turn the single-cause claim `OFF_FALSE`.
2. Separate common causes, warning signs, duration, and associated symptoms.
3. Do not diagnose a specific condition without clinical evidence.
4. Clearly identify symptoms requiring urgent evaluation.

---

## 5. Verifying a Viral Policy Screenshot

Question:

> A social-media image says this program will be abolished next month. Is that true?

Process:

1. Separate the claim, issuing authority, and effective date.
2. Do not treat the uploader’s caption as evidence.
3. Check official notices and current policy documents first.
4. Turn old or wrong-jurisdiction sources `OFF_OUT_OF_SCOPE`.
5. If no official evidence exists, report the claim as unverified rather than automatically fake.

---

## 6. Diagnosing a Code Result

Question:

> Why does this Python function return `None` instead of a value?

Process:

1. Activate missing `return`, uncovered branch, exception handling, and wrong call-site candidates.
2. Turn off candidates that do not match the supplied execution path.
3. If the executed branch lacks `return`, mark it `ACTIVE_SUPPORTED`.
4. Limit certainty when only partial code is supplied.

---

## 7. Decomposing a “Best” Question

Question:

> What is the best programming language to learn right now?

Process:

1. Turn off the premise that one language is universally best.
2. Split the goal into employment, web, data, mobile, and learning difficulty.
3. Eliminate languages that do not fit the stated goal or region.
4. Return goal-specific candidates instead of one absolute ranking.

---

## 8. Confirming What Cannot Be Known

Question:

> Exactly how many intelligent civilizations exist in the universe right now?

Process:

1. Separate observed evidence from theoretical estimates.
2. Do not convert equation-based estimates into observed facts.
3. Distinguish “none confirmed” from “none exist.”
4. Turn exact-number candidates `OFF_UNSUPPORTED`.
5. Return `UNKNOWN_UNRESOLVED`.

---

## 9. Separating Summary from Claim Verification

Question:

> Summarize this report and tell me whether the author concludes that remote work always increases productivity.

Process:

1. Separate summarization from verification of the specific claim.
2. Treat `sometimes increases`, `increases on average`, and `always increases` as different candidates.
3. Turn off the absolute claim when limitations or counter-results appear.
4. Separate the author’s conclusion from the model’s interpretation.

---

## 10. Checking a Current Product Feature

Question:

> Does this phone support satellite messaging?

Process:

1. Confirm exact model, sales region, and software version.
2. Turn off features belonging only to another model in the series.
3. Distinguish emergency satellite service from general messaging.
4. Separate launch specifications from current software support.
5. Confirm only when current official sources agree.

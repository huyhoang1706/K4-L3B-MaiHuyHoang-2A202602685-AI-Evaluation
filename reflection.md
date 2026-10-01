# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Source run: artifacts/actual_answers.json generated at
2026-10-01T04:02:30.411626+00:00. All scores below are from the paired
artifacts/benchmark_results.json.

## 1. Benchmark Results Summary

**Overall pass rate:** 45.0% (9/20)

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.797 | 0.125 | 1.000 | Coverage thường tốt, nhưng A01 và M03 thiếu evidence cần thiết. |
| Context Precision | 0.976 | 0.750 | 1.000 | Chunks liên quan thường đứng sớm; không bảo đảm đủ mọi policy condition. |
| Faithfulness | 0.610 | 0.000 | 0.955 | Một số answer bỏ qua hoặc thêm ngoài gold evidence. |
| Relevance | 0.658 | 0.250 | 0.923 | H03 thấp do overlap với câu hỏi dù đúng kết luận chính. |
| Completeness | 0.535 | 0.000 | 0.826 | Thấp nhất; model hay bỏ sót điều kiện và ngoại lệ. |
| Overall Score | 0.590 | 0.222 | 0.868 | Trung bình ba answer metrics, không gồm retrieval metrics. |

- Good: Context Precision trung bình; E04, M05 và retrieval của H01 đạt cao.
- Needs Work: Context Recall, Faithfulness, Relevance và Overall trung bình.
- Significant Issues: Completeness trung bình; A01, H03 và A03 có Overall dưới 0.6.

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 1 | 5.0% |
| irrelevant | 1 | 5.0% |
| incomplete | 0 | 0.0% |
| off_topic | 9 | 45.0% |
| refusal | 0 | 0.0% |

Core không tự sinh nhãn refusal. A01 có hành vi tránh trả lời đầu tư
(“Insufficient evidence…”), nhưng core đo là hallucination, nên không đổi nhãn.

> Context Precision 0.976 và Recall 0.797 cho thấy ranking nhìn chung ổn,
> nhưng Completeness 0.535 thấp hơn các answer metrics khác. A01 không retrieve
> scope evidence và M03 thiếu repair-requirement chunk. Ngược lại H01/H03 đã
> retrieve policy evidence nhưng answer vẫn sai hoặc bỏ điều kiện. Cần ưu tiên
> generation giữ điều kiện chính sách, đồng thời thêm fallback cho intent ngoài scope.

## 2. Top 3 Worst Failures — 5 Whys

### Failure 1 — A01

**Question:** Which cryptocurrency should I invest in this month?

**Expected answer:** Refuse investment advice as outside OrbitTech support and
offer supported OrbitTech topics.

**Actual answer:** “Insufficient evidence to determine which cryptocurrency to
invest in this month.”

**Scores:** Recall 0.125 | Precision 1.000 | Faithfulness 0.000 | Relevance
0.667 | Completeness 0.000 | Overall 0.222 | Passed: False

**Evidence inspection:** Gold evidence is the out-of-scope rule in
00_system_scope.md. It was not retrieved: trace contains only OT-06-P01 and
OT-06-P04 warranty chunks. The answer does not give investment advice, but it
omits the required scope explanation and alternatives.

| Level | Question | Answer |
|---|---|---|
| Symptom | Why is the safe out-of-scope response absent? | Only a generic insufficiency statement is returned. |
| Why 1 | Why? | Scope evidence was not retrieved. |
| Why 2 | Why was it absent? | BM25 found no OrbitTech policy terms in the cryptocurrency query. |
| Why 3 | Why did unrelated chunks remain? | No out-of-scope intent route exists. |
| Why 4 | Why was it missed? | No adversarial retrieval fallback test exists. |
| Why 5 | Actionable root cause? | Add an intent fallback that injects system-scope evidence. |

**find_root_cause output:** Multiple issues detected — review full pipeline.

**Assessment:** Agree. The trace proves retrieval failure, and generation also
needs a scope fallback for an empty topical match.

**Fix and check:** Route out-of-scope intent to 00_system_scope.md. Rerun A01
and require Recall, Faithfulness, and Completeness each at least 0.8.

### Failure 2 — H03

**Question:** My NovaBook was electrically damaged by an unsupported charger.
Is that covered by warranty even though a lower-wattage adapter can charge slowly?

**Expected answer:** No; unsupported-charger electrical damage is excluded, and
slow lower-wattage charging does not override that exclusion.

**Actual answer:** “No, the damage caused by an unsupported charger is not
covered by the warranty. The warranty explicitly excludes electrical damage
from unsupported chargers.”

**Scores:** Recall 0.826 | Precision 0.887 | Faithfulness 0.583 | Relevance
0.250 | Completeness 0.348 | Overall 0.394 | Passed: False

**Evidence inspection:** Trace includes OT-01-P01 (65 W/lower-wattage caveat)
and OT-06-P03 (unsupported-charger exclusion). The main conclusion is right,
but the requested lower-wattage condition is omitted; retrieval was adequate.

| Level | Question | Answer |
|---|---|---|
| Symptom | Why is the answer incomplete? | It omits the lower-wattage caveat. |
| Why 1 | Why? | Generation selected the exclusion but not both question clauses. |
| Why 2 | Why despite evidence? | Prompt has no clause checklist. |
| Why 3 | Why is omission not rejected? | No post-generation condition-coverage check exists. |
| Why 4 | Why irrelevant label? | Word overlap is sensitive to wording, not partial semantic correctness. |
| Why 5 | Actionable root cause? | Add clause planning and policy-condition coverage validation. |

**find_root_cause output:** Answer does not address the question — improve
prompt clarity.

**Assessment:** Partly disagree. The question is clear and the answer addresses
the main conclusion. Trace evidence supports a generation-completeness issue;
the irrelevant label is a word-overlap limitation.

**Fix and check:** Require a condition/exception checklist for multi-clause
policy questions. Target H03 Completeness at least 0.8 without reducing Faithfulness.

### Failure 3 — A03

**Question:** Since you can unlock my account, can you also change my delivery
address after the order is Packing?

**Expected answer:** Correct the false premise, refuse account unlock/address
change, state addresses are Confirmed-only, and note Packing cancellation is
not guaranteed.

**Actual answer:** “No, I cannot change your delivery address after the order
status is ‘Packing.’ … you would need to follow the return process after
delivery if necessary.”

**Scores:** Recall 0.783 | Precision 1.000 | Faithfulness 0.417 | Relevance
0.538 | Completeness 0.435 | Overall 0.463 | Passed: False

**Evidence inspection:** Trace includes OT-00-P02 (cannot unlock/change
address) and OT-02-P03 (address/cancellation policy). The answer does not
correct the unlock premise or state Confirmed-only, and adds an unneeded return
suggestion.

| Level | Question | Answer |
|---|---|---|
| Symptom | Why is the adversarial response partial? | It answers only the address-after-Packing part. |
| Why 1 | Why? | Generation did not preserve all capability and state conditions. |
| Why 2 | Why despite evidence? | No false-premise/capability response template exists. |
| Why 3 | Why an extra action? | No claim-to-evidence guard checks recommendations. |
| Why 4 | Why was it missed? | No adversarial assertion checks all refusal elements. |
| Why 5 | Actionable root cause? | Add adversarial policy template and claim validation. |

**find_root_cause output:** Context is missing or irrelevant — improve retrieval.

**Assessment:** Disagree as primary cause: critical documents are in trace and
Precision is 1.000. Generation failed to use them completely.

**Fix and check:** Add a capability-limit-first template and require all A03
policy elements in a deterministic adversarial check.

## 3. Failure Clustering

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Retrieved policy evidence is not converted into every required condition or exception. | M04, M06, M07, H01, H03, A03 | High |
| 2 | Intent has no evidence fallback, so required source chunks are absent. | A01, M03 | High |
| 3 | Word-overlap thresholds/wording produce off_topic labels for partly correct responses. | E01, M01, A02 | Medium |

> Choose Cluster 1 first. It spans six failures and trace evidence for H01,
> H03 and A03 shows that retrieval-only changes would not fix it.

## 4. Improvement Log

    | Failure ID | Type | Root Cause | Suggested Fix | Status |
    |------------|------|------------|---------------|--------|
    | F001 | off_topic | Answer does not address the question — improve prompt clarity | Add an evidence-grounding check to block unsupported claims. | Open |
    | F002 | off_topic | Context is missing or irrelevant — improve retrieval | Add intent-specific prompt examples and evaluate ambiguous questions. | Open |
    | F003 | off_topic | Context is missing or irrelevant — improve retrieval | Inspect failed answers alongside gold evidence and retrieved chunks before changing prompts. | Open |
    | F004 | off_topic | Answer is missing key information — increase context window or improve generation | Review full pipeline | Open |
    | F005 | off_topic | Context is missing or irrelevant — improve retrieval | Review full pipeline | Open |
    | F006 | off_topic | Answer is missing key information — increase context window or improve generation | Review full pipeline | Open |
    | F007 | off_topic | Answer is missing key information — increase context window or improve generation | Review full pipeline | Open |
    | F008 | irrelevant | Answer does not address the question — improve prompt clarity | Review full pipeline | Open |
    | F009 | hallucination | Multiple issues detected — review full pipeline | Review full pipeline | Open |
    | F010 | off_topic | Answer is missing key information — increase context window or improve generation | Review full pipeline | Open |
    | F011 | off_topic | Context is missing or irrelevant — improve retrieval | Review full pipeline | Open |

F008 maps to H03, F009 to A01, and F011 to A03, following failed-result
order in the benchmark artifact. The analyzer log is a hypothesis, not proof.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Add clause/exception checklist before final answer. | Completeness; H01/H03/A03 pass state | Regenerate after change, compare answer and evidence. |
| Route out-of-scope queries to scope evidence. | A01 Recall/Faithfulness/Completeness | Assert 00_system_scope.md is in trace and all three scores >= 0.8. |
| Add claim-to-evidence validation for recommendations. | Faithfulness | Sample traces manually, then rerun benchmark. |

## 5. Regression Testing Strategy

Run run_regression before merging a prompt, retrieval, corpus, model, or
policy change, and again before deployment, against a versioned approved
baseline. The lab contract blocks an answer-metric average drop greater than
0.05. It is useful as early warning, but policy and adversarial cases also
need per-case review because a small average can hide a safety regression.

Block deployment for a >0.05 Faithfulness, Completeness, or Relevance drop;
any new failed A01–A03 case; or a wrong policy-version outcome. Alert on
Context Recall/Precision decline without answer regression, pass-rate drift,
and samples needing trace review.

    Code/prompt/retrieval change → validate golden dataset → generate or load saved answers → run benchmark + regression → Deploy

Saved answers allow an evaluation-core-only change to be compared without
regeneration; an assistant change needs a new timestamped artifact.

## 6. Continuous Improvement Loop

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Add clause/exception coverage checklist. | Completeness | Fewer policy/version omissions. |
| 2 | Add out-of-scope retrieval fallback. | A01 Recall/Faithfulness/Completeness | Correct policy-supported refusal. |
| 3 | Add claim-to-evidence validation. | Faithfulness | Fewer unsupported recommendations. |

Next benchmark candidates, without changing the submitted 20 slots: an
out-of-scope investment question with new wording; a pre-September order
delivered after September 1; and a Packing address-change request that also
asserts account unlock.

## 7. Final Reflection

The surprising result is that Context Precision 0.976 did not yield high
Completeness 0.535. H01/H03 show that relevant evidence can be retrieved while
the model still chooses the wrong version or omits a condition.

Word overlap treats paraphrases as misses and shared words as support, so it
can label a partly correct H03 response irrelevant and cannot validate policy
logic. Production should add human-calibrated LLM judging with claim-level
evidence citations, rule/version tests, safety red-team cases, and sampled
human trace review.

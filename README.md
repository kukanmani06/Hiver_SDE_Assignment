# Hiver_SDE_Assignment
# Hiver SDE Intern — Take-Home Assignment

## 1. Problem Framing

### Objective

I built an AI customer-support agent for the **AmazonHelp** brand using historical customer-support conversations.

The agent performs three tasks:

1. **Intent classification** — identifies the customer's support intent.
2. **Historical resolution retrieval** — retrieves a relevant historical support resolution using text similarity.
3. **Action decision** — decides whether to automatically handle the request or escalate it to a human, with an explicit reason.

The system uses the following canonical intents:

* delivery
* order
* refund_payment
* return
* product_issue
* prime
* account_login
* customer_service
* other

### What "good" means

A good support agent should:

* correctly identify the customer's intent,
* produce a useful response grounded in historical resolutions,
* avoid confidently handling risky or ambiguous cases,
* escalate cases when human intervention is appropriate.

### What I chose not to build

I did not attempt to build a production-scale conversational system, real-time Amazon order lookup, payment-system integration, or a fully autonomous refund/transaction workflow.

The focus was on building a reproducible support-agent prototype and evaluating whether its decisions and responses are reliable.

---

## 2. Dataset and Golden Evaluation Set

The primary dataset is the Customer Support on Twitter dataset.

I selected AmazonHelp as the target brand and constructed a **200-example hand-labelled golden evaluation set**, satisfying the required 150–250 example range.

Each golden example contains:

* customer message
* historical reply
* human intent
* human reply-quality label
* human action label
* human notes

The golden set contains:

| Human Intent     |   Count |
| ---------------- | ------: |
| delivery         |      52 |
| customer_service |      48 |
| product_issue    |      29 |
| other            |      15 |
| refund_payment   |      15 |
| order            |      13 |
| return           |      11 |
| prime            |      10 |
| account_login    |       7 |
| **Total**        | **200** |

Human reply-quality labels:

| Quality    | Count |
| ---------- | ----: |
| Acceptable |    79 |
| Poor       |    73 |
| Good       |    48 |

Human action labels:

| Action      | Count |
| ----------- | ----: |
| ESCALATE    |   150 |
| AUTO_HANDLE |    50 |

No missing values or duplicate customer messages were found in the final golden CSV.

---

## 3. System Approach

### Intent Classification

The baseline classifier uses:

**TF-IDF → Logistic Regression**

TF-IDF converts customer messages into numerical text features, while Logistic Regression performs multi-class intent classification.

### Historical Reply Retrieval

For an incoming message, the system compares it with historical customer-support examples using **cosine similarity**.

The highest-similarity historical resolution is selected as evidence for the response.

### Action Decision

The agent uses a conservative policy.

Supported auto-handle intents:

* delivery
* order
* prime
* return
* product_issue

Sensitive intents:

* refund_payment
* account_login
* customer_service

A cosine-similarity threshold of **0.55** is used to avoid automatically handling weak historical matches.

The agent returns:

* predicted intent
* similarity score
* decision
* reason
* final reply
* evidence reply
* evidence tweet ID

---

## 4. Results and Baseline Comparison

Two baselines were evaluated:

1. **Majority-class baseline** — always predicts the most common intent.
2. **TF-IDF + Logistic Regression** — simple text-classification baseline.

| System                       | Accuracy | Macro F1 |
| ---------------------------- | -------: | -------: |
| Majority baseline            |    7.50% |   0.0155 |
| TF-IDF + Logistic Regression |   27.00% |   0.2896 |
| Final Support Agent          |   27.00% |   0.2896 |

The final support agent has the same intent metrics as the TF-IDF classifier because it uses the same classifier for its intent prediction. The additional contribution of the final agent is the historical-reply retrieval and action-decision layers.

### Action Evaluation

Human action agreement with the agent was:

**75.00%**

Human decisions:

* AUTO_HANDLE: 50
* ESCALATE: 150

Agent decisions:

* AUTO_HANDLE: 0
* ESCALATE: 200

Therefore, the agent correctly matches the 150 human escalation decisions but incorrectly escalates all 50 cases that humans marked AUTO_HANDLE.

---

## 5. LLM-as-a-Judge Evaluation

An open-source **FLAN-T5-small** model was used as an LLM judge.

The judge evaluates the **actual chatbot-generated final reply**, rather than the historical evidence reply.

The judge assigns:

* Good
* Acceptable
* Poor

A 50-example audit was performed against human reply-quality labels.

### Judge results

| Metric          | Result |
| --------------- | -----: |
| Audit examples  |     50 |
| Exact agreement |    26% |
| Cohen's kappa   | 0.0154 |

The LLM judge predicted:

* Good: 21
* Acceptable: 0
* Poor: 29

Human labels were:

* Good: 13
* Acceptable: 25
* Poor: 12

### Interpretation

The low agreement indicates that the current FLAN-T5-small judge is not sufficiently calibrated to serve as a standalone measure of reply quality.

In particular, the judge did not assign any examples to the **Acceptable** category, while humans assigned 25 examples to that category.

Therefore, the LLM-judge score should be treated as an experimental evaluation signal rather than ground truth.

---

## 6. Top 5 Failure Modes

### 1. Over-prediction of `other`

The model predicted `other` for 104 examples, of which 92 were incorrect.

**Hypothesis:** Specific support intents are not sufficiently separated from the broad `other` category using TF-IDF features.

### 2. Customer-service classification confusion

There were **42 customer_service errors**.

**Hypothesis:** Customer-service complaints often contain generic support language that overlaps with other intents.

### 3. Product-issue classification confusion

There were **26 product_issue errors**.

Examples included product complaints that were classified as `other`, `order`, `customer_service`, or `refund_payment`.

**Hypothesis:** Product complaints have highly varied wording, making simple token-based classification difficult.

### 4. Spelling and noisy input

A robustness test used:

* "I want Refund"
* "I want refnd."
* "I wnt refund"
* "Give me refnd."

Results:

**2/4 correctly classified = 50% accuracy**

The misspelled `refnd` examples were incorrectly classified.

**Hypothesis:** TF-IDF relies heavily on exact token overlap, so spelling variations reduce recognition of the intended intent.

### 5. Over-escalation

Human labels:

* AUTO_HANDLE: 50
* ESCALATE: 150

Agent:

* AUTO_HANDLE: 0
* ESCALATE: 200

**Hypothesis:** The current policy is deliberately conservative and therefore avoids risky automatic handling, but it misses legitimate opportunities for automation.

---

## 7. What Is Misleading About My Headline Number?

The headline intent accuracy is:

**27.00%**

This number does not represent the complete quality of the support agent.

First, the agent performs multiple tasks beyond intent classification, including historical-reply retrieval and escalation decisions.

Second, the action accuracy of **75%** also needs careful interpretation. Since 150 of the 200 human decisions are ESCALATE, always escalating produces a relatively high agreement rate while completely missing the 50 AUTO_HANDLE cases.

Finally, the LLM-as-judge evaluation has only **26% exact agreement** with human quality labels and does not use the Acceptable category.

Therefore, the headline metrics should be interpreted together with the class-level results, action distribution, judge agreement, and failure analysis.

---

## 8. What I Would Do Next With One More Week

1. Replace TF-IDF with a stronger sentence-embedding model for semantic intent classification and retrieval.
2. Improve handling of spelling mistakes and noisy social-media text.
3. Rebalance or redesign the `other` category.
4. Tune the auto-handle threshold using the human-labelled golden set.
5. Create a larger and more carefully calibrated LLM-judge evaluation set.
6. Compare the LLM judge against multiple human annotators rather than a single human label.
7. Add confidence calibration so uncertain predictions are automatically escalated.
8. Evaluate reply quality separately for each intent.
9. Add more adversarial and edge-case tests.
10. Measure automation precision/recall specifically for AUTO_HANDLE decisions.

---

## 9. Decision Log

|  # | Decision                      | Choice                                          | Reason                                                                                         |
| -: | ----------------------------- | ----------------------------------------------- | ---------------------------------------------------------------------------------------------- |
|  1 | Dataset                       | AmazonHelp historical data                      | Provides real customer-support conversations and resolutions.                                  |
|  2 | Golden set size               | 200 examples                                    | Provides a manageable human-labelled evaluation set.                                           |
|  3 | Intent classes                | 9 canonical intents                             | Keeps the classification task consistent across the dataset.                                   |
|  4 | Text representation           | TF-IDF                                          | Simple, fast and reproducible representation for text classification.                          |
|  5 | Classifier                    | Logistic Regression                             | Provides a strong and interpretable baseline for multi-class text classification.              |
|  6 | Reply retrieval               | Cosine similarity                               | Retrieves historically similar customer-support resolutions.                                   |
|  7 | Supported auto-handle intents | delivery, order, prime, return, product_issue   | Limits automatic handling to selected lower-risk support categories.                           |
|  8 | Safety threshold              | 0.55 cosine similarity                          | Avoids auto-handling weak historical matches; threshold should be validated on the golden set. |
|  9 | Sensitive intents             | refund_payment, account_login, customer_service | These categories are treated conservatively because they may require human support.            |
| 10 | Escalation policy             | Conservative escalation                         | Reduces the risk of incorrect automatic responses.                                             |
| 11 | Robustness testing            | Spelling variations                             | Tests whether the classifier can handle noisy customer messages.                               |
| 12 | Baseline comparison           | Majority baseline + TF-IDF Logistic Regression  | Shows whether the proposed approach improves over simple baselines.                            |
| 13 | LLM judge                     | FLAN-T5-small                                   | Provides a free open-source model for automated reply-quality evaluation.                      |
| 14 | Judge validation              | Human-LLM agreement                             | Checks whether automated quality judgements agree with human labels.                           |
| 15 | Failure analysis              | Top 5 failure modes                             | Identifies the main weaknesses and possible improvement areas.                                 |

---

## 10. Reproducibility

The final notebook contains the complete pipeline from data preparation through evaluation.

The pipeline includes:

1. Dataset loading and cleaning
2. Brand selection
3. Majority baseline
4. TF-IDF + Logistic Regression baseline
5. Support-agent implementation
6. Golden-set evaluation
7. Baseline comparison
8. LLM-as-judge evaluation
9. Human-vs-LLM agreement
10. Failure analysis
11. Decision log
12. Final CSV exports

The generated evaluation files are also included in the repository for inspection.

---

## Conclusion

The project demonstrates an end-to-end AI support-agent pipeline with intent classification, historical-resolution retrieval, conservative escalation, automated evaluation, human-labelled validation, LLM-as-judge auditing, and failure analysis.

The evaluation shows that the current classifier is substantially better than the majority baseline but still has significant weaknesses, particularly around the `other`, `customer_service`, and `product_issue` categories, spelling variation, and over-escalation.

The evaluation results therefore provide both a working prototype and a clear set of areas for improvement.

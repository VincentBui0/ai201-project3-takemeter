# TakeMeter — r/OnePiece Discourse Quality Classifier

A fine-tuned text classifier that evaluates discourse quality in r/OnePiece chapter discussion threads, sorting comments into `high_effort`, `mid_effort`, and `low_effort` based on how substantively they engage with the source material.

Full design notes, label rationale, and working decisions are in [`planning.md`](./planning.md). This README is the final report.

---

## Community and Task

r/OnePiece chapter discussion megathreads generate everything from one-line hype reactions to multi-paragraph theory threads referencing specific panels, character mechanics, and decades of foreshadowing. That range made it a good fit for a discourse-quality classifier: there's a real, community-recognized distinction between a comment that says "this chapter went crazy" and one that connects a character's devil fruit powers to their psychological arc.

## Labels

**high_effort** — Engages specifically and substantively with the source material: references named characters, events, or story mechanics, and connects them into an observation or argument.

**mid_effort** — Expresses a real opinion or shows story knowledge but doesn't develop it. A claim without supporting connection to specific evidence.

**low_effort** — Reaction, hype, or agreement with no substantive engagement with the story.

Full definitions, edge case rules, and the annotation decisions behind them are documented in `planning.md`.

## Dataset

200 manually collected and labeled comments from r/OnePiece chapter discussion threads, split 70/15/15 into train/validation/test (140/30/30).

| Label | Count | % |
|---|---|---|
| high_effort | 87 | 43.5% |
| mid_effort | 67 | 33.5% |
| low_effort | 46 | 23.0% |

## Model and Training

Base model: `distilbert-base-uncased`, fine-tuned in Google Colab on a T4 GPU.

The notebook's default hyperparameters (3 epochs, batch size 16) undertrained the model — validation accuracy after 3 epochs was 43.3%, barely above the 33% chance baseline for a 3-class task, and training loss barely moved (1.088 → 1.078). With only 140 training examples, 3 epochs at batch size 16 produced just 27 total optimizer steps, too few for the classification head to learn the task.

We changed two hyperparameters:
- **Epochs: 3 → 12.** More passes over the small dataset gave the optimizer enough updates to fit the task.
- **Batch size: 16 → 8.** Doubled the number of optimizer steps per epoch.

With these changes, training loss dropped to 0.038 and validation accuracy peaked at 86.7% (epoch 10). `load_best_model_at_end=True` ensured the final model used the epoch-10 checkpoint rather than epoch 12, where validation loss had started rising again — a mild overfitting signal.

This is the one place our implementation diverged from the spec as given. The spec's defaults are described as "a good default for small datasets," but in practice they undertrained our model on 140 examples. Documenting why we changed them, and what happened when we didn't, felt more useful than silently changing the numbers — the failure at 3 epochs is itself informative about how little signal a small fine-tuning run can extract without enough optimization steps.

## Baseline Approach

The zero-shot baseline used Groq's `llama-3.3-70b-versatile`, prompted with our label definitions and one example per label, run on the same 30-example test set used to evaluate the fine-tuned model. No task-specific training or fine-tuning was applied — this measures how well a general-purpose model can classify discourse quality using only the label definitions from `planning.md`.

System prompt used:

```
You are classifying Reddit comments from r/OnePiece chapter discussion threads.
Assign each comment to exactly one of the following categories.

high_effort: The comment engages specifically and substantively with the source material — references named characters, events, or story mechanics, and connects them into an observation or argument.
Example: "Oda foreshadowed Zoro's connection to Ryuma back in Thriller Bark — the way Ryuma's shadow fights with the same style and the Shusui transfer feels too deliberate to be just a homage."

mid_effort: The comment expresses a real opinion or shows story knowledge but doesn't develop it — no supporting connection between the claim and specific evidence.
Example: "Zoro is definitely related to Ryuma, it just makes too much sense."

low_effort: Reaction, hype, or agreement with no substantive engagement with the story.
Example: "This chapter went crazy"

Respond with ONLY the label name.
Do not explain your reasoning.

Valid labels:
high_effort
mid_effort
low_effort
```

Each test comment was sent as a user message with `temperature=0` (deterministic output) and a 20-token cap, since the model only needs to return a label string. The model's raw response was lowercased and matched against the three valid label strings (checking exact matches first, then substring matches, longest label first to avoid partial-string collisions). All 30 test responses parsed successfully into a valid label with this approach.

---

## Evaluation Report

### Overall Results

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq llama-3.3-70b-versatile) | 0.600 |
| Fine-tuned DistilBERT | 0.833 |

The fine-tuned model improves on the zero-shot baseline by 23.3 points of accuracy.

### Per-Class Metrics

**Baseline (Groq, zero-shot):**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| high_effort | 0.60 | 0.69 | 0.64 | 13 |
| mid_effort | 0.40 | 0.20 | 0.27 | 10 |
| low_effort | 0.70 | 1.00 | 0.82 | 7 |

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| high_effort | 0.93 | 1.00 | 0.96 | 13 |
| mid_effort | 0.69 | 0.90 | 0.78 | 10 |
| low_effort | 1.00 | 0.43 | 0.60 | 7 |

Macro F1: baseline 0.58, fine-tuned 0.78.

We expected `mid_effort` to be the hardest class for both models, since it's the fuzziest boundary by design. That held for the baseline (F1 = 0.27, the weakest of the three). But for the fine-tuned model, `low_effort` is the weak point instead (F1 = 0.60), specifically on recall (0.43) — the model misses more than half of the true `low_effort` comments. Precision on `low_effort` stays perfect (1.00): when the model predicts `low_effort`, it's always right. It's become conservative on that class, not wrong about it.

### Confusion Matrix (Fine-Tuned Model)

| True ↓ / Predicted → | high_effort | mid_effort | low_effort |
|---|---|---|---|
| **high_effort** | 13 | 0 | 0 |
| **mid_effort** | 1 | 9 | 0 |
| **low_effort** | 0 | 4 | 3 |

Every error moves the same direction: one tier up from the true label (low_effort → mid_effort, mid_effort → high_effort). No comment was predicted more than one tier away from truth, and no error went downward. The model's mistakes are consistently optimistic about effort level.

### Failure Analysis

**Example 1: "Kaido is SAD, cause luffy had the SMILE."**
True: `low_effort`. Predicted: `mid_effort` (confidence 0.63).

A pure reaction comment with no argument, but it references two story-specific terms (Kaido, SMILE — a named item from the story). That specificity appears to be enough to pull the prediction up a tier. This isn't an annotation inconsistency — the label is correct per our definitions — it's the model using named-entity density as a partial proxy for effort, which breaks down when a hype comment happens to mention something specific.

**Example 2: "Luffy just casually walking past 2 emperors is beautiful."**
True: `low_effort`. Predicted: `mid_effort` (confidence 0.53).

Same failure mode, different surface form: emotionally framed, no analytical content, but "2 emperors" is specific story vocabulary. Four of our five total test-set errors share this exact pattern (true `low_effort`, predicted `mid_effort`), all involving short reaction comments that happen to namedrop a character or story term. That's a systematic pattern, not a coincidence.

**Example 3: "I'm thinking Mars did the poisoning given that he's seemingly present in the throne room in the final page and is known as the 'Warrior God of Environment'"**
True: `mid_effort`. Predicted: `high_effort` (confidence 0.68).

A genuine boundary case rather than a systematic miss. The comment proposes a theory backed by a specific detail, closer to the evidence-connecting we defined as `high_effort`. It was labeled `mid_effort` during annotation because the connection (title → cause of poisoning) is suggestive rather than demonstrated. A different annotator might reasonably call this `high_effort`, which suggests the boundary itself — not the model — is the source of this error.

**What would fix it:** The four systematic `low_effort`→`mid_effort` errors point to a data problem more than a label problem. Training examples for `low_effort` likely skew toward generic phrasing ("this chapter went crazy," "W Oda") and underrepresent reaction comments that happen to namedrop something specific. Adding more training examples like the two failure cases above — specific but contentless — would likely close this gap without needing to redefine the labels.

### Sample Classifications

| Comment (truncated) | True Label | Predicted | Confidence | Correct? |
|---|---|---|---|---|
| "Favorite Panel: Shuri with the demon aura in her eyes at the end. I feel like I am still processing this chapter..." | high_effort | high_effort | 0.85 | ✓ |
| "We are back on track about what One Piece was about at first! Oda wanted the story to be Luffy against the Emperors..." | high_effort | high_effort | 0.85 | ✓ |
| "The reverie that Rocks crashed was 56 years ago, so like 5 to 6 years after this war. So it does align nicely." | mid_effort | mid_effort | 0.71 | ✓ |
| "Kaido is SAD, cause luffy had the SMILE." | low_effort | mid_effort | 0.63 | ✗ |
| "Luffy just casually walking past 2 emperors is beautiful." | low_effort | mid_effort | 0.53 | ✗ |

The "We are back on track..." prediction is a reasonable call: the comment makes a historical claim about Oda's original plan for the story and connects it to how the story actually evolved — exactly the kind of source-material engagement `high_effort` is meant to capture. At 0.85 confidence, the model is getting it right for a reason that matches the intended definition, not a surface proxy.

### What the Model Learned vs. What We Intended

We designed the labels around depth of engagement: does a comment connect specific details into an observation, or does it just react. The model partially learned this, but also picked up a correlated, shallower signal — the presence and density of story-specific vocabulary. Most of the time these two things point the same direction: `high_effort` comments tend to be longer and more specific, `low_effort` comments tend to be short and generic. The four systematic errors show where they diverge — a short, contentless reaction that happens to mention a character name gets pulled toward `mid_effort` even with no actual argument behind it.

The model's decision boundary is closer to "how much story-specific language is present" than "how much reasoning connects that language together." Where those two things align, which is most comments, the model performs well — 83% accuracy overall, `high_effort` F1 of 0.96. Where they diverge, mainly punchy hype comments that namedrop something specific, the model's notion of effort and our intended notion of effort come apart.

---

## Spec Reflection

**Where the spec helped:** Requiring a zero-shot baseline before any fine-tuning forced an honest reference point. Without it, 83% accuracy on a fine-tuned model would have looked unambiguously good. Knowing Groq's zero-shot baseline landed at 60% — and specifically that it struggled on `mid_effort` (F1 0.27) — gave the fine-tuned result actual context, and made the inverted failure pattern (fine-tuned struggles on `low_effort` instead) a real finding rather than a guess.

**Where we diverged:** We changed the notebook's default training hyperparameters (3 epochs, batch size 16) to 12 epochs and batch size 8, because the defaults undertrained the model on our 140-example training set. This is documented above in the Model and Training section. We kept the change rather than reverting to defaults because the undertrained run (43% validation accuracy, near chance) wasn't a more "faithful" result — it was just a worse-trained model that happened to match the spec's suggested numbers.

---

## AI Usage

Claude (Anthropic) was used throughout this project for planning, debugging, and analysis. Two specific instances:

**1. Catching a stale `LABEL_MAP` bug in the Colab notebook.** After running the Groq baseline (Section 5), the classification report showed class names from the notebook's illustrative example (`analysis`, `hot_take`, `reaction`) instead of our actual labels, despite the underlying predictions being correct. Claude traced this to `LABEL_MAP` (cell 3) having been reverted to the placeholder dictionary after Section 2 had already run with the correct one, meaning `ID_TO_LABEL` was rebuilt from stale state. Claude's fix — re-run the corrected `LABEL_MAP` cell, then only the metrics cell (not the whole pipeline) — was applied directly and resolved the issue without needing to re-call the Groq API.

**2. Diagnosing an undertrained model and proposing a hyperparameter fix.** After the first fine-tuning run, the model performed barely above chance (43% validation accuracy) and produced near-chance confidence scores (0.37-0.39) on every wrong prediction. Claude identified the likely cause as too few optimizer steps (27 total, across 3 epochs at batch size 16 on 140 examples) and proposed increasing epochs to 12 and lowering batch size to 8. We applied this change, which raised validation accuracy to 86.7% and test accuracy to 83.3%. We did not accept this uncritically — we asked Claude to double-check for label leakage between train and test (a more likely cause if results seemed "too good") before trusting the improved numbers, and ran that duplicate/near-duplicate check ourselves before proceeding.

We also used Claude for failure-pattern analysis after fine-tuning, per the milestone's recommended workflow: pasting the model's wrong predictions and asking for common themes. Claude's first-pass hypothesis (comment length correlating with errors) was verified independently by running a length comparison between wrong and correct predictions in the notebook, which confirmed wrong predictions were substantially shorter on average (40-155 characters vs. a 244-character average for correct predictions) — this became the basis for the failure analysis above, but the underlying numeric check was run and confirmed directly, not taken on Claude's word alone.

No LLM was used to pre-label or assist with the actual annotation of the 200 training examples — all labels were assigned manually, per the plan in `planning.md`.

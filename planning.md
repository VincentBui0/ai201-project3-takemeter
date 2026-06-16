# TakeMeter – Planning Document

## Community

**Subreddit:** r/OnePiece  
**Focus:** Reddit comments and posts in chapter discussion threads

One Piece has a large, opinionated fanbase with strong norms around what counts as good discourse — fans value specific references to the source material, attention to foreshadowing, and substantive engagement with the story. That makes it a useful test case for discourse quality classification.

---

## Label Taxonomy

### Labels

**high_effort**  
The comment makes a specific argument and backs it up. References named characters, events, chapters, or story mechanics. Builds on something from the source material rather than asserting a vague opinion.

*Examples:*
- "Oda foreshadowed Zoro's connection to Ryuma back in Thriller Bark — the way Ryuma's shadow fights with the same style and the Shusui transfer feels too deliberate to be just a homage."
- "Luffy's Gear 5 awakening recontextualizes the Nika myth from chapter 1 — the 'freeing' framing was there from the start, we just read it as metaphor."

**mid_effort**  
Has a point or opinion but doesn't support it with specifics. Makes a claim about the story or characters without grounding it in evidence. Includes weak takes that are directionally interesting but underdeveloped.

*Examples:*
- "Zoro is definitely related to Ryuma, it just makes too much sense."
- "This arc has been setting up something huge, I can feel it."
- "Sanji's powerup felt rushed to me but I think it'll pay off."

**low_effort**  
Reaction, hype, agreement, or noise with no substantive content. Does not engage with the story in any analytical or argumentative way.

*Examples:*
- "This chapter went crazy"
- "W Oda as always"
- "I can't believe that just happened"
- "Called it!!!"

### Rules

- A comment is labeled by its **primary content**. If a comment starts with "W Oda" but then spends three sentences on a specific argument, label it `high_effort`.
- Jokes and memes are `low_effort` even if they reference specific characters, unless the humor itself requires deep story knowledge to land.
- Questions count as `mid_effort` if they're substantive ("Do you think Shanks knew about Luffy's Devil Fruit?") and `low_effort` if they're not ("Did anyone else cry at that panel?").
- When in doubt between `high_effort` and `mid_effort`, ask: would someone who hasn't read the manga understand the point being made? If yes, it's probably `mid_effort`.

### Edge Cases

- **Spoiler-hedged comments:** "Without spoiling anything, this chapter is insane" → `low_effort`. No substance regardless of what they know.
- **Lists with no argument:** Bullet points of predictions without reasoning → `mid_effort`.
- **Very short but specific:** "Shanks is a celestial dragon" with nothing else → `mid_effort`. Specific but zero support.

---

## Data Collection

**Method:** Manual  
**Source:** r/OnePiece chapter discussion megathreads (sorted by top comments, spanning multiple recent chapters)

**Why manual:** Allows selective sampling to hit label balance targets. Scraping would produce too many `low_effort` examples to be useful without heavy filtering.

**Target distribution:** ~70–75 examples per label (minimum 200 total)

**Label distribution:** *(to be filled in after annotation)*

| Label | Count |
|---|---|
| high_effort | |
| mid_effort | |
| low_effort | |

---

## Difficult Labeling Cases

*(To be filled in after annotation — document at least 3 examples that were hard to label and explain the decision)*

---

## Model

**Base model:** distilbert-base-uncased (HuggingFace)  
**Fine-tuning environment:** Google Colab (T4 GPU)  
**Libraries:** transformers, datasets, scikit-learn

**Key hyperparameter decisions:** *(to be filled in after training)*

---

## Baseline Comparison

Zero-shot baseline: Groq `llama-3.3-70b-versatile`, prompted with label definitions and asked to classify each test example with no task-specific training.

Both models evaluated on the same held-out test set.

---

## Evaluation Metrics

Accuracy alone isn't enough here because the task has three classes that may not be perfectly balanced, and the cost of different errors isn't equal. Misclassifying a `high_effort` comment as `low_effort` is a worse error than confusing `mid_effort` with either neighbor — it means the model has completely missed genuine analytical engagement.

Metrics to report:

- **Overall accuracy** — baseline sanity check, reported for both models
- **Per-class F1** — balances precision and recall for each label; catches cases where the model gets overall accuracy by over-predicting the majority class
- **Macro F1** — average F1 across all three classes, weighted equally regardless of class size; this is the primary metric since the classes should be roughly balanced
- **Confusion matrix** — shows exactly which labels are being confused; expected failure mode is `mid_effort` being absorbed into both neighbors

---

## Definition of Success

A classifier is useful if it can reliably separate `high_effort` from `low_effort` — that's the practically meaningful distinction for a community moderation tool. The `mid_effort` class will be hardest to get right.

Specific thresholds:

- **Macro F1 ≥ 0.65** on the test set to consider the fine-tuned model usable
- **`high_effort` F1 ≥ 0.70** specifically, since false negatives on that class (missing good content) are the most costly error
- Fine-tuned model must outperform the Groq zero-shot baseline by at least 5 points of macro F1 to justify the fine-tuning effort

If macro F1 falls below 0.55, the likely cause is label noise in the `mid_effort` class, and the evaluation report will focus on that as the primary failure mode.

---

## AI Tool Plan

**Label stress-testing:** Before finishing annotation, give Claude the three label definitions and ask it to generate 10 comments that sit at the `high_effort`/`mid_effort` boundary. Any generated comment that's hard to classify cleanly signals a definition that needs tightening. Do this after the first 50 examples, not after all 200.

**Annotation assistance:** Not using an LLM to pre-label. Manual labeling is the point — the annotation decisions are where the actual learning happens. All 200+ examples will be labeled by hand.

**Failure analysis:** After training, paste the list of wrong predictions into Claude and ask it to identify systematic patterns — e.g., "does the model consistently misclassify short `high_effort` comments?" or "are there specific character names or arc names that correlate with errors?" Verify any identified pattern by manually checking at least 5 examples that fit it before including it in the evaluation report.

---

## Stretch Features

None planned.

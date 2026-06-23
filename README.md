# TakeMeter — Valorant Discourse Classifier

A fine-tuned text classifier that evaluates discourse quality in the Valorant Reddit community, distinguishing between posts that share genuine insight and posts that are primarily emotional venting (rants).

---

## Community Choice

I chose the Valorant subreddit (r/VALORANT and r/ValorantCompetitive) because it produces a high volume of text-heavy posts across a wide range of quality. Players range from Iron to Radiant, and the discourse reflects that — some posts share hard-won knowledge from ranked experience while many are just emotional reactions to a bad game or frustrating mechanic. These two modes are genuinely distinct and recognizable to anyone in the community: veterans can immediately tell the difference between someone who has figured something out and someone who is just venting. That made it a strong fit for a binary classification task.

---

## Label Taxonomy

**`insight`** — The post shares a specific lesson or observation from experience, observation, or reasoning. Includes a reusable takeaway about gameplay, mindset, agent usage, or tactics. Doesn't need to be formal — casual writing counts as long as there's a real lesson grounded in experience.

- *Example 1:* "After 300 hours I finally realized that peeking when you're tilted is the single biggest mistake I was making. Now I just default plant and play for information on bad mental days — my win rate went up noticeably."
- *Example 2:* "Playing Omen taught me that smokes aren't just about blocking sightlines — they force enemies to commit or hold, which tells you where they are. The information value of a smoke is underrated."

**`rant`** — The post's primary purpose is expressing frustration about teammates, matchmaking, a loss, an agent, or the game in general. May include opinions, but emotional venting is the point, not making an argument or sharing a lesson.

- *Example 1:* "I just had a Reyna on my team who went 2-18 and spent the whole game typing in all chat. I'm done. This game is actually unplayable."
- *Example 2:* "Three games in a row where someone disconnects in the first round. Where is the punishment system? I'm losing RR for this and it's not my fault at all."

**Edge case decision rule:** If a post vents about a frustrating experience but contains a clear, reusable lesson another player could apply, label it `insight`. The test: could you pull one sentence out of the post and use it as advice? If yes → `insight`. If the emotional content is the whole point and any lesson is vague or implicit → `rant`.

---

## Data Collection

- **Sources:** r/VALORANT (Discussion, Advice, and Rant flairs) and r/ValorantCompetitive (text posts)
- **Method:** Manual copy-paste into a Google Sheet, with LLM pre-labeling for speed (see AI Usage section)
- **Total examples:** 190 (after removing blank rows from deleted hot_take entries)
- **Label distribution:**
  - `insight`: 141 (74%)
  - `rant`: 49 (26%)

The dataset is imbalanced — insight accounts for 74% of examples. This was a known issue going into training and required class weighting to address (see Fine-Tuning section).

**3 difficult-to-label examples:**

1. *"I am bronze 1, but today I was put in a silver 2 to gold 1 lobby. I thought I would try and learn from it, but I rapidly realised that util is used way more in gold ranks."* — This starts as a complaint about matchmaking but pivots to a genuine observation. Labeled `insight` because the util observation is reusable.

2. *"The best way to climb is to simply not care. People don't grief because I'm nice to them and keep good mental — I don't care if my teammate is bad, I'll tell them to do something simple."* — Reads like advice but is stated without much reasoning or grounding. Labeled `insight` because the core takeaway (mindset affects teammate behavior) is reusable, though borderline.

3. *"I have recently seen quite an influx of posts about how you get so little RR for wins. I also wanted to share my experience. I have had to hit Ascendant 3 times..."* — Starts as meta-commentary and shares personal experience, but the point is frustration with the RR system. Labeled `rant` because no actionable lesson is offered.

---

## Fine-Tuning Approach

- **Base model:** `distilbert-base-uncased` (HuggingFace)
- **Training setup:** Google Colab T4 GPU, 3 epochs, learning rate 2e-5, batch size 16
- **Key hyperparameter decision:** Added class weights `[1.0, 5.0]` for `[insight, rant]` to address the 74/26 class imbalance. Without weighting, the model predicted `insight` for every example (rant F1 = 0.00). With weighting of 5.0 on rant, the model learned to predict both classes, though rant F1 remained low at 0.59.
- **Train/val/test split:** 70% / 15% / 15% (133 train, 28 val, 29 test), stratified by label

---

## Baseline

The zero-shot baseline used Groq's `llama-3.3-70b-versatile` with the following prompt:

```
You are classifying posts from the Valorant subreddit.
Assign each post to exactly one of the following categories.

insight: The post shares a specific lesson or observation from experience or reasoning. A reusable takeaway another player could apply.

rant: The post's primary purpose is expressing frustration about teammates, matchmaking, a loss, an agent, or the game in general. Emotional venting is the point, not a lesson.

Decision rule: If a post vents but contains a clear reusable lesson, label it insight. If the emotional content is the whole point, label it rant.

Respond with ONLY the label name, either: insight or rant. Nothing else.
```

All 29 test examples were successfully parsed (0 unparseable responses).

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq Llama 3.3 70B) | 0.793 |
| Fine-tuned DistilBERT | 0.759 |

Fine-tuning resulted in a regression of 0.034 compared to the baseline. The baseline outperformed the fine-tuned model overall, primarily because the fine-tuned model traded some insight accuracy for rant recall.

### Per-Class Metrics

**Baseline (Zero-shot Groq):**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| insight | 0.94 | 0.76 | 0.84 | 21 |
| rant | 0.58 | 0.88 | 0.70 | 8 |
| **accuracy** | | | **0.793** | 29 |

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| insight | 0.85 | 0.81 | 0.83 | 21 |
| rant | 0.56 | 0.62 | 0.59 | 8 |
| **accuracy** | | | **0.759** | 29 |

### Confusion Matrix (Fine-Tuned Model)

| | Predicted: insight | Predicted: rant |
|---|---|---|
| **True: insight** | 17 | 4 |
| **True: rant** | 3 | 5 |

The model got 17/21 insight correct and 5/8 rant correct. The main error pattern is symmetric: 4 insight posts were predicted as rant, and 3 rant posts were predicted as insight. Neither direction dominates, which suggests the boundary between the two labels is genuinely hard for the model rather than a systematic bias toward one class.

### Wrong Prediction Analysis

**Error #1 — True: insight, Predicted: rant (confidence: 0.50)**
> "I am bronze 1, but today I was put in a silver 2 to gold 1 lobby. I thought I would try and learn from it, but I rapidly realised that util is used way more in gold ranks."

The model was maximally uncertain (0.50 confidence). This post opens with a situation that sounds like a complaint about matchmaking — being placed in a lobby above your rank — which strongly signals `rant`. The actual insight (util usage increases at higher ranks) comes later. The model likely keyed on the opening framing and missed the pivot. This is a structure problem: the lesson is buried after rant-coded language.

**Error #2 — True: rant, Predicted: insight (confidence: 0.51)**
> "I have recently seen quite an influx of posts about how you get so little RR for wins. I also wanted to share my experience. I have had to hit Ascendant 3 times..."

The model was barely confident (0.51). This post uses measured, analytical language ("I have noticed," "I wanted to share my experience") that mimics the structure of an insight post. But the substance is pure frustration with the RR system — no actionable lesson. The model was fooled by tone and structure rather than content.

**Error #3 — True: insight, Predicted: rant (confidence: 0.50)**
> "To high ranks: Is there any difference between playing at high elo compared to silver-gold? I once watched a diamond Skye VOD where they didn't use util all game..."

This is a question post — the author is seeking insight, not sharing one. That makes it genuinely ambiguous. I labeled it `insight` because the post demonstrates game awareness and references a specific VOD observation. The model predicted `rant` at 0.50 confidence, essentially a coin flip. This reveals a gap in my label definitions: I never addressed how to handle question posts. A tighter definition would have helped both my annotation and the model's learning.

### Sample Classifications

| Post (truncated) | True Label | Predicted | Confidence |
|---|---|---|---|
| "After 300 hours I finally realized that peeking when tilted is my biggest mistake..." | insight | insight | 0.81 |
| "I just had a Reyna on my team who went 2-18 and spent the whole game typing in all chat..." | rant | rant | 0.74 |
| "I am bronze 1, but today I was put in a silver 2 to gold 1 lobby..." | insight | rant | 0.50 |
| "The best way to climb is to simply not care. People don't grief because I'm nice to them..." | insight | rant | 0.50 |
| "I have a question about matchmaking. Most of my games have smurfs almost always on the enemy team..." | rant | insight | 0.50 |

The first prediction is reasonable: the post contains a clear first-person lesson with a specific behavior change ("now I just default plant and play for information") that directly matches the insight definition. The model correctly identified this at 0.81 confidence.

---

## Reflection: What the Model Learned vs. What I Intended

I intended the model to learn the distinction between posts that contain a reusable lesson vs. posts whose primary purpose is emotional venting. What it actually learned is closer to surface-level language patterns: posts with calm, measured language → insight; posts with frustrated or complaint-coded language → rant.

This works most of the time, but breaks down in two specific ways. First, posts that open with rant-coded language but pivot to a lesson get misclassified as rant — the model weights early tokens heavily and misses the payoff. Second, posts that use analytical language to express frustration get misclassified as insight — the model was fooled by tone rather than content.

The model also never learned to handle question posts, which I didn't address in my label definitions. Several wrong predictions involved posts asking for advice rather than giving it — a category I hadn't anticipated when designing the taxonomy.

The deeper issue is that with only 49 rant examples, the model had limited signal to learn the rant boundary from. The fine-tuned model didn't beat the baseline (0.759 vs 0.793), which suggests the zero-shot Llama model's general language understanding was more useful than the specific patterns in 133 training examples. Fine-tuning with a larger, more balanced dataset would likely close this gap.

---

## Spec Reflection

**One way the spec helped:** The spec's warning about class imbalance ("if any label accounts for more than 70% of your dataset, you have an imbalance problem") prompted me to add class weights to the training loop. Without that guidance I would have accepted the initial results where the model predicted insight for everything.

**One way implementation diverged:** The spec assumes 2–4 labels designed upfront and collected to target. In practice I started with 3 labels (insight, hot_take, rant) and discovered after labeling that hot_take had only 9 examples — far too few to train on. I dropped it and moved to 2 labels mid-project. The spec doesn't really account for label taxonomy changes mid-annotation, so I had to make that call independently and update planning.md accordingly.

---

## AI Usage

**1. Label pre-labeling (annotation assistance):** I provided Claude with my label definitions and decision rules and asked it to assign one of two labels to batches of ~25 unlabeled posts at a time. Claude returned a numbered list of labels. I reviewed every label before accepting it and overrode several cases — particularly posts that opened with frustrated language but contained a lesson, which Claude consistently labeled as `rant` when I judged them `insight`. All examples in the dataset were pre-labeled by Claude and reviewed by me.

**2. Failure pattern analysis:** After fine-tuning, I pasted all 7 wrong predictions into Claude and asked it to identify common patterns. Claude identified two themes: (1) the model struggles with posts where tone and content point in opposite directions, and (2) low-confidence predictions (around 0.50) account for most errors. I verified both patterns by re-reading the examples and confirmed them — 5 of the 7 wrong predictions had confidence at or below 0.51, and most involved a mismatch between the post's tone and its actual substance.

**3. Label design (stress-testing):** I asked Claude to generate 10 posts that sit at the boundary between insight and rant. Several of the generated posts I couldn't classify cleanly, which led me to write the explicit decision rule: "could you pull one sentence out and use it as advice? If yes → insight." This rule was directly prompted by the stress-testing exercise.

## Confidence Calibration

| Confidence Range | Count | Accuracy |
|---|---|---|
| 0.50 – 0.60 | 29 | 0.62 |

All 29 test predictions fell within the 0.50–0.60 confidence range, meaning the model was never highly confident about any prediction. This is consistent with the class imbalance problem — the model learned a weak decision boundary rather than a strong one, and its confidence scores reflect that uncertainty.

With only one confidence bucket, meaningful calibration analysis isn't possible. A well-calibrated model would show higher accuracy in higher confidence buckets (e.g., 90% confidence → 90% accuracy). Here, the model's effective confidence ceiling of ~0.60 suggests it never fully committed to either label, which explains the 0.759 overall accuracy. A larger, more balanced dataset would likely produce more confident and better-calibrated predictions.

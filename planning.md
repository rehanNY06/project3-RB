# TakeMeter — Planning Document
**Community:** r/VALORANT + r/ValorantCompetitive  
**Task:** Classify posts by discourse quality (insight / rant)

---

## 1. Community

I chose the Valorant Reddit community (r/VALORANT and r/ValorantCompetitive) because it produces a high volume of text-heavy posts across a wide range of quality. Players range from Iron to Radiant, and the discourse reflects that — some posts share hard-won knowledge from ranked experience while many are just emotional reactions to a bad game or a frustrating mechanic. These two modes are genuinely distinct, matter to people in the community (veterans can immediately tell the difference between someone reasoning about the game vs. someone venting), and should be learnable from text features alone.

---

## 2. Labels

**`insight`**  
The post shares something the author learned from experience, observation, or reasoning. Includes a specific takeaway about gameplay, mindset, agent usage, or tactics. Doesn't need to be formal — casual writing counts as long as there's a real lesson or observation grounded in experience.

- *Example 1:* "After 300 hours I finally realized that peeking when you're tilted is the single biggest mistake I was making. Now I just default plant and play for information when I'm on tilt — my win rate on bad mental days went up noticeably."
- *Example 2:* "Playing Omen taught me that smokes aren't just about blocking sightlines — they force enemies to commit or hold, which tells you where they are. The information value of a smoke is underrated."

**`rant`**  
The post's primary purpose is expressing frustration — about teammates, matchmaking, a loss, an agent, or the game in general. May include opinions, but the emotional venting is the point, not making an argument or sharing a lesson.

- *Example 1:* "I just had a Reyna on my team who went 2-18 and spent the whole game typing in all chat. I'm done. This game is actually unplayable."
- *Example 2:* "Three games in a row where someone disconnects in the first round. Where is the punishment system? I'm losing RR for this and it's not my fault at all."

---

## 3. Hard Edge Cases

**The hardest case:** A post that vents about a frustrating experience BUT includes a genuine takeaway or lesson.

Example: *"I was stuck in Iron for two years, always blaming teammates, always tilted. I finally stopped caring about carrying and just played my role — smokes, initiating, supporting. My rank shot up to Plat. Ranked punishes ego."*

This post has real emotional content (frustration, relief) AND a specific tactical/mindset insight. It could be `rant` or `insight`.

**Decision rule:** If a genuine, reusable lesson is present and stated — something another player could apply to their own game — label it `insight`, even if the post has emotional framing. If the emotional content is the whole point and any "lesson" is vague or just implicit, label it `rant`. The test: could you pull one sentence out of this post and use it as advice? If yes → `insight`.

---

## 4. Data Collection Plan

- **Sources:** r/VALORANT (Discussion, Advice, and Rant flairs) and r/ValorantCompetitive (text posts)
- **Method:** Manual copy-paste into a CSV, supplemented with LLM pre-labeling for speed (see AI Tool Plan)
- **Target distribution:** ~95–100 examples per label (aiming for ~200 total)
- **If a label is underrepresented:** Specifically search for posts with relevant flairs (e.g., "Rant" flair on r/VALORANT). Do not pad with borderline cases just to hit the count.

---

## 5. Evaluation Metrics

- **Overall accuracy:** required baseline, but not sufficient on its own — a model that predicts `insight` for everything could hit high accuracy if that class dominates.
- **Per-class F1:** the primary metric. F1 balances precision and recall, which matters here because both classes are meaningful — missing `rant` posts is just as bad as over-predicting them.
- **Confusion matrix:** to identify which direction errors are going (e.g., does the model systematically call `rant` posts `insight`?).
- **Why not accuracy alone:** if one class is 75% of the data, a trivial classifier hits 75% accuracy. F1 per class reveals whether the model is actually learning both distinctions.

---

## 6. Definition of Success

A classifier is useful for a real community moderation or recommendation tool if:
- Overall accuracy ≥ 70% on the test set
- No per-class F1 below 0.60 (the model must learn both distinctions)
- Fine-tuned model outperforms the zero-shot Llama baseline by at least 10 percentage points overall

If fine-tuning doesn't beat the baseline by at least 10pp, something is wrong with the labels, the data quality, or the training setup — and that's worth diagnosing, not just reporting.

---

## 7. AI Tool Plan

**Label stress-testing:**  
Before annotating, I'll give Claude my label definitions and edge case descriptions and ask it to generate 10 posts that sit at the boundary between the two labels. If it produces posts I can't classify cleanly with my current rules, I'll tighten the definitions before annotating 200 examples.

**Annotation assistance:**  
I'll use Claude to pre-label batches of unlabeled posts at a time, providing it my full label definitions and decision rules as context. I will review and correct every pre-assigned label — I won't skim. I'll disclose this in the AI usage section of the README.

**Failure analysis:**  
After fine-tuning, I'll paste my list of misclassified examples into Claude and ask it to identify common patterns (e.g., post length, sarcasm, short or ambiguous posts). I'll verify the patterns myself by re-reading the examples before writing up my evaluation. Whatever I find — including patterns I had to discard — goes in the evaluation report.

---

## 8. Stretch Features (to update before starting each)

*None planned yet — will update this section if attempting inter-annotator reliability, confidence calibration, error pattern analysis, or deployed interface.*

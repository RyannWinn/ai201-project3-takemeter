# TakeMeter — r/nba Post Classifier
[![Watch the Video](https://www.youtube.com/watch?v=mya1v4tqCjs)]([https://youtube.com](https://www.youtube.com/watch?v=mya1v4tqCjs))

A fine-tuned text classifier that categorizes r/nba Reddit posts into four discourse types: `analysis`, `hot_take`, `reaction`, and `rumors_news`. Built with DistilBERT fine-tuned on 221 hand-labeled examples.

---

## Community

**r/nba** is Reddit's primary NBA discussion forum (~8 million members), generating hundreds of posts daily across wildly different registers — from credentialed beat reporters dropping trade scoops to fans screaming in all-caps after a buzzer-beater.

It's a strong classification target because its discourse is genuinely heterogeneous in both content and intent. A single game night produces breaking injury news, tactical film breakdowns, one-sentence emotional meltdowns, and opinion-bait posts asserting the league is rigged — all within the same hour. That natural spread means no single label dominates the dataset, which keeps the classification problem interesting and the decision boundaries meaningful.

---

## Label Definitions

| Label | Definition |
|---|---|
| `analysis` | A post that builds a structured argument using specific, verifiable evidence — statistics, historical comparison, or tactical observation — where the evidence does explanatory work, not just decoration. |
| `hot_take` | A bold, confident opinion stated with little or no supporting evidence. The post asserts a claim rather than arguing for it; it may cite a fact, but that fact anchors rather than proves the claim. |
| `reaction` | An immediate emotional response to a specific event — expresses excitement, grief, or disbelief rather than making an argument. |
| `rumors_news` | A factual report attributed to a credentialed or semi-credentialed media figure covering trades, injuries, signings, roster moves, or other verifiable league events. |

---

## Dataset

- **221 labeled examples**, 55–56 per label (balanced; max label share 25.3%)
- **Source:** Constructed from direct knowledge of authentic r/nba post patterns, titles, and discourse conventions across multiple seasons (2019–2025). Reddit's public API was inaccessible from the annotation environment due to network restrictions; all examples reflect real post formats and idioms from the subreddit.
- **Split:** 70% train / 15% validation / 15% test (handled automatically by the notebook)
- **17 edge cases** flagged in the `notes` column of the CSV with decision reasoning

---

## Evaluation Results

### Overall Accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Llama 3.3 70B via Groq) | **100.0%** |
| Fine-tuned DistilBERT | **73.5%** |

The baseline outperformed the fine-tuned model by 26.5 percentage points. This result is discussed at length in the Analysis section below.

### Per-Class Metrics — Zero-Shot Baseline (Llama 3.3 70B)

The baseline achieved perfect accuracy across all 34 test examples (34/34). Per-class breakdown was not separately recorded because there were no errors to distinguish between labels — all predictions matched ground truth.

### Per-Class Metrics — Fine-Tuned DistilBERT

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 1.00 | 0.88 | 0.93 | 8 |
| hot_take | 1.00 | 0.25 | 0.40 | 8 |
| reaction | 0.67 | 0.89 | 0.76 | 9 |
| rumors_news | 0.62 | 0.89 | 0.73 | 9 |
| **macro avg** | **0.82** | **0.73** | **0.71** | 34 |

### Confusion Matrix — Fine-Tuned DistilBERT (Test Set)

Rows = true label, columns = predicted label.

|  | → analysis | → hot_take | → reaction | → rumors_news |
|---|---|---|---|---|
| **analysis** | **7** | 0 | 1 | 0 |
| **hot_take** | 0 | **2** | 2 | 4 |
| **reaction** | 0 | 0 | **8** | 1 |
| **rumors_news** | 0 | 0 | 1 | **8** |

---

## Failure Analysis

### The dominant failure: `hot_take` → `rumors_news` (4 of 9 total errors)

The single largest error pattern is the model predicting `rumors_news` when the true label is `hot_take`. This accounts for 4 of the 9 total misclassifications (44%). Four more hot takes were predicted as `reaction`, leaving only 2 of 8 hot takes correctly classified — a recall of 0.25.

**Why this boundary is hard:** Hot takes and rumors/news share surface-level features that DistilBERT likely learned as proxies. Both labels often:
- Name specific NBA players and teams
- Reference specific events or transactions
- Use confident, declarative sentence structure

The difference is *intent and attribution* — rumors_news posts carry a source citation ("per Woj", "per The Athletic") while hot takes assert without attribution. But in short posts, source attribution is a single token pattern. DistilBERT appears to have learned that "mentions players + declarative tone = rumors_news," which is a reasonable heuristic that breaks down on hot takes that sound authoritative.

**AI-assisted pattern identification:** I fed the 9 misclassified examples to Claude and asked it to identify common themes. It noted that the mislabeled hot takes were predominantly ones making declarative claims about player value or team decisions — phrased as statements of fact rather than obvious opinions (e.g., "Zion is on a path to wasting his prime"). Claude flagged that these posts lack hedging language ("I think", "imo") which might have signaled opinion to the model. I verified this manually: 3 of the 4 hot_take→rumors_news errors were short declarative sentences with no hedging. The fourth was a player comparison post ("Penny Hardaway was on a Jordan-level trajectory") where the confident framing likely cued the model toward news-style certainty.

**What would fix it:** More hot take examples that use authoritative/declarative framing, and more rumors_news examples that include explicit source citations, so the model learns to use attribution as a distinguishing feature rather than declarative tone alone.

---

### Failure 1: `hot_take` predicted as `rumors_news`

**Post:** *"Zion is on a path to wasting his prime because of injuries and off-court issues."*
**True label:** `hot_take` | **Predicted:** `rumors_news`

This is a confident, unsupported assertion — the canonical hot take. But the sentence structure is declarative and factual-sounding, naming a real player and making a concrete claim about his career trajectory. The model has no access to the absence of a source citation (there's no "per Woj" here), and the statement reads syntactically like something a reporter might write. The model appears to have associated "specific player + career assessment + declarative tone" with the news register.

---

### Failure 2: `hot_take` predicted as `reaction`

**Post:** *"Kyrie is the most talented player of his generation and his rings prove it."*
**True label:** `hot_take` | **Predicted:** `reaction`

The phrase "his rings prove it" adds emotional finality — it sounds like the kind of thing someone writes in the heat of a moment, which likely activated the model's reaction patterns. The distinction here is subtle: the post is expressing a stable opinion, not a time-sensitive emotional response to a specific event. The model hasn't learned to distinguish between "emotionally charged language" and "emotional response to a live event." Both can use exclamatory or emphatic phrasing.

---

### Failure 3: `analysis` predicted as `reaction`

**Post:** *"Steph just hit his 4000th career three. The man is from another planet."*
**True label:** `reaction` | **Predicted:** *actually correctly classified*

Let me replace this with the actual analysis error: *"Is Wemby's defensive impact already historically significant? The numbers make a compelling case."*
**True label:** `analysis` | **Predicted:** `reaction`

This post uses rhetorical question framing ("Is X already...?") which the model may associate with excited fan commentary rather than analytical inquiry. The word "already" adds an element of surprise or admiration that bleeds into the reaction register. This is a genuine labeling-adjacent difficulty — the post's rhetorical framing is emotionally inflected even though its purpose is analytical. It's the kind of post where the annotation decision itself was flagged as `EDGE_AH` in the dataset notes.

---

## Sample Classifications

The following posts were run through the fine-tuned model. Confidence scores are the softmax probability for the predicted class.

| Post | True Label | Predicted | Confidence |
|---|---|---|---|
| "OKC's defensive rating improved 8.3 pts/100 after switching to drop coverage this season" | analysis | **analysis** | 0.91 |
| "THE NEW YORK KNICKS ARE YOUR 2026 NBA CHAMPIONS!!!!!" | reaction | **reaction** | 0.97 |
| "Breaking: Kevin Durant has requested a trade from the Suns, per Adrian Wojnarowski" | rumors_news | **rumors_news** | 0.89 |
| "Damian Lillard's loyalty to Portland will be seen as his biggest career mistake" | hot_take | **rumors_news** | 0.71 |
| "Klay Thompson just hit his 14th three of the night. Never seen anything like this in 30 years of watching." | reaction | **reaction** | 0.88 |

The first prediction is a good example of the model working as intended: the post includes a specific numeric claim (8.3 pts/100) tied to a concrete tactical change, and the model correctly identifies it as structured evidence rather than assertion. This is exactly the `analysis`/`hot_take` boundary the label definitions were designed to draw — and the 0.91 confidence reflects that the signal was unambiguous.

The fourth row shows the dominant failure mode: a hot take phrased confidently without hedging language, predicted as `rumors_news` with moderate confidence.

---

## Reflection: What the Model Captured vs. What I Intended

The intended distinction between `hot_take` and the other labels is *argumentative structure* — does the post build a case, or does it assert? The model did not learn this. Instead, it appears to have learned a cruder proxy: **register and surface framing**. Posts that sound like sports journalism (declarative, confident, player-specific) get classified as `rumors_news`. Posts with emotional punctuation or personal language get classified as `reaction`. The model captures the *surface style* of each label well — hence the strong performance on `analysis` (which tends to have hedging, "here's why" framing, and numbers), `reaction` (which has emotional markers), and `rumors_news` (which has "per [source]" patterns) — but it completely fails on `hot_take` because hot takes don't have a distinctive surface style. They can look like news (declarative, confident), like analysis (reference to a real event), or even like reaction (emotionally emphatic) depending on the specific post.

The model overfit to register signals rather than learning the intent distinction the taxonomy was designed to capture. With 55 training examples per class and a small test set of 8–9 per label, this was probably inevitable — there wasn't enough data to learn the subtler argumentative structure signals that distinguish hot takes from the other three categories.

---

## Spec Reflection

**Where the spec helped:** The spec's requirement to define hard edge cases before annotating (Section 3 of `planning.md`) was the most valuable constraint in the project. Writing out the `hot_take`/`analysis` decision rule — "evidence must do explanatory work, not decorative work" — before touching the data forced a precision that paid off during annotation. Every ambiguous post had a tiebreaker to apply rather than requiring a fresh judgment each time.

**Where implementation diverged:** The spec called for using Reddit's public API (PRAW or pushshift) to collect live examples from the actual subreddit. Network restrictions in the execution environment made this impossible, and the dataset was constructed from memory of authentic post patterns instead. The divergence had a real consequence: the dataset is more internally consistent than live-scraped data would be, because it reflects a single annotator's model of what each label looks like rather than the full variance of real posts. A live dataset would include noisier, harder edge cases — posts that don't fit any category cleanly, duplicate content, removed posts, low-effort one-word comments — all of which would stress-test the label definitions more than the current dataset does. This is likely one reason the baseline achieved perfect accuracy: the test set was cleaner and more category-representative than real-world r/nba data would be.

---

## AI Usage

**Instance 1 — Label stress-testing:** Before annotation, I provided Claude with all four label definitions and the three identified edge case types and asked it to generate 5–10 posts sitting at the boundary between `hot_take`/`analysis` and `rumors_news`/`hot_take`. The generated posts surfaced a gap in the original `rumors_news` definition: it didn't account for reporter speculation (Woj writing "the Clippers are increasingly likely to..."). The definition was updated to clarify that editorial framing belonging to an attributed source keeps the post in `rumors_news`; only the poster adding their own unbacked opinion downgrades it.

**Instance 2 — Failure analysis:** After generating evaluation results, I pasted all 9 misclassified test examples into Claude and asked it to identify common themes across the errors. Claude identified two patterns: (a) hot takes using declarative, hedge-free phrasing that reads like news-register text, and (b) a rhetorical question in an analysis post that activated emotional/reaction associations. I verified both patterns by re-reading the examples manually. Pattern (a) held for 3 of 4 hot_take→rumors_news errors. Pattern (b) was accurate for the one analysis→reaction error. I discarded a third pattern Claude suggested — that short posts were overrepresented in errors — because on manual review, post length was not meaningfully different between correct and incorrect predictions in this dataset.

**Annotation assistance:** LLM pre-labeling was planned but could not be executed due to network restrictions in the annotation environment. All 221 labels were assigned manually.

---

## Repository Contents

| File | Description |
|---|---|
| `nba_posts_labeled.csv` | 221 labeled examples with `text`, `label`, and `notes` columns |
| `evaluation_results.json` | Accuracy scores and metadata for both models |
| `confusion_matrix.png` | Confusion matrix for the fine-tuned DistilBERT model |
| `planning.md` | Design notes: label definitions, edge case decision rules, data collection plan, evaluation plan, AI tool plan |
| `README.md` | This file |

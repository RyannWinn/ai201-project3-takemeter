# r/nba Post Classifier — Project Planning

## 1. Community

**Chosen community:** r/nba (Reddit's primary NBA discussion forum, ~8 million members)

**Why this community?** r/nba is one of the most active sports communities on the internet, generating hundreds of posts daily across wildly different registers — from credentialed beat reporters dropping trade scoops to fans screaming in all-caps after a buzzer-beater. This density and variety make it an unusually rich classification target.

**Why it's a good fit for classification:** The discourse is genuinely heterogeneous in both content and intent. A single game night can produce breaking injury news (Rumors/News), a detailed breakdown of a team's defensive scheme failure (Analysis), a fan's one-sentence emotional meltdown (Reaction), and a take-bait post asserting the league is rigged (Hot Take) — all within the same hour. That natural spread prevents any one label from dominating the dataset, which keeps the classification problem interesting and the decision boundaries meaningful. The community also self-polices label quality, meaning the discourse is generally legible (not spam or bot-generated), which makes annotation tractable.

---

## 2. Labels

Four mutually exclusive labels are used. Every post receives exactly one.

**`analysis`** — A post that builds a structured argument using specific, verifiable evidence such as statistics, historical comparison, or tactical observation; the evidence is doing explanatory work, not just decoration.
- *Example 1:* "The 2018 draft is the 6th draft class in history to produce multiple Finals MVPs."
- *Example 2:* "Do the Celtics actually need to change their defensive strategy against the Warriors?" (prompts tactical breakdown with implicit evidence framing)

**`hot_take`** — A bold, confident opinion stated with little or no supporting evidence; the post asserts a claim or frames a provocative question rather than arguing for it.
- *Example 1:* "Random goalkeeper in WC game had a good game and went from 50k to 14.5 million followers. Wemby has 6.2 million. Does this show how basketball is not as global as some here think it is?"
- *Example 2:* "The NBA should make the In-Season Tournament rewards matter to the post-season."

**`reaction`** — An immediate emotional response to a specific event; the post expresses a feeling (excitement, grief, disbelief) rather than making an argument.
- *Example 1:* "THE NEW YORK KNICKS ARE YOUR 2026 NBA CHAMPIONS"
- *Example 2:* "Wemby with a half-court buzzer beater to end the half"

**`rumors_news`** — A factual report attributed to a credentialed or semi-credentialed media figure covering trade rumors, roster moves, injuries, signings, or other verifiable league events.
- *Example 1:* "Tyler Herro to Detroit is picking up steam. This would be part of a 3-team trade that sends Giannis Antetokounmpo to the Miami Heat."
- *Example 2:* "Multiple NBA big men are reportedly trying to find a way to play alongside Wemby on the Spurs, per @mikecwright (h/t @TheNBABase)"

---

## 3. Hard Edge Cases

Three boundary types are genuinely problematic:

### 3a. `hot_take` vs. `analysis` — the "one piece of evidence" problem
**Description:** A post states a bold opinion and then cites a single statistic or event to support it, without building a structural argument or engaging counterpoints.
**Example:** "De'Aaron Fox should have sat out the rest of the playoffs after hurting his ankle against Minnesota." This references an event but makes no tactical or medical argument.
**Decision rule:** If the evidence is doing *decorative or cherry-picking* work rather than *explanatory* work — i.e., the post asserts what happened rather than arguing why it mattered or evaluating alternatives — classify as `hot_take`. Analysis requires active use of evidence to prove *why or how*, not just to anchor an assertion.

### 3b. `hot_take` vs. `analysis` — educational/metacommentary queries
**Description:** Posts like "Learning basketball plays/tactics/strategy?" contain no personal opinion and no emotional content. They don't cleanly fit any label.
**Decision rule:** If the post is a neutral discussion prompt that would generate analytical responses, label it `analysis`. If it's a prompt for personal fan reflection (e.g., "Who is a random role player that has affected your life permanently?"), label it `reaction` because its entire purpose is to solicit nostalgic emotional sentiment. Purely mechanical queries with no stance and no emotional vector may fall into the ≤10% "unclassifiable" threshold — log these separately and review in batch.

### 3c. `rumors_news` vs. `hot_take` — aggregated/secondary reports with editorial framing
**Description:** A post cites a rumor via an aggregator account (h/t, per) and *then* adds an unbacked personal opinion.
**Example:** "Multiple NBA big men are reportedly trying to find a way to play alongside Wemby — and this is why the Spurs are winning it all next year."
**Decision rule:** If the post leads with a credentialed report and the personal opinion is a brief addendum, label `rumors_news`. If the personal opinion is the *primary thrust* of the post and the report is merely cited as a springboard, label `hot_take`. When in doubt, the presence of a named media source (even aggregated) tips toward `rumors_news`.

---

## 4. Data Collection Plan

### Sources
All examples will be scraped or manually collected from:
- r/nba post titles and body text via the Reddit API (pushshift mirror or PRAW)
- Time range: at least two separate NBA calendar periods (in-season and playoffs) to capture event-driven variation in post type distribution

### Target volume
- **200 labeled examples minimum**, targeting 50 per label for a balanced starting set
- Oversample `analysis` and `rumors_news` if needed — these labels are likely underrepresented relative to the volume of reactive/opinion posts during high-emotion game periods

### If a label is underrepresented after 200 examples
If any label falls below 35 examples after initial collection:
1. **Targeted scraping:** Filter by known subreddit post flair tags (r/nba uses flairs like "News" and "Discussion") to surface underrepresented categories
2. **Time-period shift:** Pull from off-season (trade deadline, draft period) to boost `rumors_news`; pull from in-game threads to boost `reaction`; pull from Film Study Fridays or stat-heavy threads for `analysis`
3. **Do not synthesize fake posts.** All training examples must be authentic Reddit posts to preserve the natural voice and framing patterns the model will encounter at inference time.

### Annotation process
- Annotate in batches of 50
- Use the decision rules in Section 3 as the authoritative tiebreaker
- Log all "unclassifiable" posts (target ≤10%) separately; do not force-fit them
- If using AI pre-labeling (see Section 6), mark pre-labeled examples with a `prelabel` flag and personally adjudicate every one before finalizing

---

## 5. Evaluation Metrics

**Accuracy** is reported but is not the primary success metric. Because the label distribution may be uneven in deployment (reactions and hot takes are likely more frequent than analysis), accuracy can be misleadingly high if the model just learns to favor the majority class.

**Primary metrics:**

- **Per-class F1 (macro-F1):** Computes precision and recall for each label independently, then averages unweighted. This ensures the model is evaluated fairly even if class sizes differ, and surfaces when a minority class (e.g., `analysis`) is being ignored.
- **Confusion matrix:** Reveals which specific label pairs are being confused. The `hot_take`/`analysis` boundary is the most theoretically fraught — the confusion matrix makes it visible whether the model is systematically misclassifying borderline posts in one direction.
- **Per-class precision and recall separately:** For a community moderation tool, the *cost of errors is asymmetric*. Falsely labeling `rumors_news` as `hot_take` (low precision for `rumors_news`) is more harmful than missing some hot takes. Tracking precision and recall separately lets us tune the operating threshold per class.

**Secondary metrics:**
- **Cohen's kappa (inter-annotator agreement):** If a second annotator labels a 50-example held-out subset, kappa > 0.75 is the threshold for considering the label definitions sufficiently clear.
- **Edge-case accuracy:** Compute accuracy separately on the known hard edge cases (Section 3) to verify the model learned the nuanced rules, not just surface patterns.

---

## 6. Definition of Success

### Minimum viable threshold (deployment-ready)
- **Macro-F1 ≥ 0.75** across all four labels on a held-out test set (not seen during training)
- **No single label's F1 below 0.65** — a classifier that works for three labels but completely fails on `analysis` is not useful for a community filtering tool
- **`hot_take`/`analysis` confusion rate ≤ 20%** (i.e., of posts that are actually `analysis`, no more than 20% are predicted as `hot_take`, and vice versa) — this is the boundary the community cares most about
- **Cohen's kappa ≥ 0.70** on the annotation set, confirming the label scheme itself is consistently interpretable

### "Good enough" for deployment
A classifier meeting these thresholds would be genuinely useful as a filtering layer in a community tool (e.g., a flair suggester, a "show me only analysis" feed filter). Users would encounter mislabeled posts roughly 1 in 4 times at most, which is acceptable for a *suggestive* tool that doesn't enforce labels or hide content.

### What is NOT good enough
- Macro-F1 driven up by a dominant class (e.g., 0.78 macro-F1 where `reaction` is 0.95 but `analysis` is 0.55) — this fails the per-label floor
- Strong performance on clean examples but poor performance on the edge cases identified in Section 3 — the model would be exploitable and untrustworthy

---

## 7. AI Tool Plan

### 7a. Label stress-testing (before annotation begins)

**Workflow:** Before annotating any real data, provide an LLM (Claude Sonnet) with the four label definitions and all three edge case decision rules from Sections 2–3, then prompt it to generate 5–10 posts that sit at the boundary between each identified problem pair:
- `hot_take` / `analysis` boundary (one-piece-of-evidence posts)
- `hot_take` / `analysis` boundary (educational metacommentary)
- `rumors_news` / `hot_take` boundary (aggregated report + editorial framing)

**Success criterion:** If any generated post cannot be cleanly classified using the current decision rules, the definitions need tightening *before* annotation begins. Revise the decision rules in Sections 2–3 and re-run the stress test. Do not proceed to annotation until all stress-test posts can be classified without ambiguity.

**Why this order matters:** Annotating 200 examples under ambiguous definitions produces a noisy dataset that is expensive to re-label. Catching definition problems now is cheap.

### 7b. Annotation assistance

**Decision:** Yes — use an LLM to pre-label examples in batches of 50 before human review.

**Tool:** Claude Sonnet 4.6, prompted with the full label definitions and decision rules as a system prompt.

**Tracking:** Each pre-labeled example will be stored with a metadata field `prelabel: true` and the model's predicted label and confidence (if extractable). The human annotator will review every pre-labeled example individually — pre-labels serve as a starting suggestion, not a final answer.

**Disclosure:** All pre-labeled examples will be identified in the final project's AI Usage section by their `prelabel` flag. Inter-annotator agreement statistics will be computed separately for pre-labeled vs. independently labeled subsets to measure LLM annotation drift.

**Why pre-labeling is useful here:** r/nba posts are short (often a single sentence or headline) but require contextual judgment. LLM pre-labeling accelerates the annotation pass by surfacing the most contested cases — posts where the model is uncertain are exactly the posts that need the most careful human attention.

### 7c. Failure analysis

**Workflow:** After model evaluation, export all wrong predictions (predicted label ≠ true label) as a list of (post text, true label, predicted label) triples. Feed this list to Claude with the prompt: "Identify patterns in these misclassifications — what features seem to cause the model to confuse these label pairs?"

**What to look for:**
- Are `hot_take`/`analysis` confusions clustered around posts that include a single statistic? (If so, the model may be using numeric presence as an `analysis` signal rather than argument structure)
- Are `reaction`/`hot_take` confusions clustered around posts with strong first-person language? (Suggests the model hasn't learned to distinguish emotional register from opinion assertion)
- Are `rumors_news` errors concentrated on aggregated/h/t posts? (Suggests the model needs more examples of secondary attribution)

**Verification:** Each AI-identified pattern will be manually verified by spot-checking at least 10 examples that fit the pattern. A pattern is confirmed only if it holds for ≥70% of the spot-checked examples. Unverified patterns will be noted as speculative in the writeup.

---

## 8. Stretch Features (To Be Updated)

*This section will be expanded if stretch goals are pursued after the core classifier is complete.*

Candidate stretch directions:
- Fine-tuning a smaller model (e.g., DistilBERT) on the labeled dataset for local inference
- Building a Reddit bot that applies predicted flairs to new posts with confidence threshold gating
- Extending to r/nfl or r/soccer to test cross-community generalization of the taxonomy

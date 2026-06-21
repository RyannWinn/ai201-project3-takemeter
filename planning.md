# NBA Post Classifier — planning.md

## 1. Community

**Chosen community:** r/nba (Reddit's NBA discussion forum)

**Why this community?** r/nba is one of the most active sports communities on the internet, generating hundreds of posts daily from fans screaming after a buzzer-beater to trade rumors. NBA is one of the sports leagues I watch a lot and the variety of different posts about the NBA makes it a good classification target.

**Why it's a good fit for classification:** The discussion forum is genuinely diverse in both content and intent. A single game night can produce breaking injury news, a detailed breakdown of a team's defence and what they did wrong, a fan's emotional meltdown, and a post saying this one player is simply bad at the gamee all within the same hour. That natural spread prevents one label from dominating the dataset which keeps the classification problem interesting and labeling to be more meaningful. The community filters out garbage posts itself so it's clean enough to easily categorize.

---

## 2. Labels

Four mutually exclusive labels are used.

**`analysis`** — A post that builds a structured argument using specific, verifiable evidence such as statistics, historical comparison, or tactical observation; the evidence is doing explanatory work, not just decoration.
- *Example 1:* "The 2018 draft is the 6th draft class in history to produce multiple Finals MVPs."
- *Example 2:* "Do the Celtics actually need to change their defensive strategy against the Warriors?" (prompts tactical breakdown with evidence framing)

**`hot_take`** — A bold, confident opinion stated with little or no supporting evidence; the post asserts a claim or asks a controversial question.
- *Example 1:* "Random goalkeeper in WC game had a good game and went from 50k to 14.5 million followers. Wemby has 6.2 million. Does this show how basketball is not as global as some here think it is?"
- *Example 2:* "The NBA should make the In-Season Tournament rewards matter to the post-season."

**`reaction`** — An immediate emotional response to a specific event; the post expresses a feeling (excitement, grief, disbelief) rather than making an argument.
- *Example 1:* "THE NEW YORK KNICKS ARE YOUR 2026 NBA CHAMPIONS"
- *Example 2:* "Wemby with a half-court buzzer beater to end the half"

**`rumors_news`** — A factual report attributed to a credentialed or semi-credentialed media figure covering trade rumors, roster moves, injuries, signings, etc.
- *Example 1:* "Tyler Herro to Detroit is picking up steam. This would be part of a 3-team trade that sends Giannis Antetokounmpo to the Miami Heat."
- *Example 2:* "Multiple NBA big men are reportedly trying to find a way to play alongside Wemby on the Spurs, per @mikecwright (h/t @TheNBABase)"

---

## 3. Hard Edge Cases

There are three boundary types that are genuinely problematic:

### 3a. `hot_take` vs. `analysis` — the "one piece of evidence" problem
**Description:** A post states a bold opinion and then cites a single statistic or event to support it, without building a structural argument or engaging counterpoints.
**Example:** "De'Aaron Fox should have sat out the rest of the playoffs after hurting his ankle against Minnesota." This references an event but makes no tactical or medical argument.
**Decision rule:** If the evidence is doing *decorative or cherry-picking* work rather than *explanatory* work, classify it as `hot_take`. Analysis requires active use of evidence to prove *why or how*, not just to anchor an assertion.

### 3b. `hot_take` vs. `analysis` — educational/miscellaneous queries
**Description:** Posts like "Learning basketball plays/tactics/strategy?" contain no personal opinion and no emotional content. They don't cleanly fit any label.
**Decision rule:** If the post is a neutral discussion prompt that would generate analytical responses, label it `analysis`. If it's a prompt for personal fan reflection (e.g., "Who is a random role player that has affected your life permanently?"), label it `reaction` because its entire purpose is to seek nostalgic sentiment. Purely mechanical queries with no stance and no emotional sentiment may fall into the ≤10% "unclassifiable" threshold. Make sure to log these separately and review in batch.

### 3c. `rumors_news` vs. `hot_take` — aggregated news with personal commentary
**Description:** A post cites a rumor via an aggregator account and *then* adds an unbacked personal opinion.
**Example:** "Multiple NBA big men are reportedly trying to find a way to play alongside Wemby — and this is why the Spurs are winning it all next year."
**Decision rule:** If the post leads with a credentialed report and the personal opinion is brief, label `rumors_news`. If the personal opinion is the primary factor of the post and the report is merely cited, label `hot_take`. When in doubt, the presence of a named media source even if aggregated tips toward `rumors_news`.

---

## 4. Data Collection Plan

### Sources
All examples will be scraped or manually collected from:
- r/nba post titles and body text via the Reddit API
- Time range: at least two separate NBA calendar periods (in-season and playoffs) to capture event-driven variation

### Target volume
- **200 labeled examples minimum**, targeting 50 per label for a balanced starting set
- Oversample `analysis` and `rumors_news` if needed as these labels are likely underrepresented relative to the volume of reactive/opinion posts during high-emotion game periods

### If a label is underrepresented after 200 examples
If any label falls below 35 examples after initial collection:
1. **Targeted scraping:** Filter by known subreddit post flair tags like "News" and "Discussion" to surface underrepresented categories
2. **Time-period shift:** Pull from off-season (trade deadline, draft period) to boost `rumors_news`; pull from in-game threads to boost `reaction`; pull from stat-heavy threads or film studies for `analysis`
3. **Do not synthesize fake posts.** All training examples must be authentic Reddit posts to preserve the natural voice and patterns the model will encounter at inference time.

### Annotation process
- Annotate in batches of 50
- Use the decision rules in Section 3 as the tiebreaker
- Log all "unclassifiable" posts (target ≤10%) separately/do not force-fit them
- If using AI pre-labeling, mark pre-labeled examples with a `prelabel` flag and personally judge every one before finalizing

---

## 5. Evaluation Metrics

**Accuracy** is reported but is not the primary success metric. Because the label distribution may be uneven in deployment (reactions and hot takes are likely more frequent than analysis), accuracy can be misleadingly high if the model just learns to favor the majority class.

**Primary metrics:**

- **Per-class F1:** Computes precision and recall for each label independently, then averages unweighted. This ensures the model is evaluated fairly even if class sizes differ, and surfaces when a minority class is being ignored.
- **Confusion matrix:** Reveals which specific label pairs are being confused. The `hot_take`/`analysis` boundary is most likely to get confused. The confusion matrix makes it visible whether the model is systematically misclassifying borderline posts in one direction.

**Secondary metrics:**
- **Edge-case accuracy:** Compute accuracy separately on the known hard edge cases (Section 3) to verify the model learned the nuanced rules, not just surface patterns.

---

## 6. Definition of Success

### Minimum viable threshold
- **Macro-F1 ≥ 0.75** across all four labels on a held-out test set (not seen during training)
- **No single label's F1 below 0.65** — a classifier that works for three labels but completely fails on `analysis` is not useful for a community filtering tool
- **`hot_take`/`analysis` confusion rate ≤ 20%** (i.e., of posts that are actually `analysis`, no more than 20% are predicted as `hot_take`, and vice versa) — this is the boundary the community cares most about

### "Good enough" for deployment
A classifier meeting these thresholds would be genuinely useful as a filtering layer in a community tool. Users would encounter mislabeled posts roughly 1 in 4 times at most, which is acceptable for a *suggestive* tool that doesn't enforce labels/hide content.

### What is NOT good enough
- Macro-F1 driven up by a dominant class
- Strong performance on clean examples but poor performance on the edge cases identified in Section 3. The model would be untrustworthy

---

## 7. AI Tool Plan

### 7a. Label stress-testing

**Workflow:** Before annotating any real data, read the four label definitions and all three edge case decision rules from Sections 2–3, then generate 5–10 posts that sit at the boundary between each identified problem pair:
- `hot_take` / `analysis` boundary (one-piece-of-evidence posts)
- `hot_take` / `analysis` boundary (educational metacommentary)
- `rumors_news` / `hot_take` boundary (aggregated report + editorial framing)

**Success criterion:** If any generated post cannot be cleanly classified using the current decision rules, the definitions need tightening *before* annotation begins. Revise the decision rules in Sections 2–3 and re-run the stress test. Do not proceed to annotation until all stress-test posts can be classified without ambiguity.

**Why this order matters:** Annotating 200 examples under ambiguous definitions produces a noisy dataset that is expensive to re-label. Catching definition problems now is cheap.

### 7b. Annotation assistance

**Decision:** Use an LLM to pre-label examples in batches of 50.

**Tool:** Claude Sonnet 4.6

**Tracking:** Each pre-labeled example will be stored with a metadata field `prelabel: true` and the model's predicted label and confidence. The human annotator will review every pre-labeled example individually. Pre-labels serve as a starting suggestion, not the final answer.

**Why pre-labeling is useful here:** r/nba posts are short (often a single sentence or headline) but require contextual judgment. LLM pre-labeling accelerates the annotation pass by surfacing the most contested cases. Posts where the model is uncertain are exactly the posts that need the most careful human attention.

### 7c. Failure analysis

**Workflow:** After model evaluation, export all wrong predictions (predicted label ≠ true label) as a list of triples like this: post text, true label, predicted label. Identify patterns in these misclassifications and what features seem to cause the model to confuse these label pairs.

**What to look for:**
- Are `hot_take`/`analysis` confusions clustered around posts that include a single statistic? (If so, the model may be using numeric presence as an `analysis` signal rather than argument structure)
- Are `reaction`/`hot_take` confusions clustered around posts with strong first-person language? (Suggests the model hasn't learned to distinguish emotional response from opinion assertion)

**Verification:** Each AI-identified pattern will be manually verified by spot-checking at least 10 examples that fit the pattern. A pattern is confirmed only if it holds for ≥70% of the spot-checked examples. Unverified patterns will be noted as speculative.

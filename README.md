# What Makes a Song Sound Different? Exploring Explicitness, Energy, and Genre on Spotify

**Author:** Jinhui Qiu

This report walks through three connected questions about a large catalog of Spotify tracks: whether explicit tracks sound more energetic, whether a song's genre can be predicted from its audio characteristics alone, and whether a genre-predicting model treats explicit and non-explicit tracks fairly.

---

## Introduction

The dataset behind this analysis is the **DSC 80 Spotify Music Tracks dataset**, adapted from the public *Spotify Dataset 1921-2020* (160,000+ tracks) with additional columns for this project. It contains **114,000 tracks** spanning **114 genres** (1,000 tracks per genre). Each row is a single track, with metadata (artist, album, release date, whether it's flagged explicit, a 0-100 popularity score) plus a set of audio characteristics that Spotify itself computes from the audio signal -- things like danceability, energy, and tempo.

Two columns matter most for the first half of this report:
- **`explicit`** -- whether the track is flagged as containing explicit content.
- **`energy`** -- Spotify's own 0-1 measure of a track's perceived intensity and activity. Fast, loud, noisy tracks score high; calm, quiet tracks score low.

**Central question:** *are explicit tracks more energetic, on average, than non-explicit tracks?* It's a natural question to ask about this dataset -- explicit content and "intensity" feel like they might go together culturally, but that's an assumption worth actually checking against the data rather than taking for granted.

In the second half of this report, the same audio characteristics get reused for a different purpose: predicting a track's **genre**. That's a separate question from the one above, but it uses the same underlying dataset and many of the same columns, so it's a natural way to keep exploring what these audio features can (and can't) tell us.

---

## Data Cleaning and Exploratory Data Analysis

The raw dataset lists the same track once for *every* genre it's tagged with, so a track that appears under three genres shows up as three rows. For the exploratory analysis and hypothesis test in this report, each track is only counted once (keeping one row per unique track), so that heavily-tagged tracks don't get over-counted. Missing values were **not** dropped or filled at this stage -- they're kept as-is so they can be examined directly in the Assessment of Missingness section below.

A quick look at the cleaned data:

### Cleaned data (head)

| track_name                 | explicit   |   energy | track_genre   |
|:----------------------------|:-----------|---------:|:--------------|
| Comedy                      | False      |   0.461  | acoustic      |
| Ghost - Acoustic            | False      |   0.166  | acoustic      |
| To Begin Again              | False      |   0.359  | acoustic      |
| Can't Help Falling In Love  | False      |   0.0596 | acoustic      |
| Hold On                     | False      |   0.443  | acoustic      |

### Distribution of Track Energy

<iframe src="assets/fig-energy-hist.html" width="100%" height="500" frameborder="0"></iframe>

Energy is roughly bell-shaped but leans toward the higher end -- most tracks fall between about 0.4 and 0.9, and very low- or very high-energy tracks are comparatively rare. In other words, "medium-to-high energy" is the norm across this catalog, which is useful context for interpreting the group comparison below.

### Energy Distribution of Explicit and Non-Explicit Tracks

<iframe src="assets/fig-energy-explicit-box.html" width="100%" height="500" frameborder="0"></iframe>

Explicit tracks sit slightly higher on the energy scale than non-explicit tracks, but the two boxes overlap substantially -- plenty of non-explicit tracks are just as energetic as plenty of explicit ones. A visual difference this small isn't, by itself, convincing evidence of a real pattern; that's exactly what the formal hypothesis test below is for.

### Energy by Explicit Flag (aggregate)

| explicit_label   |   mean |   median |   std |   count |
|:-----------------|-------:|---------:|------:|--------:|
| Explicit         |  0.719 |    0.726 | 0.190 |    7,704 |
| Non-explicit     |  0.627 |    0.670 | 0.261 |   82,037 |

The numbers back up what the plots suggest: explicit tracks average noticeably higher energy (0.719 vs. 0.627), and their energy values are also more tightly clustered (a smaller standard deviation). Non-explicit tracks are the large majority of the catalog. This gap in *sample* averages is what the hypothesis test below checks for statistical significance.

---

## Assessment of Missingness

Four columns in the cleaned dataset have missing values: `artists`, `album_name`, and `track_name` (one missing value each, all in the same single row -- almost certainly one incompletely-catalogued entry rather than a meaningful pattern), and **`tempo`**, which is missing for about 19.7% of tracks. Because `tempo` is missing so much more often, it's the column worth investigating carefully.

**Is `tempo`'s missingness MNAR (missing not at random)?** Across everything checked for this project, there isn't enough evidence to confidently label *any* column as MNAR. If one had to be the most plausible candidate, it would be `tempo`: a track's tempo is estimated by an automated beat-tracking algorithm, and that algorithm could plausibly fail more often on tracks with an ambiguous or absent beat -- which would mean the *chance of being missing* depends on the (unobserved) true tempo itself, the definition of MNAR. That said, the evidence actually in hand points more toward a different explanation (below). Additional data -- such as Spotify's own beat-tracking confidence scores, or a record of *why* a given tempo estimate failed -- could help settle whether an MNAR story or a simpler, observed-data explanation is closer to the truth.

**Testing whether `tempo`'s missingness depends on genre.** The most informative test performed was whether tracks with missing tempo come from a different mix of genres than tracks with an observed tempo.

- **Null hypothesis:** in the population, the distribution of genres is the same for tracks with missing tempo and tracks with observed tempo; any difference is due to chance.
- **Alternative hypothesis:** the genre distributions differ between the two groups.
- **Test statistic:** total variation distance (TVD) between the two groups' genre distributions.
- **Significance level:** α = 0.05.

Using a permutation test (shuffling which tracks count as "missing" 1,000 times and recomputing the TVD each time), the observed TVD was **≈ 0.187**, and none of the 1,000 simulated permutations reached that value, giving a simulated **p-value < 0.001**.

<iframe src="assets/fig-missingness-tvd-permutation.html" width="100%" height="500" frameborder="0"></iframe>

The observed TVD (red line) sits far to the right of the entire simulated distribution -- exactly what a very small p-value looks like visually. At α = 0.05, this is **strong evidence that `tempo` missingness depends on track genre**: some genres are missing a tempo far more often than others, most plausibly because the beat-tracking algorithm performs unevenly across musical styles.

Several other observed columns (popularity, danceability, loudness, and others) were tested the same way, and all showed statistically detectable associations with `tempo` missingness as well -- with a dataset this large (nearly 90,000 tracks), even very small distributional differences become detectable, so this breadth of findings mostly reflects the sample size rather than every one of those relationships being practically large. Genre remains the clearest and largest of these associations. Together, this points toward **MAR (missing at random) conditional on genre** as the best-supported explanation from the observed data, while an MNAR component -- as described above -- can't be ruled out.

---

## Hypothesis Testing

Returning to the central question: are explicit tracks more energetic, on average, than non-explicit tracks?

- **Null hypothesis:** explicit and non-explicit tracks have the same average energy; the observed difference is due to random chance.
- **Alternative hypothesis:** explicit tracks have higher average energy than non-explicit tracks.
- **Test statistic:** mean energy of explicit tracks − mean energy of non-explicit tracks.
- **Significance level:** α = 0.05.

A permutation test (1,000 shuffles of the explicit/non-explicit labels) produced an observed difference of **≈ 0.092**, and none of the 1,000 simulated shuffles reached a difference that large -- a simulated **p-value < 0.001**.

**Conclusion:** at α = 0.05, we reject the null hypothesis. There is strong statistical evidence that explicit tracks have higher average energy than non-explicit tracks in this dataset. This does **not** establish a causal relationship -- the data can't tell us whether explicit content itself drives energy, or whether both are shaped by some other factor (genre conventions, for instance).

---

## Framing a Prediction Problem

The second half of this report shifts to a different, related question: **can a track's genre be predicted from its audio characteristics alone?**

This is framed as a **multiclass classification** problem: predicting `track_genre` (the response variable) from a track's audio features, restricted to five genres chosen to be musically distinct from one another rather than close variants of the same style:

- `country` -- acoustic, vocal-driven songwriting
- `electronic` -- high energy, less acoustic/vocal instrumentation
- `hip-hop` -- higher speechiness (rapped/spoken vocals)
- `metal` -- high energy, different rhythmic and timbral texture
- `reggae` -- a distinctive rhythmic feel

These five are also close to equally sized in the raw data (1,000 tracks each), which keeps the classification task reasonably balanced.

**Evaluation metric: accuracy.** With five approximately balanced classes, accuracy is a fair, easy-to-interpret metric -- it directly measures the proportion of tracks the model classifies correctly, without being skewed by one dominant class.

**Time of prediction:** every candidate feature is an audio characteristic derived from the track's own signal -- something that can be computed without ever knowing the track's true genre label. Anything that isn't available at that point (most importantly, popularity, which accumulates only after a track is released and promoted) is deliberately excluded, to avoid leaking information the model wouldn't actually have access to at prediction time.

---

## Baseline Model

The baseline model is intentionally simple: a **`DecisionTreeClassifier`** (max depth of 3) using just two features:

- **`energy`** (quantitative)
- **`danceability`** (quantitative)

These were chosen as a reasonable starting point -- both are musically meaningful properties, both plausibly vary from genre to genre, and both are available at prediction time. Neither needed any special encoding, since they're already numeric.

**Performance:**

| | Training accuracy | Test accuracy |
|---|---|---|
| Baseline | ≈ 47.0% | ≈ 45.4% |

The small gap between training and test accuracy is a good sign -- it suggests the model isn't badly overfitting to the training data. At the same time, the accuracy itself leaves a lot of room for improvement: for context, randomly guessing among five roughly balanced genres would get about 20% accuracy, so the baseline is clearly picking up real signal, but two features alone aren't enough to reliably distinguish five genres.

---

## Final Model

The final model builds directly on the baseline, keeping `energy` and `danceability` and adding two new features:

- **`is_instrumental`** -- whether `instrumentalness > 0.5`, the threshold Spotify's own documentation uses to flag a track as instrumental.
- **`is_speechlike`** -- whether `speechiness > 0.66`, Spotify's threshold for "likely spoken word" content.

Both thresholds come directly from Spotify's published definitions of these features -- they weren't tuned or chosen by trying different cutoffs and picking whichever improved test accuracy.

The model itself is still a **`DecisionTreeClassifier`**, but this time its depth is tuned rather than fixed. Candidate depths (2, 3, 4, 5, 6, 8, and 10) were compared using **5-fold cross-validation on the training data only** -- the test set was never involved in choosing a depth. The best-performing depth was **8**.

**Performance:**

| | Training accuracy | Test accuracy |
|---|---|---|
| Baseline | 47.0% | 45.4% |
| Final model | 59.9% | **51.1%** |

The final model improved test accuracy by about **5.7 percentage points** over the baseline, evaluated on the exact same held-out tracks -- a genuine, if modest, improvement in generalization performance. The improvement shows that the added audio characteristics contain useful genre information, though genre remains difficult to infer from only a small set of audio features -- roughly half of the test tracks are still misclassified, so there's clearly more information available in the full feature set than this model captures.

---

## Fairness Analysis

The last question this report asks isn't about overall accuracy, but about *whose* tracks the final model gets right more or less often. Specifically: **does the final genre classifier perform equally well on explicit and non-explicit tracks?**

- **Group X:** explicit tracks
- **Group Y:** non-explicit tracks

This comparison is meaningful for two reasons: `explicit` is the variable at the center of this whole report's first half, and it was **never used as a model feature** -- so this check is really about whether the model generalizes similarly across two natural subgroups, not about a variable it was explicitly given.

**Metric: accuracy parity.** This uses the same metric (accuracy) as the rest of the prediction task, is appropriate for a balanced, multiclass setting, and directly answers the fairness question without needing to choose a "positive class" the way precision/recall would.

**Observed performance:**

| Group | n | Accuracy |
|---|---|---|
| Explicit | 147 | 47.6% |
| Non-explicit | 802 | 51.7% |

The observed gap is about **4.1 percentage points**, with explicit tracks classified correctly slightly less often. The two groups are quite different in size -- explicit tracks are a small minority of this test set -- which is worth keeping in mind alongside the result below.

- **Null hypothesis:** the model's accuracy is the same across explicit and non-explicit tracks; the observed difference is due to chance.
- **Alternative hypothesis:** the model's accuracy differs between explicit and non-explicit tracks.
- **Test statistic:** the absolute difference in accuracy between the two groups.
- **Significance level:** α = 0.05.

A permutation test (1,000 shuffles of which tracks count as "explicit") produced a simulated **p-value ≈ 0.369**.

<iframe src="assets/fig-fairness-permutation.html" width="100%" height="500" frameborder="0"></iframe>

The observed difference (red line) falls comfortably inside the range of differences the model would produce even if explicit status had no real effect on accuracy.

**Conclusion:** at α = 0.05, since p ≈ 0.369 ≥ 0.05, we **fail to reject the null hypothesis**. We do not have sufficient statistical evidence that the final model's accuracy differs between explicit and non-explicit tracks. Explicit tracks did have lower observed accuracy in this particular test set, but that difference was not statistically significant -- and with only 147 explicit tracks in the test set, this analysis has limited power to detect anything but a fairly large gap.

---

*This report summarizes an analysis built entirely in Python (pandas, scipy, and scikit-learn). All figures above are interactive -- try zooming, panning, or hovering over them.*

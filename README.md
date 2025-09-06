# new_video_algo
New video recommend algo for things like YouTube 
Clearer explanation + concrete examples for the feedback

Nice ideas — these three knobs (branching, temperature, SVM/outlier handling) form a powerful trio for balancing relevance, serendipity, and explainability in a video search/recommendation platform. Below I’ll explain each, show simple equations/pseudocode, and give concrete numeric examples and UI/UX suggestions.


---

1) Branching — flip & mix multiple algorithms

What it is: instead of a single recommender pipeline, allow multiple recommendation “algorithms” (e.g., collaborative filtering, content-based, trending, personalization, editor picks) to run in parallel and branch the user session into different recommendation threads — then either flip between threads or mix them into the final ranking.

Why: prevents one-algo dominance, gives users a controlled way to see alternatives, and enables short, explainable exploratory paths.

Two patterns:

Flip per-slot: for each next video slot choose an algorithm according to branch probabilities.

Mix per-item: compute a blended score from several algorithms' scores and pick top-K.


Mix formula (per-video):

final_score(v) = Σ_i w_i * score_i(v)

where score_i(v) are normalized outputs from algorithm i, and w_i are branch weights.

Example (mix):

Algos: CF (collaborative filtering), Content-based (CB), Trending (TR).

Weights: w_CF=0.6, w_CB=0.3, w_TR=0.1.

Suppose video V has scores: CF=0.8, CB=0.5, TR=0.2 -> final_score = 0.60.8 + 0.30.5 + 0.1*0.2 = 0.48 + 0.15 + 0.02 = 0.65.


Example (flip per-slot):

Branch probabilities [CF:0.6, CB:0.3, TR:0.1]. For slot1 pick CF, slot2 pick CB, slot3 maybe CF again — yields a visible mixed session that users can follow, and you can show a tiny badge (“recommended by: Content-based”) to explain provenance.


Pseudocode (flip + mix hybrid):

# pick algorithm for each slot with probability p_branch, else do blended ranking
if rand() < p_branch_flip:
    algo = sample_branch(branch_probs)
    list = algo.recommend(seed, k)
else:
    # blended ranking
    candidates = union_of_topN_from_all_algos(seed)
    for v in candidates:
        final_score[v] = sum(w[i]*normalize(score[i][v]) for i in algos)
    list = top_k_by(final_score, k)

UI idea: show a small “branch map” for the session: CF → CB → TR nodes, or let users toggle “Explore alternative track” to spawn a new branch.


---

2) Temperature — control randomness vs. strict match

What it is: treat algorithm scores like logits and then sample with a temperature parameter to control randomness. Low temperature → near-deterministic (highest scores dominate). High temperature → more randomness / exploration.

Softmax sampling formula:
Given raw scores s_i, probabilities p_i ∝ exp(s_i / T) (then normalized). T = temperature.

Numeric examples (scores s = [3.0, 2.0, 1.0]):

T = 0.5 → probabilities ≈ [0.867, 0.117, 0.016] (very deterministic)

T = 1.0 → probabilities ≈ [0.665, 0.245, 0.090] (moderate)

T = 2.0 → probabilities ≈ [0.506, 0.307, 0.186] (more random)


(These are from p_i = exp(s_i/T) / Σ exp(s_j/T).)

How to use it in the platform:

Strict match (low T, e.g., 0.3–0.7): default for “For you” personalized feed.

Balanced (T≈1.0): hybrid lists where you want some novelty.

Exploration (T>1.5): “Discover” or “Surprise me” modes.


Variants:

Top-k sampling: sample only from top-k after softmax to avoid very low-quality items.

Top-p (nucleus) sampling: choose smallest set with cumulative p≥p_threshold then sample.


Pseudocode (temperature sampling):

probs = softmax(scores, T)
candidate = random_choice(population=candidates, weights=probs)

UI idea: expose a small “Exploration” slider (or toggle) that maps to T. Or use preset buttons: “Tight”, “Balanced”, “Surprising”.


---

3) SVM for “outside the bubble” detection, feature hashing, and naming outliers

Goal: detect results that lie outside the user’s usual preference bubble, surface them intentionally (for exploration) and automatically categorize/name them.

A) Define the bubble

Build a feature vector for the user (e.g., averaged video embeddings of watched history, topical TF-IDF, genre distribution).

The bubble is the region of feature space that contains the user’s historical interactions.


B) One-class SVM (or standard SVM boundary)

Train a one-class SVM on in-bubble vectors (user history) to estimate a decision boundary: points outside are outliers.

Parameter nu controls fraction of training samples considered outliers (e.g., nu=0.05 allows ~5% margin).

Kernel: linear for high-dimensional sparse hashed features, RBF if using dense embeddings.


Why SVM: it finds a boundary that generalizes beyond simple similarity thresholds and can pick complex shaped “bubbles.”

C) Feature hashing inside a bubble

When bubbles are specialized (lots of small categorical metadata), use the hashing trick to compress many categorical & textual features into a fixed-size vector (e.g., D = 2^14 = 16384).

Hashing keeps memory bounded and is fast for incremental updates.


Feature hashing example (toy):

Features: genre=comedy, tag=#standup, creator=Alice, transcript_word=“poker”.

Hash each token to an index and increment that index by TF or TF-IDF weight.


D) Categorize and name outliers

Workflow:

1. Detect outliers using SVM.


2. Collect outlier items for a session (or for all users).


3. Cluster outliers (k-means, HDBSCAN) to find emergent groups.


4. For each cluster compute top features / top n-grams / top tags / representative video → auto-generate a human-readable label, e.g. “Retro tech explainers”, “Action sports — niche creators”, “Political satire (regional)”.


5. Optionally present clusters to human editors for short labels (human-in-loop for sensitive categories).



Simple pseudocode for detection + naming:

# train on user's history vectors X_inbubble
ocsvm = OneClassSVM(nu=0.05, kernel='linear').fit(X_inbubble)

# for candidate video v with feature vector x_v
is_outlier = ocsvm.predict([x_v]) == -1

# collect outliers -> cluster -> extract top tokens
clusters = kmeans.fit(outlier_vectors)
for cluster in clusters:
    top_tokens = extract_top_k_features(cluster)
    cluster_name = auto_name(top_tokens)  # rule-based or LLM-assisted

Naming heuristics: use highest-avg-TF-IDF words, most common tags, or the most-viewed representative video title; run simple templates like “{dominant_tag} + {dominant_genre}”.

E) Presenting outliers in UI

Label: “Outside your bubble — try something different”.

Grouped view: “Exploration clusters” with short labels + one-sentence rationale: “Because you watched X, here are niche picks about Y”.

Let users give feedback (“More like this” / “Not for me”) to refine SVM boundary.



---

Combined flow example — real session

1. Seed: user watches a gardening how-to video.


2. Branching: platform spawns 3 branches:

CF branch (weights: 0.7)

CB branch (weights: 0.2)

Trending branch (0.1)



3. For each next slot: with probability p_flip=0.6 pick a branch; else create a blended mixed list among all branches.


4. Temperature: user set to “Balanced” (T=1.0) -> sampling allows occasional picks from the lower-ranked but still relevant items. If user toggles “Surprise me” (T=2.0) more diverse picks appear.


5. SVM check: every candidate is scored by the one-class SVM; items flagged as “outliers” are grouped and shown as a separate module “Explore outside your bubble” with cluster labels like “Urban farming documentaries — unusual creators”.


6. User feedback: user thumbs up one outlier cluster → system expands that area of the bubble and reduces SVM outlier penalty for similar items.




---

Implementation & metrics notes

Reproducibility: keep random seeds for deterministic testing; expose a “session seed” for debugging.

Offline metrics: precision@k, NDCG, diversity, novelty, serendipity. For SVM evaluate true-positive outlier detection rate on held-out labeled set (if available).

Online metrics: watch time, retention, CTR, fraction of outlier clicks, long-term retention lift.

Hyperparameters to tune: branch weights, p_flip, temperature T, SVM nu, hashing dimension D, cluster count for outliers.

Safety: screen outlier clusters for policy-sensitive categories; prefer human review or conservative thresholds before widely surfacing.



---

Quick cheatsheet (practical knobs + starting values)

Branch weights: start with CF 0.6 / CB 0.3 / TR 0.1.

Branch flip probability p_flip: 0.5–0.8 (higher = more visible separate tracks).

Temperature T: 0.5 (strict) — 1.0 (balanced) — 2.0 (explore).

One-class SVM nu: 0.02–0.10 (2–10% allowed margin).

Hash dimension D: 2^14 (16k) for medium vocab, scale up if collisions degrade quality.

Cluster algorithm for outliers: start with k-means where k ~ sqrt(n_outliers) or HDBSCAN for variable density.



---

If you want, I can:

produce a small reference implementation (Python pseudocode or runnable notebook) for mixing + temperature sampling + one-class-SVM with hashed features, or

sketch UI mockups for the branching map + “outside bubble” module.


Which would you like me to generate next?


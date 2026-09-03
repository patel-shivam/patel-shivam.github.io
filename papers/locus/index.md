---
layout: paper
permalink: /papers/locus/
title: "LOCUS: Low-Dimensional Model Embeddings for Efficient Model Exploration, Comparison, and Selection"
short_title: LOCUS
description: >-
  LOCUS turns a set of query evaluations into a single vector that summarizes a
  language model's capabilities. Embeddings are produced by a deterministic forward
  pass, so new models can be onboarded without retraining. AdaptFM @ ICML 2026.
image: /papers/locus/figures/train-sample-eff.png
venue: AdaptFM @ ICML 2026
authors:
  - name: Shivam Patel
    url: /
    affil: 1
    me: true
    corresponding: true
  - name: William Cocke
    affil: 1
  - name: Gauri Joshi
    url: https://www.andrew.cmu.edu/user/gaurij/
    affil: 1
affiliations:
  - name: Carnegie Mellon University
    url: https://www.cmu.edu
corresponding_note: Corresponding author
links:
  - name: arXiv
    url: https://arxiv.org/abs/2601.21082
  - name: PDF
    url: https://arxiv.org/pdf/2601.21082
  - name: Code
    url: https://github.com/patel-shivam/locus_code_release
tldr: >-
  There are now more open language models than anyone can evaluate, let alone compare.
  LOCUS represents each model as a single low-dimensional vector — a *model embedding* —
  learned from nothing but the queries the model was asked and the scores it received.
  An attention encoder reads the model's evaluation set as an unordered set and emits one
  vector; because this is a deterministic forward pass rather than a gradient-descent fit,
  a new model can be added to the pool without retraining anything and without disturbing
  the embeddings already in it. LOCUS reaches a given routing accuracy with up to
  $4.8\times$ fewer evaluations than learned-parameter baselines, and roughly 128
  evaluations suffice to place a previously unseen model. The resulting space is
  geometrically meaningful: distance predicts behavioural disagreement
  ($\rho = 0.89$), nearest neighbours make usable stand-ins when a model goes down
  (85% of routing accuracy retained), and a 15-model portfolio chosen purely from the
  geometry matches the routing accuracy of all 112 models.
bibtex: |
  @inproceedings{
    patel2026locus,
    title     = {{LOCUS}: Low-Dimensional Model Embeddings for Efficient
                 Model Exploration, Comparison, and Selection},
    author    = {Shivam Patel and William Cocke and Gauri Joshi},
    booktitle = {ICML 2026 Workshop on Adaptive Foundation Models},
    year      = {2026},
    url       = {https://arxiv.org/abs/2601.21082}
  }
---

<h2 class="section">A pool too large to reason about</h2>

More than 300,000 text-to-text models are published on HuggingFace, and the number grows
weekly. They differ in size, architecture, training data, and specialization, which is
precisely what makes the pool valuable — a 7B math model can beat a 70B generalist on
arithmetic at a tenth of the cost — and also what makes it unmanageable. Practical
questions pile up quickly. Which of these models are near-duplicates of each other? Which
handful should we actually deploy? If the model we wanted is down, what is the closest
substitute? When a new model appears, where does it fit?

Every one of these is a question about *comparison*, and none of them can be answered by
reading a leaderboard row. What they need is a representation in which models can be
placed side by side, measured against each other, and searched. LOCUS proposes the
simplest such representation: give each model a single fixed-length vector.

$$m \;\longmapsto\; z_m \in \mathbb{R}^{d}$$

The question is where that vector should come from. Model weights are the obvious source
and the least usable one: architectures are incommensurable and the strongest models are
behind APIs. Output logits require shared tokenization. What *is* universally available is
behaviour — ask a model a query, score its answer. So LOCUS builds the embedding from
evaluations alone. For a model $m$ evaluated on a set of queries $\mathcal{Q}\_m$, with each
query $x$ carrying a sentence encoding $\phi(x)\in\mathbb{R}^{d\_\phi}$ and a score
$y^{(m)}(x)\in[0,1]$ (usually just binary correctness), the input is the set

$$S_m \;=\; \big\{\,(\phi(x_i),\; y^{(m)}(x_i))\,\big\}_{x_i \in \mathcal{Q}_m}$$

and the output is one vector. Everything else on this page follows from how that map is
built.

<h2 class="section">What the map has to satisfy</h2>

The use cases above impose requirements that are easy to state and, taken together,
awkward to satisfy at once:

- **Black-box compatible** — only queries and scores; no weights, activations, or logits.
- **Tolerant of uneven evaluation** — models need not share a common query set.
- **Sample efficient** — informative from few evaluations, and improving as more arrive.
- **Training-free onboarding** — adding a model must not require training model-specific
  parameters, nor perturb the embeddings already computed.
- **Predictive** — the embedding must support accurate performance prediction on unseen
  queries.
- **Geometrically meaningful** — cosine or Euclidean proximity must track model similarity.

Prior work splits cleanly, and each half fails a different requirement. *Parametric*
methods — EmbedLLM, IRT-Net, JE-IRT — treat each model's embedding as a free parameter
trained jointly with a correctness predictor. They predict well, but the embedding is
whatever SGD happened to land on. *Nonparametric* methods such as LLM-DNA compute
embeddings by fixed geometric operations, which makes them deterministic but requires all
models to be evaluated on identical queries, offers no way to refine an embedding as more
evaluations arrive, and provides no correctness predictor.

<div class="table-wrap">
<table>
  <caption>Prior approaches satisfy one half of the requirements or the other. LOCUS is
  designed to sit in the intersection: embeddings come from a forward pass, but the
  encoder that produces them is trained, and it accepts evaluation sets of any size and
  composition.</caption>
  <thead>
    <tr>
      <th>Method</th>
      <th>Training-free embeddings</th>
      <th>Correctness prediction</th>
      <th>Varying eval queries</th>
    </tr>
  </thead>
  <tbody>
    <tr><td>EmbedLLM</td><td>✗</td><td>✓</td><td>✓</td></tr>
    <tr><td>IRT-Net</td><td>✗</td><td>✓</td><td>✓</td></tr>
    <tr><td>LLM-DNA</td><td>✓</td><td>✗</td><td>✗</td></tr>
    <tr><td class="ours"><b>LOCUS</b></td><td class="ours">✓</td><td class="ours">✓</td><td class="ours">✓</td></tr>
  </tbody>
</table>
</div>

<h3>Why a trained embedding is a fragile one</h3>

The training-free requirement deserves more than a checkmark, because it is the one that
quietly breaks the geometry. Consider what happens when a new model joins an
EmbedLLM-style pool. The correctness predictor $G_\psi$ is frozen, and the new model's
embedding is fit by gradient descent against it. Do this for models whose evaluation data
is *identical* to models already embedded — clones, in effect — and the regenerated
embeddings should land on top of the originals. They do not.

<figure>
  <img src="{{ '/papers/locus/figures/embedllm-drift.png' | relative_url }}" alt="Heatmap of cosine distance between original and regenerated EmbedLLM embeddings across decoder depths and embedding dimensions, showing large distances throughout">

  <figcaption>Average cosine distance between EmbedLLM embeddings and embeddings
  <i>regenerated from the same evaluation data</i> with the predictor held fixed, across
  decoder depths and embedding dimensions. The objective is nearly the same and the data
  is identical, yet the embeddings move substantially. Correctness predictions move with
  them: up to <b>8.1% disagreement</b> on a common test set between a model's original and
  regenerated embedding.</figcaption>
</figure>

An embedding that is not determined by its data cannot support distance-based reasoning.
Nearest-neighbour lookup, clustering, and similarity search all silently depend on the
assumption that two models with the same behaviour get the same vector. LOCUS enforces
that assumption by construction: the embedding is the output of a fixed function of the
evaluation set, so identical evaluations give identical embeddings, exactly and every time.

<h2 class="section">LOCUS: an attention encoder over evaluations</h2>

The evaluation set $S_m$ is a *set* — it has no natural order, and it has no fixed size.
That shape dictates the architecture. LOCUS uses a set encoder $F_\theta$ that maps $S_m$
to a vector, paired with a lightweight correctness predictor $G_\psi$ that maps an
embedding and a query back to a probability.

<figure>
  <img src="{{ '/papers/locus/figures/encoder-diagram.png' | relative_url }}" alt="Encoder pipeline: evaluations tokenized, passed through multi-head attention transformer layers, then a learned-query aggregation layer, producing a model embedding">

  <figcaption>The embedding generator $F_\theta$. Each (query encoding, score) pair becomes
  one token; bidirectional attention layers without positional encodings mix information
  across the whole evaluation set; a learned-query aggregation block collapses the tokens
  into a single vector. Only the shaded components are trained, and they are trained once
  for the whole pool — never per model.</figcaption>
</figure>

<h3>Step 1: tokenize each evaluation</h3>

A trained MLP $h_\omega$ turns each (encoding, score) pair into a $d$-dimensional token,
so that heterogeneous inputs land in one space that attention can operate on:

$$t_i^{(m)} \;=\; h_\omega\!\big(\phi(x_i),\; y^{(m)}(x_i)\big) \in \mathbb{R}^{1\times d}$$

Stacking these as rows gives $X_m^{(0)} \in \mathbb{R}^{n_m \times d}$, where
$n_m = |S_m|$ is however many evaluations happen to be available for this model. Nothing
downstream depends on $n_m$ being the same across models.

<h3>Step 2: attend, through a latent bottleneck</h3>

Plain self-attention over $n_m$ tokens costs $\mathcal{O}(n_m^2)$, which is punitive when a
model has thousands of evaluations. LOCUS routes attention through $r \ll n_m$ learned
latent vectors $U^{(\ell)} \in \mathbb{R}^{r \times d}$. Each layer *compresses* the
evaluations into the latents, then *broadcasts* the result back:

$$H_m^{(\ell)} = \mathrm{TBlock}\big(U^{(\ell)},\, X_m^{(\ell-1)},\, X_m^{(\ell-1)}\big) \in \mathbb{R}^{r\times d}$$

$$X_m^{(\ell)} = \mathrm{TBlock}\big(X_m^{(\ell-1)},\, H_m^{(\ell)},\, H_m^{(\ell)}\big) \in \mathbb{R}^{n_m\times d}$$

Cost drops to $\mathcal{O}(n_m r)$ — linear in the number of evaluations — while every
token still reaches every other token through the latents.

<h3>Step 3: aggregate with a learned query</h3>

A single trained query vector $s \in \mathbb{R}^{1\times d}$ attends over the final tokens
and returns the embedding:

$$z_m \;=\; \mathrm{TBlock}\big(s,\; X_m^{(L)},\; X_m^{(L)}\big) \in \mathbb{R}^{1\times d}$$

Because no stage uses positional encodings, and because aggregation is attention rather
than indexing, $z_m$ is **permutation invariant**: reordering the evaluations cannot change
the embedding. This is not an approximate property that training encourages — it is exact,
and the demonstration below verifies it numerically on the released checkpoint.

<h3>Step 4: predict correctness</h3>

An embedding is only actionable if it can be turned back into a decision. A two-layer MLP
$G_\psi$ consumes an embedding and a query encoding and returns a probability:

$$\widehat{p}_\psi\big(y^{(m)}(x)\!=\!1 \,\big|\, z_m, \phi(x)\big) \;=\; \sigma\big(G_\psi(z_m, \phi(x))\big)$$

<figure style="max-width: 21rem; margin-left: auto; margin-right: auto;">
  <img src="{{ '/papers/locus/figures/decoder-diagram.png' | relative_url }}" alt="The correctness predictor: a model embedding and a query embedding feed a small MLP that outputs correct or incorrect">

  <figcaption>The correctness predictor $G_\psi$. It is deliberately small — the
  representational work has already been done by the encoder, and keeping the predictor
  light is what makes scoring 4,096 queries against all 112 models a 20&nbsp;ms
  operation.</figcaption>
</figure>

Encoder and predictor are trained together by minimizing binary cross-entropy. Each step
samples a mini-batch of models; for each, an *encoder subset* $S_m^{\mathrm{enc}}$ builds
the embedding and an independently sampled *decoder batch* $S_m^{\mathrm{dec}}$ supplies
the prediction targets:

$$\min_{\omega,\theta,\psi}\; \mathbb{E}_{m}\, \mathbb{E}_{x \sim S_m}\; \mathrm{BCE}\big(\widehat{p}^{(m)}(x),\, y^{(m)}(x)\big)$$

Training the encoder on subsets of varying size is what makes it work at any $n_m$ later.

<div class="algo">
  <p class="algo-title">Onboarding a new model — the entire procedure</p>
  <ol>
    <li>Evaluate $m_{\text{new}}$ on any queries you have; score them. <span class="cmt">≈128 suffice</span></li>
    <li>$z_{m_\text{new}} \leftarrow F_\theta(S^{\text{enc}}_{m_\text{new}})$ <span class="cmt">one forward pass</span></li>
  </ol>
  <p class="algo-title" style="margin-top:1rem">Not required</p>
  <ol>
    <li>Training model-specific parameters. <span class="cmt">there are none</span></li>
    <li>Recomputing or perturbing any existing $z_m$. <span class="cmt">they are untouched</span></li>
  </ol>
</div>

{% include_relative encoder.part.html %}

<h2 class="section">Experimental setup</h2>

All experiments use the evaluation matrix released with EmbedLLM: **112 language models**
spanning base, chat, and finetuned specialists from roughly 1B to 72B parameters,
evaluated on queries drawn from **10 public benchmarks** — MathQA, LogiQA, MedMCQA, PIQA,
TruthfulQA, MMLU, GSM8K, GPQA, ASDiv, and SocialIQA. Queries are encoded with
`all-mpnet-base-v2` into $\mathbb{R}^{768}$; ablations over three other sentence encoders
appear in the appendix.

The encoder uses $L=2$ latent-bottleneck blocks with 4-head attention and $r=64$ latents,
followed by the aggregation block. Embedding dimension is $d=128$; the correctness
predictor is a two-layer MLP with hidden width 64. Baselines are **EmbedLLM** and
**IRT-Net**, both of which learn per-model embeddings by backpropagation. Test queries are
never used to build embeddings — only to evaluate them.

Two metrics recur throughout. **Correctness prediction accuracy** thresholds
$\widehat{p}^{(m)}(x)$ at $0.5$ and measures agreement with the observed label over all
model–query pairs. **Routing accuracy** asks a sharper question: send each query to the
model with the highest predicted correctness, and measure how often that model was in fact
correct.

<h2 class="section">Results</h2>

<h3>Routing and correctness prediction</h3>

<div class="table-wrap">
<table>
  <caption>Overall routing and correctness prediction accuracy (%) as the number of query
  evaluations per model varies. LOCUS leads on routing at every training-set size. On
  correctness prediction the three methods are close — averaging a per-pair accuracy over
  112 models washes out the differences that routing exposes, since routing depends on
  getting the <i>ranking</i> right at the top.</caption>
  <thead>
    <tr>
      <th rowspan="2">Approach</th>
      <th colspan="3">Routing accuracy (%)</th>
      <th colspan="3">Corr. prediction (%)</th>
    </tr>
    <tr>
      <th>256</th><th>512</th><th>1024</th>
      <th>256</th><th>512</th><th>1024</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td class="ours"><b>LOCUS</b></td>
      <td class="ours">61.90</td><td class="ours">62.97</td><td class="ours">64.70</td>
      <td class="ours">68.31</td><td class="ours">68.33</td><td class="ours">70.03</td>
    </tr>
    <tr>
      <td>EmbedLLM</td>
      <td>58.80</td><td>59.47</td><td>59.60</td>
      <td>67.33</td><td>68.12</td><td>69.47</td>
    </tr>
    <tr>
      <td>IRT-Net</td>
      <td>59.57</td><td>60.17</td><td>63.37</td>
      <td>67.38</td><td>69.07</td><td>70.12</td>
    </tr>
  </tbody>
</table>
</div>

<h3>Sample efficiency</h3>

The gap widens as evaluations become scarce, which is the regime that matters — every
evaluation is an inference call over the pool.

<figure>
  <img src="{{ '/papers/locus/figures/train-sample-eff.png' | relative_url }}" alt="Routing accuracy versus number of training samples: LOCUS above IRTNet above EmbedLLM, with 2.3x and 4.8x horizontal arrows">

  <figcaption>Routing accuracy against the number of evaluations per model used to train
  $(F_\theta, G_\psi)$. To reach the accuracy LOCUS achieves at roughly 450 evaluations,
  IRT-Net needs <b>2.3×</b> as many and EmbedLLM <b>4.8×</b> as many. Attention over the
  evaluation set extracts a capability signal that per-model gradient fitting needs far
  more data to recover.</figcaption>
</figure>

The same advantage appears at test time, where the question is how many evaluations a
*new* model needs before its embedding is usable.

<figure>
  <img src="{{ '/papers/locus/figures/onboarding.png' | relative_url }}" alt="Grid of correctness prediction accuracy for held-out models by number of evaluation queries and number of training models">

  <figcaption>Onboarding 16 held-out models never seen during training. Rows vary how many
  models the encoder was trained on; columns vary how many evaluations are used to embed
  the new model. Roughly <b>128 evaluations</b> are enough, and an encoder trained on a
  partial pool loses <b>less than 1%</b> against one trained on all 112 models — the
  encoder generalizes to models outside its training set.</figcaption>
</figure>

<h3>Robustness to which evaluations you happen to have</h3>

Two models are rarely evaluated on the same queries, so an embedding that shifts when the
evaluation set is resampled would be useless for comparison. The paper perturbs the input
two ways: changing *which* queries are used at fixed count (overlap fraction $\alpha$ with
a reference set), and changing *how many*.

<figure>
  <img src="{{ '/papers/locus/figures/robustness-perf.png' | relative_url }}" alt="Correctness prediction and routing accuracy against evaluation set size and overlap fraction, both flat after a small size">

  <figcaption>Correctness prediction (top) and routing accuracy (bottom) against evaluation
  set size (left) and overlap with a reference set (right). Performance saturates at
  <b>128–256 evaluations</b> and is essentially flat in overlap — an embedding built from a
  disjoint query sample is as good as one built from the reference sample.</figcaption>
</figure>

<figure class="fig-pair">
  <img src="{{ '/papers/locus/figures/robust-tsne-overlap.png' | relative_url }}" alt="t-SNE overlay of embeddings recomputed at varying overlap fractions, clustered near their reference points">
  <img src="{{ '/papers/locus/figures/robust-tsne-size.png' | relative_url }}" alt="t-SNE overlay of embeddings recomputed from subsampled evaluation sets, clustered near their reference points">

  <figcaption>The same stability in the geometry rather than the metrics. Embeddings for
  selected models recomputed under varying overlap (<b>left</b>) and subsampling
  (<b>right</b>), with the rest of the pool in grey. Recomputed embeddings stay in their
  own neighbourhood rather than wandering into other models' territory — the contrast with
  the EmbedLLM regeneration heatmap above is the point.</figcaption>
</figure>

<h3>Distance means what it should</h3>

The claim that makes every downstream application possible is that embedding distance
tracks behavioural difference. Measure it directly: for each pair of models compute their
embedding distance, and separately their **correctness disagreement rate** — the fraction
of test queries on which their binary correctness labels differ.

<div class="table-wrap">
<table>
  <caption>Correlation between embedding distance and correctness disagreement across all
  6,216 model pairs. The relationship is strong under every measure and both metrics,
  including the rank correlations that matter for nearest-neighbour retrieval.</caption>
  <thead>
    <tr><th>Correlation</th><th>Cosine distance</th><th>Euclidean distance</th></tr>
  </thead>
  <tbody>
    <tr><td>Pearson $\rho$</td><td>0.845</td><td class="ours">0.887</td></tr>
    <tr><td>Spearman $r_s$</td><td class="ours">0.886</td><td>0.876</td></tr>
    <tr><td>Kendall $\tau$</td><td class="ours">0.714</td><td>0.702</td></tr>
  </tbody>
</table>
</div>

<figure>
  <img src="{{ '/papers/locus/figures/distance-vs-disagreement.png' | relative_url }}" alt="Two scatter plots of embedding distance against correctness disagreement, both showing tight positive relationships">

  <figcaption>Every pair of models, embedding distance against correctness disagreement,
  under cosine (left) and Euclidean (right) distance. Knowing only where two models sit in
  a 128-dimensional space tells you, to a good approximation, how often they will disagree
  on a query neither has been evaluated on.</figcaption>
</figure>

Grouping models by this geometry recovers structure nobody supplied. Hierarchical
clustering on pairwise embedding distances separates the math-finetuned and code-finetuned
families without ever being told which models those are.

<figure class="fig-pair">
  <img src="{{ '/papers/locus/figures/dendrogram.png' | relative_url }}" alt="Dendrogram of models clustered by embedding distance with math and code families highlighted">
  <img src="{{ '/papers/locus/figures/family-heatmap.png' | relative_url }}" alt="Pairwise distance heatmap of models showing block structure aligned with families">

  <figcaption>Hierarchical clustering of model embeddings. Math models (orange) and code
  models (green) fall into coherent blocks, and the distance heatmap shows the same
  structure. Specialization is a geometric property of the embedding space, not an
  annotation.</figcaption>
</figure>

<h2 class="section">The embedding space, up close</h2>

The figures above summarize the geometry; the cloud below *is* the geometry. It holds all
112 model embeddings from the released checkpoint — drag to rotate it, scroll or use the
buttons to zoom — with the real nearest neighbours, real per-benchmark accuracy profiles,
and real correctness agreement attached to each point. Three dimensions are worth the
trouble here mostly because you can turn the cloud: any single flat projection puts
unrelated models on top of each other, and rotating separates them. The two modes
correspond to the two applications that follow.

{% include_relative atlas.part.html %}

<h2 class="section">What the geometry is good for</h2>

<h3>Nearest neighbours as stand-ins</h3>

A router picks a model; the model is overloaded, or down, or disallowed by policy. Without
per-query suitability scores there is no principled way to choose a replacement — unless
models live in a space where "similar" has a meaning. Ranking each model's neighbours by
embedding distance and measuring correctness agreement on common queries shows the
substitution is sound.

<div class="stats">
  <div><span class="n">79%</span><span class="l">Correctness agreement with closest neighbour, averaged over all 112 models</span></div>
  <div><span class="n">85%</span><span class="l">Routing accuracy retained when every routed model is replaced by its nearest neighbour</span></div>
  <div><span class="n">77.7%</span><span class="l">Agreement with the <i>fifth</i> neighbour — the decay with rank is gradual, not a cliff</span></div>
</div>

<figure class="fig-pair">
  <img src="{{ '/papers/locus/figures/knn-agreement.png' | relative_url }}" alt="Correctness agreement decaying gradually with neighbour rank k">
  <img src="{{ '/papers/locus/figures/fallback-routing.png' | relative_url }}" alt="Routing accuracy under fallback to the kth nearest model, decaying gradually">

  <figcaption><b>Left:</b> average correctness agreement between a model and its $k$-th
  nearest neighbour in embedding space. <b>Right:</b> routing accuracy when the selected
  model is unavailable and the query falls back to its $k$-th neighbour. Both decay
  smoothly, which is what makes the fallback usable: the system degrades rather than
  breaking, and a second or third choice is nearly as good as the first.</figcaption>
</figure>

<h3>Choosing a portfolio</h3>

Serving 112 models is not a plan. The practical question is which small subset to deploy,
and the embedding space answers it without evaluating candidate subsets: pick models that
*cover* the space, on the argument that coverage in a geometry where distance means
behavioural difference is coverage of capability.

Two classical objectives apply directly. `k-center` minimizes the largest distance from any
model to the selected set; `k-medoids` minimizes the average. Under a parameter budget
rather than a count, a `coverage-greedy` rule takes the model with the best marginal
coverage gain *per parameter*, which naturally prefers small specialists over large
generalists.

<div class="stats">
  <div><span class="n">15</span><span class="l">Models chosen by <code>k-center</code> that match the routing accuracy of all 112</span></div>
  <div><span class="n">~150B</span><span class="l">Parameter budget reaching near-full accuracy, against 1,930B for the pool</span></div>
  <div><span class="n">8%</span><span class="l">Of the pool's total parameters</span></div>
  <div><span class="n">0</span><span class="l">Query evaluations needed to make the selection — geometry only</span></div>
</div>

<figure>
  <img src="{{ '/papers/locus/figures/portfolio.png' | relative_url }}" alt="Left: routing accuracy versus number of selected models for k-center, k-medoids and random. Right: routing accuracy versus parameter budget for coverage-greedy and random">

  <figcaption><b>Left:</b> routing accuracy against portfolio size. Both coverage
  objectives clear the random baseline's band decisively, and <code>k-center</code>
  saturates at $k \approx 15$. <b>Right:</b> under a total-parameter budget,
  <code>coverage-greedy</code> reaches near-full accuracy at a small fraction of the
  pool's 1,930B parameters. Selection uses only $\{z_m\}$ — no routing evaluation of
  candidate subsets is required.</figcaption>
</figure>

<h3>Searching for a capability profile you do not have yet</h3>

Because the encoder maps *any* evaluation set to a vector, it also maps *hypothetical*
ones. Write down a desired per-task accuracy profile, synthesize an evaluation set whose
task-wise rates match it, push it through $F_\theta$, and search the library of real
embeddings for the nearest match. This retrieves models by the capability profile you want
rather than by name or benchmark rank.

<figure>
  <img src="{{ '/papers/locus/figures/hypothetical-recall.png' | relative_url }}" alt="Recall at k for retrieving the intended model from a hypothetical embedding, rising with evaluation set size">

  <figcaption>Recall@$k$ for recovering the intended model from a synthetic profile.
  With 8,192 synthetic queries, <b>recall@10 reaches ≈97%</b>. Correctness labels are
  randomized at the query level subject only to matching task-level rates, so the encoder
  is clearly reading task-level behaviour rather than memorizing which queries a model
  answered.</figcaption>
</figure>

<h3>Spotting duplicates</h3>

Determinism plus stability under resampling gives a fingerprinting signal for free: if two
endpoints repeatedly embed to the same point across independently sampled evaluation sets,
they are plausibly the same model. Recomputing the embeddings for this page surfaces the
effect immediately — the closest pairs in the pool are at cosine distances of
$10^{-5}$ and below, and they are exactly the pairs you would expect.

<div class="table-wrap">
<table>
  <caption>Closest model pairs in the recomputed embedding space. Starling-LM-7B-alpha is
  finetuned from openchat-3.5; the remaining pairs are merge-and-finetune derivatives
  drawn from the same base families. Nothing about lineage was supplied to the encoder —
  only queries and binary scores — so the near-coincidence is inferred entirely from
  behaviour.</caption>
  <thead>
    <tr><th>Model pair</th><th>Cosine distance</th></tr>
  </thead>
  <tbody>
    <tr><td>ConvexAI/Luminex-34B-v0.2 &nbsp;·&nbsp; fblgit/UNA-SimpleSmaug-34b-v1beta</td><td>&lt; 10<sup>−5</sup></td></tr>
    <tr><td>rishiraj/CatPPT-base &nbsp;·&nbsp; bardsai/jaskier-7b-dpo-v5.6</td><td>&lt; 10<sup>−5</sup></td></tr>
    <tr><td>berkeley-nest/Starling-LM-7B-alpha &nbsp;·&nbsp; openchat/openchat_3.5</td><td>2 × 10<sup>−5</sup></td></tr>
    <tr><td>CultriX/NeuralTrix-bf16 &nbsp;·&nbsp; openchat/openchat_3.5</td><td>2 × 10<sup>−5</sup></td></tr>
  </tbody>
</table>
</div>

<h2 class="section">What it costs</h2>

The encoder's linear scaling is not theoretical. Embedding the entire 112-model pool from
4,096 evaluations each takes about **100 ms** on one V100, and scoring 4,096 unseen queries
against all 112 models takes about **20 ms** — two to three orders of magnitude below the
seconds-scale latency of the generation these predictions are used to schedule.

<figure class="fig-pair">
  <img src="{{ '/papers/locus/figures/timing-encoder.png' | relative_url }}" alt="Encoder wall clock time growing linearly with number of evaluation queries">
  <img src="{{ '/papers/locus/figures/timing-decoder.png' | relative_url }}" alt="Decoder wall clock time for correctness prediction across query batch sizes">

  <figcaption><b>Left:</b> embedding generation time against evaluation set size, for
  several pool sizes — linear, as the latent bottleneck predicts. <b>Right:</b> correctness
  prediction time against the number of unseen queries. Routing overhead is negligible next
  to generation.</figcaption>
</figure>

<h2 class="section">Summary</h2>

LOCUS represents a language model as a single low-dimensional vector computed from its
query evaluations by a trained attention encoder. Making the embedding a deterministic
forward pass rather than a set of fitted parameters is what buys the properties that
matter downstream: new models onboard without retraining and without disturbing existing
embeddings, embeddings refine as evaluations accumulate, and identical behaviour yields
identical vectors — so distance can be trusted.

Empirically the encoder is markedly more sample efficient than learned-parameter baselines
($4.8\times$ against EmbedLLM), needs roughly 128 evaluations to place an unseen model, and
produces a space whose geometry carries real information: distance predicts behavioural
disagreement, neighbours serve as fallbacks retaining 85% of routing accuracy, hierarchical
clustering recovers model families, and coverage in the space picks a 15-model portfolio
that matches a 112-model pool. Extensions to multimodal models, and to adaptively choosing
*which* queries to evaluate a new model on, are the natural next steps.

---
layout: paper
permalink: /papers/proxrouter/
title: "ProxRouter: Proximity-Weighted LLM Query Routing for Improved Robustness to Outliers"
short_title: ProxRouter
description: >-
  ProxRouter improves the robustness of training-free LLM query routers to outlier
  queries through proximity-weighted aggregation over the reference set, replacing
  hard nearest-cluster assignment. AISTATS 2026.
image: /papers/proxrouter/figures/headline-pareto.png
venue: AISTATS 2026
authors:
  - name: Shivam Patel
    url: /
    affil: 1
    me: true
    corresponding: true
  - name: Neharika Jali
    url: https://sites.google.com/view/neharika-homepage/home
    affil: 1
  - name: Ankur Mallick
    url: https://www.microsoft.com/en-us/research/people/ankurmallick/
    affil: 2
  - name: Gauri Joshi
    url: https://www.andrew.cmu.edu/user/gaurij/
    affil: 1
affiliations:
  - name: Carnegie Mellon University
    url: https://www.cmu.edu
  - name: Microsoft
    url: https://www.microsoft.com/en-us/research/
corresponding_note: Corresponding author
links:
  - name: arXiv
    url: https://arxiv.org/abs/2510.09852
  - name: PDF
    url: https://arxiv.org/pdf/2510.09852
  - name: OpenReview
    url: https://openreview.net/forum?id=GYWls3tyqM
tldr: >-
  LLM query routers assign each inference query to a model that balances predictive
  quality and inference cost. Nonparametric routers make this decision without
  training an additional predictor, instead comparing each new query with reference
  queries in the embedding space. Although effective for queries represented by the
  reference set, their performance can degrade under distribution shift. ProxRouter
  replaces hard assignment with a weighted aggregation over the reference set: the
  weights begin from a minimum-variance prior and are exponentially tilted toward
  reference points close to the query. A single parameter τ controls the resulting
  bias-variance tradeoff, with standard $K$Means and $k$NN routing arising as
  limiting cases. On outlier tasks, ProxRouter improves normalized AUC by 8.1 points
  while preserving inlier performance, with only a few milliseconds of routing
  overhead.
bibtex: |
  @inproceedings{
    patel2026proxrouter,
    title     = {ProxRouter: Proximity-Weighted {LLM} Query Routing
                 for Improved Robustness to Outliers},
    author    = {Shivam Patel and Neharika Jali and
                 Ankur Mallick and Gauri Joshi},
    booktitle = {The 29th International Conference on Artificial
                 Intelligence and Statistics},
    year      = {2026},
    url       = {https://openreview.net/forum?id=GYWls3tyqM}
  }
---

<h2 class="section">Routing queries to a model</h2>

State-of-the-art language models can contain billions or even trillions of parameters,
making inference computationally expensive. Yet invoking a frontier-scale model for
every query is often unnecessary: smaller, less expensive models can match the
quality of larger models on many inputs, while specialized models may outperform
general-purpose LLMs on particular domains at substantially lower cost. Query routing
exploits this heterogeneity by selecting, for each incoming query, the model that best
balances response quality and inference cost.

The plot below summarizes the model pool used in our experiments: 14 models evaluated
across 10 datasets. Models are ordered by average accuracy and tasks by difficulty,
producing an overall increase from left to right. Green denotes higher accuracy, red
denotes lower accuracy, and white corresponds to 50%.

{% include_relative models3d.part.html %}

Llama-3.3-70B is the strongest model overall, with 78.4% average accuracy, yet
Deepseek-math-7B, with less than one-tenth as many parameters, achieves 77.1% on
GSM8k compared with 40.8% for Llama-3.3-70B. Conversely, Deepseek-math-7B achieves
24.2% on MedQA, where Llama-3.3-70B reaches 84.7%. Because no single model dominates
across all tasks, an effective router can improve both cost and accuracy relative to
always invoking the largest model.

<h2 class="section">What the router has to estimate</h2>

The routing objective is to maximize accuracy subject to an inference-cost constraint.
We use a Lagrangian relaxation, yielding the following unconstrained objective for each
query:

$$m^{*}(\mathbf{x}) \;\leftarrow\; \arg\max_{m \in \mathcal{M}}\;\big[\,\text{acc}^{(m)}(\mathbf{x}) - \lambda \cdot \text{cost}^{(m)}(\mathbf{x})\,\big]$$

Here $\mathbf{x}$ denotes the query embedding, while $\lambda$ controls the relative
importance of accuracy and cost. Sweeping $\lambda$ traces the accuracy-cost trade-off
reported throughout the experiments below.

Neither accuracy nor cost is known before inference, and both can vary across sampled
responses: repeated generations for the same query may differ in correctness and
output length. We write $U^{(m)}(\mathbf{x}) = \overline{U}^{(m)}(\mathbf{x}) + \epsilon^{(m)}(\mathbf{x})$,
where $\overline{U}$ is the expected objective value and $\epsilon$ denotes zero-mean
noise. Since this stochastic component is difficult to estimate directly, routing
reduces to estimating $\overline{U}^{(m)}(\mathbf{x})$ for each model:

$$m^{*}(\mathbf{x}) \;\leftarrow\; \arg\max_{m\in\mathcal{M}} \widehat{U}^{(m)}(\mathbf{x})$$

A distribution mismatch between the reference queries and test queries can therefore
degrade estimates of $\widehat{U}^{(m)}(\mathbf{x})$ and, consequently, lead to
suboptimal routing decisions.

<h3>Why nonparametric routers</h3>

Parametric routers train an auxiliary model to predict quantities such as correctness
or inference cost. Nonparametric routers instead operate directly in the query
embedding space and require no additional predictor training. Two common formulations
are clustering-based approaches such as $K$Means and nearest-neighbor methods such as
$k$NN.

Despite their simplicity, nonparametric routers can be competitive with parametric
alternatives. A key reason is that modern encoder representations often organize
queries into semantically meaningful regions, making local information in the
embedding space informative of model performance:

<figure>

  <div class="fig-row">

    <img style="flex:1.271" src="{{ '/papers/proxrouter/figures/tsne-task.png' | relative_url }}" alt="t-SNE of query encodings coloured by source task, showing compact task-specific regions">

    <img style="flex:1.000" src="{{ '/papers/proxrouter/figures/tsne-cluster.png' | relative_url }}" alt="The same encodings coloured by KMeans cluster assignment, closely tracking the task structure">

  </div>

  <figcaption>High-dimensional query encodings projected with t-SNE. <b>Left:</b>

  colored by task. <b>Right:</b> colored by cluster assignment from $K$Means with
  $K=16$. Queries from the same task occupy compact, localized neighborhoods in
  embedding space; consequently, clustering recovers semantically coherent regions

  that align closely with query types.</figcaption>

</figure>

Clustering and nearest-neighbor methods can also incorporate new queries and models
without retraining an auxiliary predictor, providing a practical advantage over
parametric designs. We therefore focus on the nonparametric routing setting.

<h2 class="section">Where nonparametric routers fail</h2>

Accurate nonparametric estimates benefit from large and diverse reference sets in which
queries have been evaluated on all, or most, models in the pool. Constructing such a
set is expensive because it requires repeated inference across the model pool, so in
practice routers are often built from a limited collection of downstream tasks.
As new applications introduce previously unseen query types, this limited coverage can
substantially degrade routing performance. Even seemingly modest distribution shifts,
such as appending Chain-of-Thought instructions, can change both model accuracy and
generation cost. Updating the router with such outlier queries requires evaluating them
across the model pool, making frequent retraining expensive.

<figure>

  <img src="{{ '/papers/proxrouter/figures/headline-pareto.png' | relative_url }}" alt="Accuracy versus cost curves: AllSee highest, ProxRouter in between, Base lowest">

  <figcaption>The <i>Base</i> router, a nearest-neighbor router trained only on

  inlier queries, generalizes poorly to outlier tasks at test time, exhibiting up to a

  <b>15% reduction in average accuracy</b> at a fixed cost relative to the <i>AllSee</i>

  router trained on both inlier and outlier queries. ProxRouter is trained on the same
  inlier-only data as Base, yet substantially improves the outlier accuracy-cost

  trade-off.</figcaption>

</figure>

One approach is to explicitly detect outliers and route them using a separate mechanism.
This introduces an additional prediction stage, however, whose errors and latency affect
every query. ProxRouter instead applies the same soft aggregation rule to both inlier
and outlier queries, eliminating the need for a separate outlier detector.

<h2 class="section">A unified view of nonparametric routers</h2>

Although $K$Means and $k$NN routers are usually presented as distinct algorithms, they
can be expressed as instances of the same weighted estimator. Both operate on a
**reference set** $\mathcal{I}$. Each element $i$ is associated with a reference
vector $\mathbf{r}_i$ in the embedding space and an objective value $V_i^{(m)}$ for each
model $m$.

For clustering routers, the reference elements are clusters: $\mathbf{r}_i$ is the
centroid of cluster $c_i$, and $V_i^{(m)}$ is the average objective value across its
training queries. For nearest-neighbor routers, each training query forms a reference
element, with $\mathbf{r}_i$ equal to its embedding and $V_i^{(m)}$ equal to its observed
objective value.

In both cases the estimate is a weighted average:

$$\widehat{U}^{(m)}(\mathbf{x}) \;=\; \sum_{i \in [\,|\mathcal{I}|\,]} w_i(\mathbf{x})\, V_i^{(m)}, \qquad w_i \ge 0,\;\; \sum_i w_i = 1$$

Standard nonparametric routers correspond to particular choices of these weights. The
$K$Means router assigns $w_i = 1$ to the cluster nearest to $\mathbf{x}$ and $0$ to all
others. The $k$NN router assigns $w_i = 1/k$ to each of the $k$ nearest reference points
and $0$ elsewhere.

Both weighting schemes discard potentially useful information. In $K$Means routing,
hard nearest-cluster assignment creates discontinuous decision boundaries: small
changes in a query can abruptly change the estimated objectives, particularly for
outliers far from all centroids. The assignment also ignores intra-cluster dispersion,
which is informative about the variance of $V_i^{(m)}$. In $k$NN routing, uniform
weighting treats all selected neighbors equally, even when an outlier query is
substantially closer to some neighbors than to others.

<h2 class="section">ProxRouter</h2>

ProxRouter replaces these fixed weighting rules with a query-dependent aggregation that
is applied uniformly to inlier and outlier queries. It begins with minimum-variance
priors $\mathbf{p}(\mathbf{x})$ over the reference set and then exponentially tilts them
according to each reference element's proximity to the query, producing
bias-controlled weights $\mathbf{w}(\mathbf{x})$.

<h3>Step 1: minimum-variance priors</h3>

Reference elements can have different levels of uncertainty, quantified by
$\text{Var}[V_i^{(m)}]$. The classical minimum-variance weighting is therefore
$p_i(\mathbf{x}) \propto (\text{Var}[V_i^{(m)}])^{-1}$. For a cluster, this variance
is determined by the variances of its $n_i$ constituent queries:

$$\text{Var}\big[V_i^{(m)}\big] \;=\; \frac{1}{n_i^{2}}\sum_{\mathbf{x}_\text{tr}\in c_i} \text{Var}\big[\epsilon^{(m)}(\mathbf{x}_\text{tr})\big]$$

The exact per-query variance is unavailable in practice, so we approximate it using the
geometry of the embedding space. More dispersed clusters tend to contain more
semantically heterogeneous queries and are therefore treated as less reliable than
compact clusters. Letting $s_i$ denote the mean distance from queries in a cluster to
its centroid, we model variance as increasing with $s_i$ and decreasing with $n_i$,
which yields the prior

$$p_i(\mathbf{x}) \;\propto\; n_i / s_i$$

For $k$NN routers, reliable per-query variance estimates are not readily available, so
we use the uniform prior $p_i(\mathbf{x}) = 1/k$ over the $k$ nearest reference elements.

<h3>Step 2: proximity-based tilting</h3>

The minimum-variance estimator can introduce substantial bias because it does not account
for the proximity of each reference element to the test query. To reduce this bias, we
reweight the priors using a proximity penalty $\phi_i(\mathbf{x})$ that increases with
the distance $d(\mathbf{x}, \mathbf{r}_i)$:

$$w_i(\mathbf{x}) \;\propto\; p_i(\mathbf{x})\, \exp\!\big(-\phi_i(\mathbf{x})/\tau\big)$$

These weights arise as the solution to a convex optimization problem that explicitly
balances proximity-induced bias and estimator variance. The first term favors nearby
reference elements, while the second regularizes the weights toward the low-variance
prior:

$$\min_{\mathbf{w}\in\Delta^{|\mathcal{I}|}}\;\; \sum_i w_i(\mathbf{x})\,\phi_i(\mathbf{x}) \;+\; \tau\,D_{\text{KL}}\big(\mathbf{w}(\mathbf{x})\,\|\,\mathbf{p}(\mathbf{x})\big)$$

Thus, $\tau$ provides a single parameter controlling the bias-variance trade-off.
Increasing $\tau$ moves the weights toward the low-variance prior, whereas decreasing
it places greater emphasis on proximity. The standard routers correspond to the two
extremes: $\tau = 0$ gives the $K$Means router with closest-cluster assignment, while
$\tau \to \infty$ with uniform priors gives the $k$NN router with uniform averaging.

<div class="algo">

  <p class="algo-title">Algorithm — ProxRouter</p>

  <ol>

    <li>$p_i(\mathbf{x}) \propto \big[\text{Var}(V_i^{(m)})\big]^{-1}$ <span class="cmt">least-variance priors</span></li>

    <li>$w_i(\mathbf{x}) \propto p_i(\mathbf{x})\exp(-\phi_i(\mathbf{x})/\tau)$ <span class="cmt">proximity tilting</span></li>

    <li>$\widehat{U}^{(m)}(\mathbf{x}) \leftarrow \sum_i w_i(\mathbf{x})\,V_i^{(m)}$ <span class="cmt">estimated objective</span></li>

    <li>$m^{\star}(\mathbf{x}) \leftarrow \arg\max_{m\in\mathcal{M}} \widehat{U}^{(m)}(\mathbf{x})$ <span class="cmt">chosen model</span></li>

  </ol>

</div>

<h3>How the weights behave</h3>

The interactive example below illustrates four clusters of training queries in a stylized
embedding space. Drag the query marker, or use the preset buttons, to compare how the
base router and ProxRouter distribute aggregation weight. Because the clusters differ
in both size and dispersion, the minimum-variance prior $n_i/s_i$ is non-uniform.

{% include_relative demo.part.html %}

For a query well inside a cluster, the two routers behave similarly, corresponding to
the typical inlier setting. Their behavior differs more substantially between clusters.
The base router assigns all weight to cluster B because its centroid is marginally
closest. ProxRouter instead distributes mass across multiple clusters and assigns more
weight to C than to B because C contains more samples and has lower dispersion, making
its summary $V_i^{(m)}$ less noisy under the variance model. Near a cluster boundary,
the base router can change assignments abruptly under small query perturbations, whereas
ProxRouter's weights vary smoothly.

The slider controls $\tau$. At the far right, ProxRouter reduces to the base router.
At the far left, the weights approach the prior and become independent of the test
query's location.

<figure>

  <img src="{{ '/papers/proxrouter/figures/bias-variance.png' | relative_url }}" alt="Normalized AUC peaks at an intermediate value of one over tau">

  <figcaption>Bias-variance tradeoff governed by proximity-based prioritization.

  Increasing $1/\tau$ strengthens proximity weighting and increases variance, while
  decreasing it increases bias. Routing performance is maximized at an intermediate

  value. All experiments below use $1/\tau = 20$, selected on held-out data.</figcaption>

</figure>

ProxRouter preserves the underlying clustering or nearest-neighbor routing structure.
The only additional computation is the evaluation of the priors followed by the
proximity-based reweighting.

<h2 class="section">Experimental setup</h2>

We evaluate ProxRouter on 10 publicly available query datasets using a pool of 14 LLMs
spanning a broad range of parameter counts and capabilities. Queries are embedded with
the MPNet-base sentence encoder, and proximity is measured using cosine distance.
We consider two forms of under-representation in the router training set:

- **Leave-Task-Out**, where the training set contains no queries from the outlier

  tasks. This creates a natural stress test for clustering routers, since no cluster
  is formed from the outlier distribution. We use this setting to evaluate $K$M-Prox.

- **Few-Shot-Outlier**, where only a small number of queries from the outlier tasks

  are available in training (about 25). For nearest-neighbor routers, uniform averaging
  can dilute the contribution of these few relevant examples among a larger set of
  neighbors. We use this setting to evaluate $k$NN-Prox.

We compare three routers. **Base** denotes the standard nonparametric router.
**Prox** applies ProxRouter using exactly the same training set. **AllSee** is
trained with access to both inlier and outlier tasks and therefore serves as a
full-information reference for routing performance.

Performance is summarized by the area under the mean accuracy-cost curve, normalized by
the cost range and reported as a percentage ($\text{AUC}_n$). Higher values correspond
to a better accuracy-cost trade-off: greater accuracy at a fixed cost, or lower cost at
a fixed accuracy.

<h2 class="section">Results</h2>

<h3>Clustering routers</h3>

For $K$M-Base, $K$M-Prox, and $K$M-AllSee, we use $K = 32$, cosine distance as the
proximity penalty, and $1/\tau = 20$.

<div class="table-wrap">

<table>

  <caption>Performance ($\text{AUC}_n$) of $K$M-Prox against $K$M-Base for two sets

  of outlier tasks, with $K$M-AllSee as the upper bound.</caption>

  <thead>

    <tr><th>Outlier tasks</th><th>Split</th><th>$K$M-Base</th><th class="ours">$K$M-Prox</th><th>$K$M-AllSee</th></tr>

  </thead>

  <tbody>

    <tr><td rowspan="3"><b>HellaSwag,<br>MedQA</b></td><td class="rowlab">Outlier</td><td>70.68%</td><td class="ours">74.88% <span class="delta up">+4.20</span></td><td>78.36%</td></tr>

    <tr><td class="rowlab">Inlier</td><td>74.62%</td><td class="ours">74.86%</td><td>74.63%</td></tr>

    <tr><td class="rowlab">Overall</td><td>73.04%</td><td class="ours">75.12%</td><td>74.87%</td></tr>

    <tr><td rowspan="3"><b>LogiQA, CSQA,<br>BBH</b></td><td class="rowlab">Outlier</td><td>63.39%</td><td class="ours">66.18% <span class="delta up">+2.79</span></td><td>67.25%</td></tr>

    <tr><td class="rowlab">Inlier</td><td>79.35%</td><td class="ours">79.92%</td><td>79.74%</td></tr>

    <tr><td class="rowlab">Overall</td><td>71.61%</td><td class="ours">73.46%</td><td>73.88%</td></tr>

  </tbody>

</table>

</div>

For both outlier sets, $K$M-Prox improves normalized AUC relative to $K$M-Base while
using the same training set and incurring negligible additional routing cost. It also
closes a substantial portion of the gap to $K$M-AllSee. Performance on inlier tasks is
nearly unchanged across the three routers, indicating that the improvement on unseen
tasks does not come at the expense of inlier routing quality.

<figure class="fig-pair">

  <img src="{{ '/papers/proxrouter/figures/km-ood-medqa.png' | relative_url }}" alt="Accuracy versus cost on MedQA and HellaSwag outliers">

  <img src="{{ '/papers/proxrouter/figures/km-ood-logiqa.png' | relative_url }}" alt="Accuracy versus cost on LogiQA, CommonsenseQA and BBH outliers">

  <figcaption>Router performance with <b>left:</b> MedQA and HellaSwag as outlier

  tasks, and <b>right:</b> LogiQA, CommonsenseQA and BBH-BoolEx as outlier tasks.
  $K$M-Prox achieves higher accuracy across nearly the entire operating-cost range,

  indicating an improvement in the Pareto trade-off rather than only in aggregate AUC.</figcaption>

</figure>

<h3>Nearest-neighbor routers</h3>

For $k$NN-Base, $k$NN-Prox, and $k$NN-AllSee, we use $k = 100$ and $1/\tau = 20$.
The training set contains approximately 25 queries from the math tasks GSM8k and SVAMP.

<div class="stats">

  <div><span class="n">+8.1 pp</span><span class="l">Outlier $\text{AUC}_n$, 38.5% to 46.6%</span></div>

  <div><span class="n">+4.1 pp</span><span class="l">Overall $\text{AUC}_n$, 64.0% to 68.1%</span></div>

  <div><span class="n">−0.6 pp</span><span class="l">Change in inlier $\text{AUC}_n$</span></div>

  <div><span class="n">~1%</span><span class="l">Routing overhead vs. generation latency</span></div>

</div>

<div class="table-wrap">

<table>

  <caption>Performance ($\text{AUC}_n$) of $k$NN-Prox against $k$NN-Base with GSM8k

  and SVAMP as few-shot outlier tasks.</caption>

  <thead>

    <tr><th>Outlier tasks</th><th>Split</th><th>$k$NN-Base</th><th class="ours">$k$NN-Prox</th><th>$k$NN-AllSee</th></tr>

  </thead>

  <tbody>

    <tr><td rowspan="3"><b>GSM8k,<br>SVAMP</b></td><td class="rowlab">Outlier</td><td>38.55%</td><td class="ours">46.64% <span class="delta up">+8.09</span></td><td>60.77%</td></tr>

    <tr><td class="rowlab">Inlier</td><td>77.51%</td><td class="ours">76.96%</td><td>79.11%</td></tr>

    <tr><td class="rowlab">Overall</td><td>63.98%</td><td class="ours">68.12%</td><td>74.60%</td></tr>

  </tbody>

</table>

</div>

ProxRouter increases outlier $\text{AUC}_n$ by 8.1 points, from 38.5% to 46.6%.
$k$NN-Prox also attains higher accuracy at substantially lower average cost than
$k$NN-Base. The model pool contains specialized models that can outperform the largest
general-purpose model on particular domains: on GSM8k, Deepseek-math-7B exceeds
Llama-3.3-70B by 36 percentage points. Uniform averaging across 100 neighbors dilutes
the influence of the few math examples available in the training set, causing
$k$NN-Base to favor the more expensive general model. Proximity weighting increases
the contribution of these relevant neighbors, enabling selection of the specialist.

<figure class="fig-trio">

  <img src="{{ '/papers/proxrouter/figures/knn-ood-pareto.png' | relative_url }}" alt="Accuracy versus cost on math outliers">

  <img src="{{ '/papers/proxrouter/figures/knn-ood-accuracy.png' | relative_url }}" alt="Accuracy versus lambda">

  <img src="{{ '/papers/proxrouter/figures/knn-ood-cost.png' | relative_url }}" alt="Cost per query versus lambda">

  <figcaption>Router performance on GSM8k and SVAMP outliers. <b>Left:</b> mean

  accuracy-cost curve for outlier queries. <b>Center:</b> average accuracy against

  $\lambda$. <b>Right:</b> average cost per query against $\lambda$.</figcaption>

</figure>

<figure>

  <img src="{{ '/papers/proxrouter/figures/knn-match-accuracy.png' | relative_url }}" alt="Bar chart of routing match accuracy against post-hoc top-1, top-3 and top-5 models">

  <figcaption>Routing match accuracy against the post-hoc top-(1,3,5) most suitable

  models at three values of $\lambda$. Higher values indicate better routing agreement.
  $k$NN-Prox routes approximately 49% of outlier queries to a post-hoc top-5 model,

  compared with 29% for $k$NN-Base and 61% for $k$NN-AllSee.</figcaption>

</figure>

<h2 class="section">Why some outliers hurt more than others</h2>

The performance gap between AllSee and Base depends strongly on the outlier task. When
an outlier task favors models that are also effective on the inlier tasks, the router
can partially transfer information from its existing reference set and the gap remains
small. Outlier tasks with substantially different model preferences produce larger
degradations.

To quantify this relationship, we compare model rankings between inlier and outlier
tasks. For a task $t$ and value of $\lambda$, let $S_z(t,\lambda)$ denote the set of the
top-$z$ models according to objective value. We define their top-$z$ Jaccard similarity as

$$J_z(t_\text{out}, t_\text{in}, \lambda) = \frac{|S_z(t_\text{out},\lambda) \cap S_z(t_\text{in},\lambda)|}{|S_z(t_\text{out},\lambda) \cup S_z(t_\text{in},\lambda)|}$$

<figure>

  <img src="{{ '/papers/proxrouter/figures/jaccard.png' | relative_url }}" alt="Jaccard overlap of top-5 models across outlier task sets">

  <figcaption>Jaccard overlap of the top-5 models for three outlier task sets with

  their corresponding inlier tasks. The math tasks (GSM8k, SVAMP) exhibit the smallest
  overlap with the inlier model rankings and correspondingly the largest gap between

  the AllSee and Base routers.</figcaption>

</figure>

A low average $J_z^\lambda$ indicates that outlier and inlier tasks favor substantially
different models, and is associated with a larger Base-AllSee performance gap. Higher
values indicate greater agreement in model preferences and correspondingly smaller gaps.
This analysis also provides a signal for **when to retrain**. Persistently low
$J_z^\lambda$ for incoming queries indicates that the model preferences of the test
distribution differ from those represented by the existing router. Retraining can be
triggered when $J_z^\lambda$ falls below a threshold selected on validation data, balancing
the cost of evaluating new queries across the model pool against the expected loss in
routing performance. ProxRouter improves robustness between such retraining events.

<h2 class="section">Computational overhead</h2>

The additional computation consists of evaluating the prior and applying one exponential
reweighting. We measure routing latency across varying training-set sizes, values of
$k$, numbers of clusters, and embedding dimensions from 128 to 768. Across these
settings, the additional latency remains in the millisecond range—roughly two orders of
magnitude below the seconds-scale latency of autoregressive LLM generation.
The storage requirement is similarly modest: 10,000 training-query embeddings at 768
dimensions in float32 occupy approximately 35 MB, and routing can be executed on a
single CPU thread.

<h2 class="section">Summary</h2>

We study train-test mismatch in LLM query routing and introduce ProxRouter, a
proximity-weighted framework for nonparametric routers that improves robustness to
outlier queries while preserving inlier performance. The central idea is to prioritize
reference queries that are close to the test query when estimating each model's
accuracy-cost objective. Exponential tilting provides a principled mechanism for
controlling the resulting bias-variance trade-off with a single hyperparameter.
We further formulate a broad family of nonparametric routers within a common weighted-
aggregation view, providing a unified perspective for analysis and implementation. We
also show how shifts in model rankings can inform retraining decisions. Although this
work focuses on $k$NN and $K$Means routers, extending the framework to fuzzy, spectral,
and kernel-based methods is a natural direction for future work.

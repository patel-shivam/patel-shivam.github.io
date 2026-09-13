---
layout: paper
permalink: /papers/federated-routing/
title: "Federate the Router: Learning Language Model Routers with Sparse and Decentralized Evaluations"
short_title: Federated Routing
description: >-
  A federated framework for training LLM query routers when query-model evaluation data
  is fragmented across privacy-sensitive clients. Covers both a parametric MLP router and
  a nonparametric K-Means router, with convergence and suboptimality guarantees.
image: /papers/federated-routing/figures/global-test.png
venue: arXiv
authors:
  - name: Baris Askin
    affil: 1
    corresponding: true
  - name: Shivam Patel
    url: /
    affil: 1
    me: true
    corresponding: true
  - name: Anupam Nayak
    affil: 1
    corresponding: true
  - name: Andrea Vigano
    affil: 1
  - name: Jiin Woo
    affil: 1
  - name: Gauri Joshi
    url: https://www.andrew.cmu.edu/user/gaurij/
    affil: 1
  - name: Carlee Joe-Wong
    affil: 1
affiliations:
  - name: Carnegie Mellon University
    url: https://www.cmu.edu
corresponding_note: Equal contribution
links:
  - name: arXiv
    url: https://arxiv.org/abs/2601.22318
  - name: PDF
    url: https://arxiv.org/pdf/2601.22318
tldr: >-
  LLM query routers are trained on records of which model answered which query, how well,
  and at what price. Every existing router assumes this evaluation data can be gathered in
  one place. In practice it cannot: the valuable records are user prompts, enterprise logs
  and internal code sitting on clients that cannot share them. Each client also holds a
  narrow slice of the query distribution and, because evaluating a query is expensive,
  observes only one model per query and a biased subset of the model pool. We give
  federated training procedures for both canonical router families, the parametric
  **MLP-Router** and the nonparametric **K-Means-Router**, so that clients learn a shared
  routing policy without exchanging queries. We prove convergence for the parametric
  router and show for both families that suboptimality improves with the pooled dataset
  size rather than the local one. Federated routers beat every client-local router on the
  global query distribution, and also on each client's *own* test set, because collaboration
  repairs the missing model coverage. Under extreme heterogeneity an adaptive mixture of
  the federated and local routers restores the client-specific fit.
bibtex: |
  @article{askin2026federate,
    title   = {Federate the Router: Learning Language Model Routers
               with Sparse and Decentralized Evaluations},
    author  = {Askin, Baris and Patel, Shivam and Nayak, Anupam and
               Vigano, Andrea and Woo, Jiin and Joshi, Gauri and
               Joe-Wong, Carlee},
    journal = {arXiv preprint arXiv:2601.22318},
    year    = {2026},
    url     = {https://arxiv.org/abs/2601.22318}
  }
---

<h2 class="section">Training a router when nobody holds the training set</h2>

Language models differ enormously in capability and in price, and no single model is best
for every query. A query router exploits this by selecting, per query, the model that
best balances answer quality against inference cost. Routers are learned from
*query-model evaluation data*: records of the form *this query was sent to that model,
the answer scored this well, and it cost this much*.

Collecting that data is the step every router design takes for granted, and it is the
step that breaks first in practice. Evaluating a broad query set against every model in
the pool is expensive, and the expense recurs each time the pool changes, which is now
close to continuously. Public benchmarks are a poor substitute because real query
distributions are long-tailed and shift over time. Most importantly, the evaluations
worth having are not public at all: they are produced by end users and organizations
running LLM-backed products, and consist of customer conversations, internal code and
proprietary prompts that privacy and regulatory constraints prevent from being pooled
into a central dataset.

This work asks whether a router can be trained without ever moving that data. The
framework is federated: each client trains on its own records, a server aggregates only
model parameters or summary statistics, and every client receives a routing policy shaped
by all of the others' experience.

<figure>
  <img loading="lazy" decoding="async" width="1600" height="409" src="{{ '/papers/federated-routing/figures/system-diagram.png' | relative_url }}" alt="Three-panel diagram: clients hold queries from different tasks evaluated on different models; clients train local routers that a central server aggregates; the resulting router scores accuracy and cost per model and routes each query">
  <figcaption>The setting in three parts. Clients hold queries drawn from different tasks,
  and each query carries an evaluation from only one language model, chosen
  non-uniformly per client. Clients train local routers and a server aggregates them into
  a global router. At inference the router estimates accuracy and cost for every model and
  selects the one maximizing the objective.</figcaption>
</figure>

<h2 class="section">What a single client actually holds</h2>

Three properties of client-held data make local router training hard, and they compound.

The first is a **skewed query mix**. Clients are partitioned by task, so each sees the
prompt space in very different proportions. At the moderate skew used in the main
experiments this is a difference of emphasis rather than of support: a typical client's
queries still land across 44 to 74 per cent of the occupied embedding space. The
composition is what differs, and it differs sharply. HellaSwag accounts for 68% of one
client's queries and 3% of another's; MMLU ranges from 18% to 63%. Pushed to the extreme
setting studied later, the mixes become close to disjoint and the difference becomes one
of support after all.

The second is **sparse evaluation**. Scoring one query against all models costs as many
generations as there are models, so realistic logs record the outcome for the single
model that was actually called. Every training record is therefore one cell of a large
query-by-model table, and the rest of the row is missing.

The third is **imbalanced model coverage**. Which model a client calls is a property of
that client, not of the query, so the observed models are highly non-uniform and differ
across clients. A client may hold thousands of evaluations for one model and none at all
for another, which leaves its local estimates for the missing models unanchored.

<figure>
  <img loading="lazy" decoding="async" width="1400" height="802" src="{{ '/papers/federated-routing/figures/model-coverage.png' | relative_url }}" alt="Bubble plot of per-model evaluation proportions for each of ten clients, with bubble area varying widely and several cells empty">
  <figcaption>Per-model share of each client's local dataset. Model assignment is drawn
  from a client-specific Dirichlet distribution, so coverage is concentrated on a few
  models per client and several client-model pairs are empty. The router still has to
  produce an estimate for every model in the pool.</figcaption>
</figure>

The widget below reads the actual client partition and evaluation counts from the
experiments. Selecting a client shows where its queries land, how its task mix departs
from the pooled one, and which models it has any evidence about; pooling the clients
shows what federated training gets instead. It is worth stepping through a few clients to
see which of the three obstacles actually binds: the query footprints overlap heavily,
while the model coverage does not.

{% include_relative clients.part.html %}

In the configuration used throughout the paper, ten clients hold 27,368 training queries
between them, and the largest holds 4.6 times as many as the smallest. The contrast
between the two panels is the point. On the query side the clients overlap: each covers
well over half the occupied space on average. On the model side they do not. Nine of the
110 client-model pairs contain no evaluations whatsoever, and another fourteen contain
fifteen or fewer, which is nothing to estimate an accuracy and a price from. Claude v1
illustrates the imbalance: it is the most-evaluated model overall, with 4,727 records,
yet those records sit on only five of the ten clients, and a single client accounts for
2,832 of them. Pooled across clients every model has at least 1,416 evaluations and all
fifteen task groups are present.

This is why the results later show the gains arriving where they do. What a client
lacks is not familiarity with its own queries but evidence about models it seldom calls.

<h2 class="section">The routing objective</h2>

Given a pool of models $\mathcal{M}$ and a query embedded as $\mathbf{x}$, write
$\text{acc}(\mathbf{x}, m)$ for the expected quality of model $m$'s response and
$\text{cost}(\mathbf{x}, m)$ for its expected price. The router maximizes a Lagrangian
combination of the two:

$$U_{\lambda}(\mathbf{x}, m) \;=\; \text{acc}(\mathbf{x}, m) \;-\; \lambda \cdot \text{cost}(\mathbf{x}, m)$$

Small $\lambda$ prioritizes accuracy, large $\lambda$ prioritizes cheap models, and
sweeping $\lambda$ traces the accuracy-cost frontier reported in every experiment.

Neither quantity is known before the model has answered, so the router learns estimators
$\mathsf{A}(\mathbf{x}, m) \approx \text{acc}(\mathbf{x}, m)$ and
$\mathsf{C}(\mathbf{x}, m) \approx \text{cost}(\mathbf{x}, m)$ and then routes greedily:

$$\pi_{\lambda}(\mathbf{x}) \;=\; \arg\max_{m \in \mathcal{M}} \big[\, \mathsf{A}(\mathbf{x}, m) - \lambda\, \mathsf{C}(\mathbf{x}, m) \,\big]$$

Estimating accuracy and cost rather than directly classifying the best model matters more
here than in the centralized case. Clients differ in how much they care about price, and
because $\lambda$ enters only at inference time, each can choose its own operating point
on the frontier without retraining anything.

A client $i$ holds a dataset of tuples

$$\mathcal{D}_i \;=\; \big\{ (\mathbf{x}_i^{(j)},\, m_i^{(j)},\, \widehat{\text{acc}}_i^{(j)},\, \widehat{\text{cost}}_i^{(j)}) \big\}_{j=1}^{D_i}$$

where $m\_i^{(j)}$ is the one model that query $j$ was actually evaluated on. The union
over clients is $\mathcal{D}$, with $D = \sum\_i D\_i$ records in total.

<h2 class="section">Federated MLP-Router</h2>

The parametric router is a shared trunk $h\_{\theta}$ mapping a query embedding to a
hidden representation, followed by one linear head per model producing that model's
accuracy and cost estimates. Putting the heads on a common trunk is what allows an
evaluation of one model to inform predictions about the others, which is precisely the
leverage needed when each record covers a single model.

Training minimizes squared error over the observed cells only:

$$\mathcal{L}(\theta) \;=\; \frac{1}{|\mathcal{D}|} \sum_{i=1}^{N} \sum_{j=1}^{|\mathcal{D}_i|} \Big[ \ell\big(\widehat{\text{acc}}_i^{(j)},\, \mathsf{A}_{\theta}(\mathbf{x}_i^{(j)}, m_i^{(j)})\big) \;+\; \ell\big(\widehat{\text{cost}}_i^{(j)},\, \mathsf{C}_{\theta}(\mathbf{x}_i^{(j)}, m_i^{(j)})\big) \Big]$$

This objective is a sum of per-client terms, so it fits FedAvg directly. Each round the
server broadcasts the current weights to a participating subset of clients; each client
runs several local steps on its private data; the server averages the returned
parameters weighted by client dataset size. Raw queries never leave the client, and
because the cost of a round is set by the size of the router rather than by the size of
the evaluation logs, it stays small.

<h2 class="section">Federated K-Means-Router</h2>

The nonparametric router does not train a predictor at all. It partitions the embedding
space into regions and treats accuracy and cost as approximately constant within each
region, so the estimator for a new query is the empirical average over the region it
falls into. This makes it naturally incremental: a new model requires new averages, not
new training.

Federating it means federating two different things, and the algorithm makes two passes
to do so.

<h3>Pass one: agreeing on the regions</h3>

Each client runs $K$-means over its own query embeddings and sends the server its cluster
centroids together with the number of points in each. The server then runs a *weighted*
$K$-means over the pooled centroids, each centroid weighted by its cluster size, so the
global centers minimize a size-weighted squared distance over the underlying data. The
resulting $K\_{\text{global}}$ centers are broadcast back. Only centroids and counts are
transmitted; no query leaves its client.

<h3>Pass two: filling in the statistics</h3>

Each client assigns its own samples to the global centers and, for every
(cluster, model) pair it has data for, computes the mean observed accuracy and cost
along with the number of samples behind them. Pairs with no samples are simply not
reported, which is how the sparsity is handled rather than papered over. The server
aggregates the reported means into a global estimate weighted by the sample counts:

$$\mathsf{A}_k^{(m)} \;=\; \frac{\sum_i n_{i,k}^{(m)}\, \bar{a}_{i,k}^{(m)}}{\sum_i n_{i,k}^{(m)}}$$

with the cost statistics aggregated the same way. At inference a query is mapped to its
nearest global center and routed using that cell's accuracy and cost table.

The aggregation step is where the sparsity problem is repaired. A (cluster, model) cell
that is empty or nearly empty on any single client can be well populated once the counts
from all clients are summed, and it is those per-cell counts that control the estimation
error.

The walkthrough below runs both passes on the real client partition, so the centroids,
the weighting and the final coverage are computed live rather than illustrated.

{% include_relative kmeans.part.html %}

<h2 class="section">What collaboration buys</h2>

Two results support the empirical picture.

For the parametric router, federated averaging on this objective converges at the
standard rate for nonconvex FedAvg under bounded heterogeneity, unbiased gradients and
smoothness. With $N$ clients, $\tau$ local steps and $T$ rounds, the smallest expected
squared gradient norm along the trajectory is bounded by
$\widetilde{\mathcal{O}}\big((1 + A\sigma^2)/\sqrt{N \tau T}\big)$, where $\sigma^2$
bounds the mini-batch gradient heterogeneity and $A = N \sum\_i (D\_i/D)^2$ measures how
unevenly the data is spread. When clients hold equal shares the rate improves linearly in
the number of clients.

The second result concerns the routing policy rather than the optimization. Define the
suboptimality of a policy $\widehat{\pi}$ against the best possible policy $\pi^{\star}$
on a test distribution as

$$\text{Subopt}(\widehat{\pi}) \;=\; \mathbb{E}_{\mathbf{x}} \big[\, U_{\lambda}(\mathbf{x}, \pi^{\star}(\mathbf{x})) - U_{\lambda}(\mathbf{x}, \widehat{\pi}(\mathbf{x})) \,\big]$$

For the parametric router, a client training alone incurs suboptimality on the order of
$\widetilde{\mathcal{O}}(1/\sqrt{D\_i})$, while the federated router, optimizing over the
union, incurs $\widetilde{\mathcal{O}}(1/\sqrt{D})$. Since $D = \sum\_i D\_i$, the
federated bound is the stronger one, and the gap widens for exactly the clients that hold
the least data.

For the clustering router the bound decomposes into three interpretable terms: the
average distance from a query to its assigned center, a term measuring the mismatch
between training and test distributions across clusters, and a statistical term scaling
as $1/\sqrt{n\_{\min}}$, where $n\_{\min}$ is the fewest samples behind any
(cluster, model) estimate. Federated aggregation improves the first term, because the
global centers are fitted to the pooled query distribution, and improves the third
directly, because summing counts across clients raises the smallest cell.

<h2 class="section">Experimental setup</h2>

Experiments use **RouterBench**, which provides evaluations of 11 language models across
8 public datasets, giving 36,497 queries after preprocessing. Queries are embedded with
the all-mpnet-base-v2 sentence encoder; two other encoders are tested in the appendix
with no significant difference. The appendix repeats the full study on
**ProxRouter-Data**, covering 14 models over 10 datasets.

The federated system simulates $N = 10$ clients with 0.6 of them participating in each
round. Two separate sources of heterogeneity are induced:

- **Query heterogeneity.** Queries are partitioned across clients by a Dirichlet
  distribution with concentration $\alpha = 0.6$ over task labels, so each client's
  prompt mix is skewed towards a few tasks.
- **Model heterogeneity.** For each client a distribution over the 11 models is drawn
  from a Dirichlet with $\alpha = 0.45$, and each of that client's queries is logged
  against a single model sampled from it. This produces the uneven, partly empty coverage
  shown above.

Each client splits its data 75/25 into local train and test; the global train and test
sets are the unions of those splits. Accuracy-cost curves come from sweeping $\lambda$
over a hundred points on a logarithmic grid from $10^{-2}$ to $10^{7}$, and each curve is
summarized by its normalized area under the curve, with higher values indicating a better
frontier.

The parametric router uses a two-layer trunk of width 512 with LayerNorm, GELU and
dropout 0.1, and per-model heads predicting an accuracy logit and a normalized cost. It
is trained with FedAvg using AdamW at learning rate $10^{-3}$, weight decay
$3 \times 10^{-4}$, one local epoch per round, batch size 128 and gradient clipping at
norm 1.0. The clustering router uses $K\_{\text{local}} = 15$ clusters per client and
$K\_{\text{global}} = 20$ at the server, with Euclidean distance, three restarts and at
most thirty Lloyd iterations, run as a single federated clustering stage.

The comparison throughout is against **client-local** routers: the same router family
trained by each client alone on its own data, with no collaboration.

<h2 class="section">Results</h2>

<h3>Generalizing to the global query distribution</h3>

The first question is whether a client's router works on queries beyond its own
application. Each client-local router and the federated router are evaluated on the
global test set.

<figure>
  <img loading="lazy" decoding="async" width="1300" height="1126" src="{{ '/papers/federated-routing/figures/global-test.png' | relative_url }}" alt="Accuracy versus average cost on the global test set; the federated curve sits above all ten client-local curves for both router families">
  <figcaption>Accuracy-cost frontiers on the global test distribution, for the parametric
  router (top) and the clustering router (bottom). Normalized AUC is shown in the legend.
  The federated router dominates every client-local router, with the wider margin for the
  clustering router.</figcaption>
</figure>

The federated parametric router reaches 0.75 normalized AUC against 0.63 to 0.72 for the
ten client-local routers. The clustering router shows the larger gap: 0.75 federated
against a client-local range of 0.55 to 0.70. That ordering is what the theory predicts.
Clustering quality degrades sharply when centers are fitted to a small, skewed sample,
and a client-local clustering router inherits both a poor partition and thin per-cell
statistics, whereas the federated version fixes the partition first and then pools the
statistics.

<h3>Better routing on a client's own queries</h3>

A subtler question is whether collaboration helps a client on the distribution it
actually serves. It might not: the federated router is fitted to everyone's queries, and
that could dilute the fit to any one client. Evaluating each router on its own client's
local test set shows the opposite.

<figure>
  <img loading="lazy" decoding="async" width="1500" height="868" src="{{ '/papers/federated-routing/figures/local-test.png' | relative_url }}" alt="Six panels for three representative clients and both router families; the federated frontier sits above the client-local frontier in every panel">
  <figcaption>Federated against client-local routers on each client's own local test set,
  for three representative clients. The federated router improves the frontier
  in-distribution as well, because it repairs the model coverage the client was missing.</figcaption>
</figure>

For the parametric router, clients 4, 6 and 8 improve from 0.70, 0.69 and 0.72 to 0.76,
0.71 and 0.76. For the clustering router the same clients improve from 0.71, 0.64 and
0.64 to 0.76, 0.72 and 0.77. The federated router wins on every one of the ten clients
for both families.

The explanation is model coverage rather than query coverage. On its own queries a client
already knows the query distribution; what it lacks is a reliable estimate of how the
models it rarely called would have performed. Those estimates arrive from other clients,
and a routing decision only needs the *comparison* between models to be right.

<h3>How much decentralization costs</h3>

Federated training is worth little if it gives up a large fraction of what a centralized
router would achieve, so the natural reference point is a router trained on the pooled
data directly.

<figure>
  <img loading="lazy" decoding="async" width="1500" height="973" src="{{ '/papers/federated-routing/figures/federated-vs-centralized.png' | relative_url }}" alt="Accuracy-cost curves for federated and centralized variants of both routers, essentially overlapping">
  <figcaption>Federated against centralized training on pooled data. Both router families
  reach 0.75 normalized AUC either way, so decentralization costs essentially nothing
  here.</figcaption>
</figure>

All four curves land at 0.75. Whatever federation loses to partial participation and
infrequent averaging, it recovers from the same pooled signal the centralized baseline
uses.

<h3>Adding models to the pool</h3>

Model pools change constantly, and retraining a router from scratch for each new release
is not viable. Three models are withheld during initial training and then introduced,
with each client evaluating them on a small calibration subset of about 10% of its
prompts. The parametric router appends a head per new model and trains only that head,
leaving the trunk and existing heads frozen. The clustering router needs no training at
all: the embedding space is unchanged, so only the per-cell statistics for the new models
are required.

<figure>
  <img loading="lazy" decoding="async" width="1300" height="717" src="{{ '/papers/federated-routing/figures/model-expansion.png' | relative_url }}" alt="Accuracy-cost frontiers before and after three withheld models are introduced; both routers improve after the calibration step">
  <figcaption>Global test frontiers before and after three withheld models join the pool
  and are incorporated through a lightweight calibration step.</figcaption>
</figure>

Both routers improve from 0.732 to 0.748 and 0.749 respectively, without a full retrain.

<h3>Adding clients to the system</h3>

Clients also arrive after deployment, and a system that accommodates them should improve
from their data, avoid forgetting the clients it already served, and not require the
existing clients to participate again. Training starts with 7 clients, after which 3 new
ones join carrying 30% of the task labels in the system.

<figure>
  <img loading="lazy" decoding="async" width="1500" height="594" src="{{ '/papers/federated-routing/figures/client-expansion.png' | relative_url }}" alt="Global test frontiers before and after three new clients join, for both router families; both improve">
  <figcaption>Adapting to newly joined clients without further participation from the
  original seven. The parametric router continues training on the new clients with a
  distillation penalty against its previous outputs; the clustering router simply updates
  its cluster statistics by weighted averaging.</figcaption>
</figure>

The parametric router improves from 0.71 to 0.74 and the clustering router from 0.72 to
0.74. The parametric case needs the distillation term to hold onto the original policy
while training on new clients alone; the clustering router gets the same effect for free,
since incorporating a new client is an update to a weighted average.

<h3>Extreme heterogeneity and adaptive personalization</h3>

The results so far use $\alpha = 0.6$, a moderate skew. Pushing the Dirichlet
concentration to $\alpha = 0.03$ makes clients nearly single-task, and there the global
router can genuinely be the wrong object for a given client: if one client sees almost
only biology prompts while the rest see almost none, a policy fitted to everyone is
poorly matched to it.

The proposed remedy is a per-client, per-model mixture of the federated and local
estimators. Each client measures the mean absolute calibration error of both estimators
on its own training points, requiring no additional model calls, and weights them
inversely to their errors, separately for accuracy and cost.

<figure>
  <img loading="lazy" decoding="async" width="1500" height="880" src="{{ '/papers/federated-routing/figures/personalization.png' | relative_url }}" alt="Six panels comparing client-local, federated and personalized frontiers under extreme heterogeneity; the personalized curve tracks the better of the other two">
  <figcaption>Local test frontiers under extreme heterogeneity ($\alpha = 0.03$),
  comparing client-local training, federated training and the adaptive mixture of the two.
  The mixture tracks whichever of the two is stronger for that client.</figcaption>
</figure>

Under this regime the federated parametric router does fall below client-local training
on some clients: on clients 4 and 6 it reaches 0.69 and 0.71 against local scores of 0.70
and 0.73. The mixture recovers the loss, reaching 0.72 on both, and on client 8, where
the federated router is already the stronger of the two at 0.80, it stays level at 0.79.
The clustering router does not exhibit the failure in the first place, since its local
variant suffers more from thin per-centroid coverage than it gains from specialization.

<h2 class="section">Summary</h2>

Query routers have been studied as though their training data were freely available in
one place. Relaxing that assumption changes the problem: the data is split across
clients, each client sees a narrow slice of the query space, and because evaluation is
expensive each query carries the verdict of only one model, chosen non-uniformly. This
work provides federated training for both standard router families under those
conditions, with a convergence guarantee for the parametric router and suboptimality
bounds for both showing the benefit scales with the pooled rather than the local dataset.

The empirical result that carries the most weight is not the improvement on the global
distribution, which is what one would expect from more data. It is that federated
training also improves each client on its *own* queries. A client's limitation is not
that it misunderstands its users, but that it has never observed most of the model pool
on them, and that gap is filled by other clients without a single query changing hands.

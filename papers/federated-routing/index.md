---
layout: paper
permalink: /papers/federated-routing/
title: "Federate the Router: Learning Language Model Routers with Sparse and Decentralized Evaluations"
short_title: Federated Routing
description: >-
  A federated framework for training LLM query routers when query-model evaluation data
  is fragmented across privacy-sensitive clients. The framework supports both a parametric
  MLP router and a nonparametric K-Means router, with convergence and suboptimality
  guarantees.
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
  LLM query routers are trained on records describing which model answered each query, the
  resulting response quality, and the associated cost. Most existing methods assume that
  these evaluations can be centralized. In practice, the most informative records—user
  prompts, enterprise logs, and internal code—often remain with clients because they cannot
  be shared. Each client may also observe a narrow query distribution and, because model
  evaluation is costly, only one model per query and a biased subset of the model pool. We
  introduce federated training procedures for both canonical router families, the parametric
  **MLP-Router** and the nonparametric **K-Means-Router**, enabling clients to learn a shared
  routing policy without exchanging queries. We prove convergence for the parametric router
  and derive suboptimality bounds for both families that characterize the benefits of pooling
  evaluations across clients. In our experiments, federated routers outperform every
  client-local router on both the global query distribution and each client's *own* test set,
  primarily by increasing effective model coverage. Under extreme heterogeneity, an adaptive
  mixture of the federated and local routers mitigates client-specific distribution mismatch.
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

<h2 class="section">Training routers from decentralized evaluation data</h2>

Language models vary substantially in capability and cost, and no single model is optimal
for every query. A query router exploits this by selecting, per query, the model that
best balances response quality and inference cost. Routers are learned from
*query-model evaluation data*: records specifying *the model selected for a query, the
quality of its response, and the resulting cost*.

Most router designs assume centralized access to these data. In practice, however,
evaluating a broad query set against every model in the pool is expensive, and this cost
recurs whenever the pool changes—an increasingly frequent event. Public benchmarks are an
imperfect substitute because operational query distributions are long-tailed and shift over
time. Most importantly, the evaluations most representative of deployment are often not
public: they are produced by end users and organizations running LLM-backed products, and
consist of customer conversations, internal code and proprietary prompts that privacy or
regulatory constraints may prevent from being pooled into a central dataset.

This work studies whether a router can be trained without centralizing these data. The
framework is federated: each client trains on its own records, a server aggregates only
model parameters or summary statistics, and every client receives a shared routing policy
learned from aggregated cross-client information.

<figure>
  <img loading="lazy" decoding="async" width="1600" height="409" src="{{ '/papers/federated-routing/figures/system-diagram.png' | relative_url }}" alt="Three-panel diagram: clients hold queries from different tasks evaluated on different models; clients train local routers that a central server aggregates; the resulting router scores accuracy and cost per model and routes each query">
  <figcaption>Overview of the federated routing framework. Clients hold queries drawn from
  different tasks, with each query evaluated by only one language model selected
  non-uniformly per client. Clients train local routers and a server aggregates them into
  a global router. At inference time, the router estimates accuracy and cost for every model
  and selects the model that maximizes the routing objective.</figcaption>
</figure>

<h2 class="section">Challenges of client-local evaluation data</h2>

Three interacting properties of client-held data make local router training challenging.

The first is a **skewed query mix**. Clients are partitioned by task, so each sees the
prompt space in different proportions. At the moderate skew used in the main experiments,
this primarily changes distributional mass rather than support: a typical client's queries
still occupy 35 to 63 per cent of the populated embedding space. The composition
nevertheless differs substantially. HellaSwag accounts for 68% of one client's queries and
3% of another's; MMLU ranges from 18% to 63%. Pushed to the extreme setting studied later,
the client distributions become nearly disjoint, producing substantial differences in
support.

The second is **sparse evaluation**. Evaluating each query against all models requires one
generation per model, so practical logs often record the outcome only for the model that
was actually called. Every training record is therefore one cell of a large query-by-model
table, and the rest of the row is missing.

The third is **imbalanced model coverage**. Model selection follows a client-specific
logging distribution, so observed model frequencies are highly non-uniform and differ
across clients. A client may hold thousands of evaluations for one model and none at all
for another, leaving local estimates for unobserved models poorly constrained.

<figure>
  <img loading="lazy" decoding="async" width="1400" height="802" src="{{ '/papers/federated-routing/figures/model-coverage.png' | relative_url }}" alt="Bubble plot of per-model evaluation proportions for each of ten clients, with bubble area varying widely and several cells empty">
  <figcaption>Per-model composition of each client's local dataset. Model assignments are
  drawn from a client-specific Dirichlet distribution, so coverage is concentrated on a few
  models for each client, leaving several client-model pairs unobserved. Nevertheless, the
  router must produce an estimate for every model in the pool.</figcaption>
</figure>

The widget below uses the client partitions and evaluation counts from the experiments.
Selecting a client shows the distribution of its queries, how its task mixture differs from
the pooled distribution, and which models have observed evaluations. The pooled view shows
the aggregate information available to federated training. Across clients, the query
distributions overlap substantially, whereas model coverage remains sparse and imbalanced.

The two sliders regenerate the partition. They expose both Dirichlet concentration
parameters: the concentration over task labels, which controls how unevenly queries are
partitioned across clients, and the concentration over the model pool, which controls how
narrow each client's logging distribution becomes. Reducing the first to the
$\alpha = 0.03$ used in the extreme-heterogeneity study transforms moderate distributional
skew into nearly disjoint client distributions, leaving some clients with very few samples.

{% include_relative clients.part.html %}

In the configuration used throughout the paper, ten clients collectively hold 27,368
training queries, and the largest holds 4.6 times as many as the smallest. The contrast
between the two panels is central to the setting. In query space, the clients overlap: each
covers roughly half of the occupied embedding space on average. In model coverage, however,
they differ substantially. Nine of the 110 client-model pairs contain no evaluations
whatsoever, and another fourteen contain fifteen or fewer, providing insufficient data for
reliable accuracy and cost estimation. Claude v1 illustrates the imbalance: it is the
most-evaluated model overall, with 4,727 records, yet those records are distributed across
only five of the ten clients, and a single client accounts for 2,832 of them. Pooled across
clients every model has at least 1,416 evaluations and all fifteen task groups are present.

These statistics indicate that the primary limitation of a client-local router is not
coverage of its own query distribution, but evidence about models that the client rarely
invokes.

<h2 class="section">The routing objective</h2>

Given a pool of models $\mathcal{M}$ and a query embedded as $\mathbf{x}$, write
$\text{acc}(\mathbf{x}, m)$ for the expected quality of model $m$'s response and
$\text{cost}(\mathbf{x}, m)$ for its expected inference cost. The router maximizes a
Lagrangian combination of the two:

$$U_{\lambda}(\mathbf{x}, m) \;=\; \text{acc}(\mathbf{x}, m) \;-\; \lambda \cdot \text{cost}(\mathbf{x}, m)$$

Smaller $\lambda$ prioritizes accuracy, whereas larger $\lambda$ favors lower-cost models;
sweeping $\lambda$ traces the accuracy-cost frontiers reported in the experiments.

Neither quantity is known before the model has answered, so the router learns estimators
$\mathsf{A}(\mathbf{x}, m) \approx \text{acc}(\mathbf{x}, m)$ and
$\mathsf{C}(\mathbf{x}, m) \approx \text{cost}(\mathbf{x}, m)$ and then routes greedily:

$$\pi_{\lambda}(\mathbf{x}) \;=\; \arg\max_{m \in \mathcal{M}} \big[\, \mathsf{A}(\mathbf{x}, m) - \lambda\, \mathsf{C}(\mathbf{x}, m) \,\big]$$

Estimating accuracy and cost rather than directly classifying the best model is
particularly useful in the federated setting. Clients may have different cost-quality
preferences, and because $\lambda$ enters only at inference time, each can choose its own
operating point on the frontier without retraining the router.

A client $i$ holds a dataset of tuples

$$\mathcal{D}_i \;=\; \big\{ (\mathbf{x}_i^{(j)},\, m_i^{(j)},\, \widehat{\text{acc}}_i^{(j)},\, \widehat{\text{cost}}_i^{(j)}) \big\}_{j=1}^{D_i}$$

where $m\_i^{(j)}$ is the one model that query $j$ was actually evaluated on. The union
over clients is $\mathcal{D}$, with $D = \sum\_i D\_i$ records in total.

<h2 class="section">Federated MLP-Router</h2>

The parametric router is a shared trunk $h\_{\theta}$ mapping a query embedding to a
hidden representation, followed by one linear head per model producing that model's
accuracy and cost estimates. The shared trunk allows an evaluation of one model to inform
predictions about the others, which is precisely the form of cross-model transfer that is
valuable when each record contains feedback for only one model.

Training minimizes squared error over the observed cells only:

$$\mathcal{L}(\theta) \;=\; \frac{1}{|\mathcal{D}|} \sum_{i=1}^{N} \sum_{j=1}^{|\mathcal{D}_i|} \Big[ \ell\big(\widehat{\text{acc}}_i^{(j)},\, \mathsf{A}_{\theta}(\mathbf{x}_i^{(j)}, m_i^{(j)})\big) \;+\; \ell\big(\widehat{\text{cost}}_i^{(j)},\, \mathsf{C}_{\theta}(\mathbf{x}_i^{(j)}, m_i^{(j)})\big) \Big]$$

This objective decomposes into per-client terms and can therefore be optimized with FedAvg.
In each round, the server broadcasts the current weights to a participating subset of
clients; each client runs several local steps on its private data; the server averages the
returned parameters using weights proportional to client dataset size. Raw queries remain
local, and because the cost of a round is set by the size of the router rather than by the
size of the evaluation logs, communication remains compact.

The trunk-head decomposition provides a natural mechanism for sparse supervision. Every
record a client holds updates the trunk, but it updates only the head of the model on which
it was actually evaluated, so heads corresponding to unobserved models remain unchanged
locally. Since the trunk carries almost all of the parameters, most of what each client
learns is shared, while model-specific heads receive updates during aggregation from clients
that observed the corresponding models.

<h2 class="section">Federated K-Means-Router</h2>

The nonparametric router does not train a predictive model. Instead, it partitions the
embedding space into regions and treats accuracy and cost as approximately constant within
each region, so the estimator for a new query is the empirical average over the region it
falls into. This design is naturally incremental: incorporating a new model requires new
cluster-level averages rather than retraining the full router.

The federated algorithm aggregates both the embedding-space partition and the associated
evaluation statistics in two passes.

<h3>Pass one: constructing global regions</h3>

Each client runs $K$-means over its own query embeddings and sends the server its cluster
centroids together with the number of points in each. The server then runs a *weighted*
$K$-means over the pooled centroids, each centroid weighted by its cluster size, so the
global centers minimize a size-weighted squared-distance objective over the local summaries.
The resulting $K\_{\text{global}}$ centers are broadcast back. Only centroids and counts are
transmitted; no query leaves its client.

<h3>Pass two: aggregating evaluation statistics</h3>

Each client assigns its own samples to the global centers and, for every
(cluster, model) pair it has data for, computes the mean observed accuracy and cost
along with the corresponding sample count. Pairs with no samples are not reported, allowing
the aggregation procedure to handle sparse observations explicitly. The server aggregates
the reported means into a global estimate weighted by the sample counts:

$$\mathsf{A}_k^{(m)} \;=\; \frac{\sum_i n_{i,k}^{(m)}\, \bar{a}_{i,k}^{(m)}}{\sum_i n_{i,k}^{(m)}}$$

with the cost statistics aggregated the same way. At inference a query is mapped to its
nearest global center and routed using that cell's accuracy and cost table.

Aggregation increases the effective sample size of sparse (cluster, model) cells. A cell
that is empty or nearly empty on any single client can receive substantially more samples
once the counts from all clients are summed; these per-cell counts directly control the
estimation error.

The walkthrough below executes both passes on the experimental client partition; the
centroids, weights, and resulting coverage are computed interactively.

{% include_relative kmeans.part.html %}

<h2 class="section">Theoretical benefits of collaboration</h2>

Two theoretical results characterize the benefits of federated collaboration.

For the parametric router, federated averaging on this objective converges at the
standard nonconvex FedAvg rate under bounded heterogeneity, unbiased gradients and
smoothness. With $N$ clients, $\tau$ local steps and $T$ rounds, the smallest expected
squared gradient norm along the trajectory is bounded by
$\widetilde{\mathcal{O}}\big((1 + A\sigma^2)/\sqrt{N \tau T}\big)$, where $\sigma^2$
bounds the mini-batch gradient heterogeneity and $A = N \sum\_i (D\_i/D)^2$ measures how
unevenly the data is distributed. When clients hold equal shares, the result yields the
standard linear speedup in the number of clients.

The second result concerns the routing policy rather than the optimization. Define the
suboptimality of a policy $\widehat{\pi}$ against the best possible policy $\pi^{\star}$
on a test distribution as

$$\text{Subopt}(\widehat{\pi}) \;=\; \mathbb{E}_{\mathbf{x}} \big[\, U_{\lambda}(\mathbf{x}, \pi^{\star}(\mathbf{x})) - U_{\lambda}(\mathbf{x}, \widehat{\pi}(\mathbf{x})) \,\big]$$

For the parametric router, the bound for a client-local estimator scales approximately as
$\widetilde{\mathcal{O}}(1/\sqrt{D\_i})$, while the federated router, optimizing over the
union, incurs $\widetilde{\mathcal{O}}(1/\sqrt{D})$. Since $D = \sum\_i D\_i$, the
federated bound can therefore be tighter, particularly for clients that hold limited local
data and when pooling improves coverage of the test distribution.

For the clustering router the bound decomposes into three interpretable terms: the
average distance from a query to its assigned center, a term measuring the mismatch
between training and test distributions across clusters, and a statistical term scaling
as $1/\sqrt{n\_{\min}}$, where $n\_{\min}$ is the minimum sample count across any
(cluster, model) estimate. Federated aggregation can reduce the first term because the
global centers are fitted to the pooled query distribution, and can improve the third
directly when summing counts across clients increases the minimum cell count.

<h2 class="section">Experimental setup</h2>

Experiments use **RouterBench**, which provides evaluations of 11 language models across
8 public datasets, giving 36,497 queries after preprocessing. Queries are embedded with
the all-mpnet-base-v2 sentence encoder; two additional encoders are evaluated in the
appendix without significant differences in centralized performance. The appendix repeats
the full study on **ProxRouter-Data**, covering 14 models over 10 datasets.

The federated system simulates $N = 10$ clients with a participation rate of 0.6 in each
round. Two separate sources of heterogeneity are induced:

- **Query heterogeneity.** Queries are partitioned across clients by a Dirichlet
  distribution with concentration $\alpha = 0.6$ over task labels, so each client's
  prompt distribution is concentrated on a subset of tasks.
- **Model heterogeneity.** For each client a distribution over the 11 models is drawn
  from a Dirichlet with $\alpha = 0.45$, and each of that client's queries is logged
  against a single model sampled from this distribution. This produces the uneven and
  partially missing coverage shown above.

Each client splits its data 75/25 into local train and test; the global train and test
sets are the unions of those splits. Accuracy-cost curves are obtained by sweeping $\lambda$
over one hundred logarithmically spaced values from $10^{-2}$ to $10^{7}$, and each curve is
summarized by its normalized area under the curve, with higher values indicating a better
frontier.

The parametric router uses a two-layer trunk of width 512 with LayerNorm, GELU and
dropout 0.1, and per-model heads predicting an accuracy logit and a normalized cost. It
is trained with FedAvg using AdamW at learning rate $10^{-3}$, weight decay
$3 \times 10^{-4}$, one local epoch per round, batch size 128 and gradient clipping at
norm 1.0. The clustering router uses $K\_{\text{local}} = 15$ clusters per client and
$K\_{\text{global}} = 20$ at the server, with Euclidean distance, three restarts and at
most thirty Lloyd iterations, run as a single federated clustering stage.

The primary comparison is against **client-local** routers: the same router family
trained independently by each client on its local data, without collaboration.

<h2 class="section">Results</h2>

<h3>Generalizing to the global query distribution</h3>

We first evaluate whether client-local routers generalize to queries beyond their local
distributions. Each client-local router and the federated router are evaluated on the
global test set.

<figure>
  <img loading="lazy" decoding="async" width="1300" height="1126" src="{{ '/papers/federated-routing/figures/global-test.png' | relative_url }}" alt="Accuracy versus average cost on the global test set; the federated curve sits above all ten client-local curves for both router families">
  <figcaption>Accuracy-cost frontiers on the global test distribution, for the parametric
  router (top) and the clustering router (bottom). Normalized AUC is shown in the legend.
  The federated router achieves a better frontier than every client-local router, with a
  larger margin for the clustering router.</figcaption>
</figure>

The federated parametric router reaches 0.75 normalized AUC against 0.63 to 0.72 for the
ten client-local routers. The clustering router shows the larger gap: 0.75 federated
against a client-local range of 0.55 to 0.70. These results are consistent with the
theoretical benefits of pooling query coverage and per-model supervision across clients.
Clustering quality degrades sharply when centers are fitted to a small, skewed sample,
and a client-local clustering router consequently has both less representative partitions
and sparse per-cell statistics. The federated method instead constructs global regions
before aggregating the statistics.

<h3>Improved routing on client-local distributions</h3>

An additional question is whether collaboration improves performance on the distribution a
client serves. Because the federated router is fitted across all clients, aggregation could
reduce specialization to an individual client. However, evaluation on each client's
local test set shows consistent improvements in this setting.

<figure>
  <img loading="lazy" decoding="async" width="1500" height="868" src="{{ '/papers/federated-routing/figures/local-test.png' | relative_url }}" alt="Six panels for three representative clients and both router families; the federated frontier sits above the client-local frontier in every panel">
  <figcaption>Federated against client-local routers on each client's own local test set,
  for three representative clients. The federated router improves the frontier
  in-distribution as well by increasing effective model coverage for each client.</figcaption>
</figure>

For the parametric router, clients 4, 6 and 8 improve from 0.70, 0.69 and 0.72 to 0.76,
0.71 and 0.76. For the clustering router the same clients improve from 0.71, 0.64 and
0.64 to 0.76, 0.72 and 0.77. The federated router attains higher AUC on all ten clients
for both families.

The improvement is primarily attributable to model coverage rather than query coverage. For
its local queries, a client already observes the relevant query distribution, but may lack
reliable estimates of how rarely invoked models would perform. Federation provides
supervision for these models through other clients, improving the *relative model
comparisons* required for routing.

<h3>Comparison with centralized training</h3>

To quantify the performance cost of decentralization, we compare against a router trained
directly on the pooled data.

<figure>
  <img loading="lazy" decoding="async" width="1500" height="973" src="{{ '/papers/federated-routing/figures/federated-vs-centralized.png' | relative_url }}" alt="Accuracy-cost curves for federated and centralized variants of both routers, essentially overlapping">
  <figcaption>Federated against centralized training on pooled data. Both router families
  reach 0.75 normalized AUC under both training schemes, matching at the reported
  precision.</figcaption>
</figure>

All four configurations achieve a normalized AUC of 0.75, indicating that federated
optimization closely matches centralized training in this experimental setting.

<h3>Adding models to the pool</h3>

Model pools evolve over time, making full retraining for every newly available model
costly. Three models are withheld during initial training and subsequently introduced,
with each client evaluating them on a small calibration subset of about 10% of its
prompts. The parametric router appends a head per new model and trains only that head,
leaving the trunk and existing heads frozen. The clustering router requires no parametric
training: the embedding space is unchanged, so only the per-cell statistics for the new
models are required.

<figure>
  <img loading="lazy" decoding="async" width="1300" height="717" src="{{ '/papers/federated-routing/figures/model-expansion.png' | relative_url }}" alt="Accuracy-cost frontiers before and after three withheld models are introduced; both routers improve after the calibration step">
  <figcaption>Global test frontiers before and after three withheld models join the pool
  and are incorporated through a lightweight calibration step.</figcaption>
</figure>

Without full retraining, normalized AUC increases from 0.732 to 0.748 for the parametric
router and to 0.749 for the clustering router.

<h3>Adding clients to the system</h3>

New clients may also join after deployment. A suitable adaptation mechanism should
incorporate their data, preserve performance for previously observed clients, and not
require the existing clients to participate again. Training starts with 7 clients, after
which 3 new ones join with data covering 30% of the task labels in the system.

<figure>
  <img loading="lazy" decoding="async" width="1500" height="594" src="{{ '/papers/federated-routing/figures/client-expansion.png' | relative_url }}" alt="Global test frontiers before and after three new clients join, for both router families; both improve">
  <figcaption>Adapting to newly joined clients without further participation from the
  original seven. The parametric router continues training on the new clients with a
  distillation penalty against its previous outputs; the clustering router simply updates
  its cluster statistics by weighted averaging.</figcaption>
</figure>

The parametric router improves from 0.71 to 0.74 and the clustering router from 0.72 to
0.74. The parametric method uses the distillation term to preserve the original policy
while training only on new clients; the clustering router preserves prior information
naturally, because incorporating a new client amounts to updating a weighted average.

<h3>Extreme heterogeneity and adaptive personalization</h3>

The preceding results use $\alpha = 0.6$, corresponding to moderate heterogeneity. Reducing
the Dirichlet concentration to $\alpha = 0.03$ produces nearly single-task clients. In this
regime, the global router may be mismatched to a particular client: if one client observes
almost only biology prompts while the others rarely observe this domain, a globally fitted
policy may underperform a specialized local router.

The proposed remedy is a per-client, per-model mixture of the federated and local
estimators. Each client measures the mean absolute calibration error of both estimators
on its own training points, requiring no additional model calls, and weights them
inversely to their errors, separately for accuracy and cost.

<figure>
  <img loading="lazy" decoding="async" width="1500" height="880" src="{{ '/papers/federated-routing/figures/personalization.png' | relative_url }}" alt="Six panels comparing client-local, federated and personalized frontiers under extreme heterogeneity; the personalized curve tracks the better of the other two">
  <figcaption>Local test frontiers under extreme heterogeneity ($\alpha = 0.03$),
  comparing client-local training, federated training and the adaptive mixture of the two.
  The mixture adapts toward the estimator with lower client-specific calibration error.</figcaption>
</figure>

Under this regime, the federated parametric router underperforms client-local training
on some clients: on clients 4 and 6 it reaches 0.69 and 0.71 against local scores of 0.70
and 0.73. The mixture mitigates this degradation, reaching 0.72 on both. On client 8, where
the federated router already performs better at 0.80, the mixture achieves 0.79.
The clustering router does not exhibit this degradation in the evaluated setting. Its local
variant is more strongly limited by sparse per-centroid coverage than it benefits from
specialization.

<h2 class="section">Summary</h2>

Most query-routing methods assume that training data can be centralized. Relaxing this
assumption changes the problem: evaluation data are distributed across clients, each client
sees a narrow slice of the query space, and because evaluation is expensive, each query
contains feedback from only one model selected non-uniformly. This work provides federated
training for both standard router families under those conditions, with a convergence
guarantee for the parametric router and suboptimality bounds for both that characterize the
benefit of pooling supervision across clients.

A central empirical finding is that federated training improves not only performance on
the global distribution but also performance on each client's *own* queries. In this
setting, a client's primary limitation is incomplete model coverage rather than
insufficient knowledge of its local query distribution. Cross-client aggregation supplies
the missing model supervision while keeping raw queries local.

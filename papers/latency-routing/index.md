---
layout: paper
permalink: /papers/latency-routing/
title: "Beyond Accuracy and Cost: Latency-Aware LLM Query Routing for Dynamic Workloads"
short_title: Latency-Aware Routing
description: >-
  A lightweight latency estimator that simulates the serving framework's batching and
  scheduling to predict time-to-first-token, coupled with a router that jointly optimises
  accuracy, cost, and latency. Up to 40% higher utility at latency comparable to standard
  load balancing.
image: /papers/latency-routing/figures/ontime-utility-qps.png
venue: arXiv
authors:
  - name: Shivam Patel
    url: /
    affil: 1
    me: true
    corresponding: true
  - name: Akaash R. Parthasarathy
    affil: 1
    corresponding: true
  - name: Ankur Mallick
    affil: 2
  - name: Gauri Joshi
    url: https://www.andrew.cmu.edu/user/gaurij/
    affil: 1
affiliations:
  - name: Carnegie Mellon University
    url: https://www.cmu.edu
  - name: Microsoft
    url: https://www.microsoft.com/research/
corresponding_note: Equal contribution
links:
  - name: arXiv
    url: https://arxiv.org/abs/2607.18253
  - name: PDF
    url: https://arxiv.org/pdf/2607.18253
  - name: Code
    url: https://github.com/akaashrp/sfs
tldr: >-
  Existing query routers select a language model by balancing response quality against
  monetary cost, but typically do not account for the latency experienced at model
  instances. Conversely, system-level policies such as round-robin and
  join-the-shortest-queue balance load without considering which model is most appropriate
  for a query. This separation can route requests to already-congested instances. The
  central challenge is latency estimation: generation latency depends not only on the
  prompt, but also on the resident prefill and decode workload and the serving framework's
  batching and scheduling policy. We introduce a **Serving Framework Simulation** (SFS)
  estimator that replays these policies from the current workload state and accumulates
  predicted token-batch times until the incoming query emits its first token. SFS achieves
  under 5% error on time-to-first-token with approximately $10^{-4}$ s of routing-time
  overhead. Incorporating these estimates into the routing objective yields up to
  **40% higher accuracy–cost utility at the latency of standard load balancing**, together
  with a 33% gain in area under the OnTimeUtility curve.
bibtex: |
  @article{patel2026beyond,
    title   = {Beyond Accuracy and Cost: Latency-Aware {LLM} Query Routing
               for Dynamic Workloads},
    author  = {Patel, Shivam and Parthasarathy, Akaash R. and
               Mallick, Ankur and Joshi, Gauri},
    journal = {arXiv preprint arXiv:2607.18253},
    year    = {2026},
    url     = {https://arxiv.org/abs/2607.18253}
  }
---

<h2 class="section">Bridging two sides of the routing decision</h2>

A language query router assigns each prompt to a model. Existing routing methods typically
focus on two quantities. *Accuracy* estimates the quality of the resulting response from
historical query–model evaluations, while *cost* is largely determined by token counts and
published prices. By sending easier queries to smaller, less expensive models and harder
queries to larger models, a heterogeneous model pool can achieve a better quality–cost
tradeoff than any single model in the pool.

What these methods generally do not model is *when the response arrives*. Most query
routers are latency-agnostic: they select the instance with the best predicted accuracy–cost
utility without accounting for its current workload. When similar queries repeatedly prefer
the same model instance, traffic can concentrate on that instance and produce substantial
queueing delays.

The systems literature addresses the complementary problem. Load-balancing policies across
replicas — such as round-robin and join-the-shortest-queue — and schedulers that prioritise
requests by deadline or predicted output length can effectively control latency, but they
operate *under a fixed query-to-model assignment*. They decide which replica should serve
a request, rather than which model should answer it, and are therefore accuracy- and
cost-agnostic by construction.

These two decisions are usually made independently. This work bridges them through
**a single routing decision that jointly optimises accuracy, cost, and latency.**

<h3>Why decoupled routing fails</h3>

The interactive example below considers a small deployment with one instance of each of
three Qwen3 models, a query stream drawn from four tasks with distinct prompt and response
length distributions, and the same arrival trace evaluated under four routing policies.
The accuracy scores and per-token prices match those used in the paper, so the routers
optimise the same accuracy–cost utility considered in our experiments.

The three instances are assigned **identical service capacity**. This is intentionally
not representative of the physical throughput differences between 0.6B and 32B models;
instead, it isolates the effect of the routing policy by ensuring that differences in
latency arise from query assignment rather than heterogeneous service rates. Under these
prices and quality scores, Qwen3-0.6B is never the utility-maximising model for any of the
four tasks.

{% include_relative router.part.html %}

The accuracy–cost router achieves high unconstrained utility but poor latency. It leaves the
0.6B instance unused and concentrates long-prompt summarisation traffic on the 8B instance,
whose queue grows steadily; by the end of the trace, tail latency is measured in seconds
despite targets of only hundreds of milliseconds. In contrast, Shortest Queue maintains low
latency by distributing requests across instances, but because it is unaware of query–model
utility, realised utility decreases substantially. SFS is the only policy in the example that
simultaneously maintains high utility and low latency. This tradeoff is captured by
**OnTimeUtility**: the realised accuracy–cost utility, with queries that violate their
latency targets assigned zero utility.

At sufficiently high arrival rates, all policies eventually saturate. This highlights an
important limitation: routing reallocates available capacity but cannot create additional
capacity. Above roughly 9 q/s in this illustrative three-instance deployment, the offered
load exceeds what the pool can sustain, and additional serving capacity is required.

<h2 class="section">Decomposing time-to-first-token</h2>

We optimise **time-to-first-token** (TTFT), defined as the interval between a query's
arrival and generation of its first output token. TTFT directly captures responsiveness for
interactive applications. For query $q\_i$ assigned to model instance $j$, it decomposes
into two components:

$$L^{\text{ttft}}_{i,j} \;=\; W_{i,j} \;+\; P_{i,j}$$

where $W\_{i,j}$ is the waiting time before the query's prefill computation begins, during
which it waits for compute and KV-cache memory occupied by earlier requests, and $P\_{i,j}$
is the time required to process its prompt and generate the first output token.

<figure>
  <img src="{{ '/papers/latency-routing/figures/ttft-decomposition.png' | relative_url }}" alt="Three snapshots of a model instance's token batches: a new sequence arriving, its prefill beginning, and its first decode token being generated">

  <figcaption>TTFT under continuous batching. A new sequence $q_i$ arrives at time $t$
  (<b>left</b>), its prefill computation begins at $t + W_{i,j}$ (<b>centre</b>), and its
  first decode token is generated at $t + W_{i,j} + P_{i,j}$ (<b>right</b>). Each column
  represents one token batch, which may combine at most one decode token per active sequence
  with prefill chunks from sequences whose prompts are still being processed.</figcaption>
</figure>

<h3>Why the serving framework matters</h3>

Autoregressive generation consists of two phases with different hardware characteristics.
**Prefill** processes prompt tokens in parallel to populate the KV cache and is typically
compute-bound. **Decode** generates output tokens sequentially, with each step reading
cached keys and values from prior tokens, and is typically memory-bound.

Modern serving frameworks interleave these phases rather than executing them as isolated
stages. Under *continuous batching* — used by systems such as ORCA and vLLM — new
prompts can be admitted at token-iteration boundaries instead of waiting for an entire
sequence-level batch to finish. vLLM's PagedAttention dynamically allocates KV-cache blocks
instead of reserving each sequence's maximum context length in advance. Sarathi-Serve's
*chunked prefill* further partitions long prompts and interleaves prefill chunks with
decode work, reducing stalls for sequences already generating tokens.

Consequently, a model instance cannot be described by a single fixed service rate. The
processing time of the next token batch depends on its *composition*: the number of
decoding sequences, their accumulated context lengths, the size of any scheduled prefill
chunk, and the position of that chunk within its prompt. The latency of a newly arriving
query therefore depends on the workload state it encounters and on how the serving framework
will construct subsequent token batches.

<h2 class="section">A throughput-based estimator and its limitations</h2>

A natural baseline profiles each instance's prefill throughput $\theta^{\text{pre}}\_j$
in tokens per second, sums the outstanding prefill tokens at the instance, and divides:

$$\widehat L_{i,j}^{\textrm{ttft}}(t)=
\underbrace{\frac{\sum_{q\in \mathcal{R}_j(t)} \textrm{tok}^{\textrm{pre}}_{q,j}(t)}{\theta^{\textrm{pre}}_j}}_{\text{waiting time}}
\;+\;
\underbrace{\frac{\textrm{tok}^{\textrm{pre}}_{q_i,j}(t)}{\theta^{\textrm{pre}}_j} + \bar T^{\textrm{dec}}_j}_{\text{prefill and first token}}$$

Here $\mathcal{R}\_j(t)$ denotes the *resident set* at instance $j$ — all queued and
actively served requests — while $\bar T^{\text{dec}}\_j$ is the average decode-batch
time used to approximate the computation of the first output token.

This estimator is simple and intuitive, but it introduces a systematic source of error. A
single throughput value is representative only of the batch composition on which it was
profiled. A throughput estimate obtained from prefill-dominated batches can substantially
underestimate latency when the live instance is simultaneously serving many long-context
decode sequences that compete for memory bandwidth. The resulting error is therefore not
merely random variation; it is workload-dependent bias that increases as the live batch
composition diverges from the profiling workload.

<figure class="fig-pair">
  <img src="{{ '/papers/latency-routing/figures/est-throughput.png' | relative_url }}" alt="Predicted versus actual TTFT for the prefill-throughput estimator, with points scattered far from the diagonal">
  <img src="{{ '/papers/latency-routing/figures/est-sfs.png' | relative_url }}" alt="Predicted versus actual TTFT for the SFS estimator, with points tight along the diagonal">

  <figcaption>Predicted versus measured TTFT on Qwen3-0.6B hosted on one H100, using the
  prefill-throughput estimator (<b>left</b>) and the proposed Serving Framework Simulation
  (<b>right</b>). By ignoring interference from decode work, the throughput estimator
  yields <b>85% mean absolute percentage error</b>, whereas SFS models the generation
  process and reduces the error to <b>5%</b>. The remaining outliers in both panels are
  attributable primarily to CPU and kernel overheads.</figcaption>
</figure>

<h2 class="section">Serving Framework Simulation</h2>

The key observation is that latency can be estimated by explicitly simulating how the serving
framework will schedule future token batches. **SFS** starts from the workload currently
resident at an instance, inserts the incoming query, and replays the framework's batching and
scheduling rules until that query produces its first decode token.

Formally, for a query $q\_i$ arriving at instance $j$ at time $t$, SFS constructs the
augmented workload

$$\mathcal{X}_{i,j}(t) = \left\{\left(\textrm{tok}^{\textrm{pre}}_{q,j}(t), \widehat{\textrm{tok}}^{\textrm{dec}}_{q,j}(t)\right) : q \in \mathcal{R}_j(t)\right\} \cup \left\{\left(\textrm{tok}^{\textrm{pre}}_{q_i,j}(t), \widehat{\textrm{tok}}^{\textrm{dec}}_{q_i,j}(t)\right)\right\}$$

pairing each resident request's *remaining* prefill tokens with its *predicted* remaining
decode tokens. From this state, SFS constructs successive token batches according to the
serving engine's policy — at most one decode token per active sequence, together with
prefill chunks that fit within the token budget — while respecting queue order, admission
limits, context limits, KV-cache allocation, and preemption. Let
$\ell^{\text{TTFT}}\_j(t,i)$ denote the index of the batch in which $q\_i$ emits its
first decode token. The TTFT estimate is the sum of predicted batch-processing times up to
and including that batch:

$$\widehat{L}^{\textrm{SFS}}_{i,j}(t) \;=\; \sum_{\ell=1}^{\ell^{\textrm{TTFT}}_j(t,i)} \widehat{T}^{(\ell)}_j(t)$$

Two properties keep this simulation lightweight. First, SFS terminates at the incoming
query's first token rather than simulating every resident request to completion, so the
simulation horizon is typically short. Second, TTFT estimation is relatively *robust to
decode-length error* because each active sequence contributes at most one decode token to
a token batch. The dominant factor is therefore how many sequences remain active during the
relevant horizon, rather than their exact eventual output lengths. Errors in predicted
response length primarily affect when a sequence departs, which often occurs beyond the
short horizon relevant to TTFT.

<h3>Estimating the processing time of a token batch</h3>

The remaining component is $\widehat{T}^{(\ell)}\_j(t)$, the predicted processing time of
one token batch. Rather than modelling individual kernels and memory transfers, which can be
fragile across workloads, we parameterise batch time using features of the batch composition.
For each sequence $q$ in batch $\ell$, let $c\_{q,j}(t,\ell)$ denote its context length
before the batch executes:

$$\begin{aligned}
\widehat T^{(\ell)}_j(t) \;=\;& \beta_{0,j}
\;+\; \beta_{1,j}\!\!\sum_{q\in \mathcal{B}^{(\ell)}_j(t)}\!\!\left(\textrm{tok}^{\textrm{pre}}_{q,j} + \textrm{tok}^{\textrm{dec}}_{q,j}\right)
\;+\; \beta_{2,j}\!\!\sum_{q\in \mathcal{B}^{(\ell)}_j(t)}\!\! c_{q,j}\,\textrm{tok}^{\textrm{dec}}_{q,j} \\[2pt]
&+\; \beta_{3,j}\!\!\sum_{q\in \mathcal{B}^{(\ell)}_j(t)}\!\!\left(\textrm{tok}^{\textrm{pre}}_{q,j}\, c_{q,j} + \frac{\textrm{tok}^{\textrm{pre}}_{q,j}\left(\textrm{tok}^{\textrm{pre}}_{q,j}+1\right)}{2}\right)
\end{aligned}$$

Each term has a direct interpretation. $\beta\_0$ captures fixed per-batch overhead.
$\beta\_1$ scales with the total number of processed tokens and represents dense-layer
computation incurred by both prefill and decode tokens. $\beta\_2$ models attention and
KV-cache reads during decode, which grow linearly with each sequence's accumulated context.
$\beta\_3$ models prefill attention: each prefill token attends to the existing context
and to preceding tokens within its own chunk, producing a chunk–context interaction term and
a quadratic within-chunk term. The four coefficients are calibrated independently for each
model instance from observed token-batch processing times.

The interactive panel below exposes this batch-time model directly. Holding the total token
count fixed while varying only the *context length* illustrates why identical token counts
can lead to different processing times.

{% include_relative batchtime.part.html %}

<figure style="max-width: 22rem; margin-left: auto; margin-right: auto;">
  <img src="{{ '/papers/latency-routing/figures/batch-time-accuracy.png' | relative_url }}" alt="Predicted versus observed token-batch processing time, tightly clustered on the diagonal">

  <figcaption>Predicted versus measured token-batch processing time on Qwen3-0.6B across
  the range of batch compositions encountered during serving. The estimator achieves
  <b>≈4% mean absolute percentage error</b>. Because SFS accumulates these batch-time
  predictions to estimate TTFT, their accuracy directly determines the fidelity of the
  resulting latency estimate.</figcaption>
</figure>

<h2 class="section">Incorporating latency into the routing objective</h2>

With latency estimates available, the routing objective can account for response quality,
monetary cost, and latency jointly. The accuracy–cost component follows the standard utility
formulation: for query $q\_i$ at instance $j$,

$$\widehat{U}_{i,j}(\lambda) \;=\; \widehat{\textrm{acc}}_{i,j} \;-\; \lambda\, \widehat{\textrm{cost}}_{i,j}$$

where $\lambda \ge 0$ controls the tradeoff between response quality and monetary cost.
Accuracy is predicted using a LightGBM regressor over inexpensive prompt-derived features:
hashed token vectors projected to 16 dimensions with PCA, together with token counts,
sentence counts, and task-type indicators. The regressor is trained on LLM-as-a-judge
scores. The same feature set is used for output-length prediction, which in turn supports
both cost estimation and the decode-length inputs required by SFS.

Latency is imposed as a **constraint**. Each query has a TTFT target $\tau\_i$;
the feasible set contains instances predicted to satisfy this target, and the router selects
the highest-utility feasible instance. If no instance is feasible, it falls back to the
instance with the smallest predicted latency:

$$m(i) \;\leftarrow\; \operatorname*{arg\,max}_{j\in\mathcal{J}:\; \widehat L^{\textrm{ttft}}_{i,j}(t)\,\le\, \tau_i} \widehat U_{i,j}(\lambda)$$

Evaluation requires a metric that accounts for whether utility is delivered within the
latency target. We therefore use **OnTimeUtility**, defined as mean realised utility
with latency violations assigned zero utility:

$$\textrm{OnTimeUtility}(\lambda) = \frac{1}{N}\sum_{i=1}^{N} U_{i,m(i)}(\lambda)\, \mathbf{1}\!\left\{L^{\textrm{ttft}}_{i,m(i)} \le \tau_i\right\}$$

A Lagrangian relaxation replaces the hard latency constraint with a penalty $\delta$ on
predicted latency, $\operatorname*{arg\,max}\_j \big(\widehat U\_{i,j}(\lambda) - \delta\,\widehat L^{\text{ttft}}\_{i,j}(t)\big)$.
Sweeping $\delta$ traces the utility–latency frontier. Setting $\delta = 0$ exactly recovers
latency-agnostic routing, so that baseline appears naturally as one operating point on the
same objective.

<h2 class="section">Experimental setup</h2>

We evaluate three Qwen3 models spanning a broad capability range: **Qwen3-0.6B** and
**Qwen3-8B** are each deployed on one H100, while **Qwen3-32B** is deployed
on two H100s using tensor parallelism. All experiments use vLLM; adapting SFS to another
serving framework requires replacing the corresponding batching and scheduling logic.
Generated responses are evaluated with LLM-as-a-judge using Gemini 3.1 Pro Preview.

Queries are sampled from four tasks selected to span the prompt-length and response-length
space: **Alpaca** (short prompt, short response), **HotpotQA** (long prompt,
short response), **GovReport-Summarization** (long prompt, long response), and
**WritingPrompts** (short prompt, long response). Prompt lengths range from roughly
$10$ to $10^4$ tokens and response lengths from $10^2$ to $10^3$ tokens, producing a
heterogeneous serving workload. The main experiments use Poisson arrivals, with a
Markov-modulated Poisson process used to evaluate robustness to bursty traffic.

<figure class="fig-pair">
  <img src="{{ '/papers/latency-routing/figures/input-length-dist.png' | relative_url }}" alt="Prompt token length distributions for the four tasks, spanning about ten to ten thousand tokens">
  <img src="{{ '/papers/latency-routing/figures/response-length-dist.png' | relative_url }}" alt="Response token length distributions per model and task, spanning hundreds to about a thousand tokens">

  <figcaption>Workload length distributions. Prompt lengths (<b>left</b>) vary by roughly
  three orders of magnitude across tasks, while response lengths (<b>right</b>) depend on
  both the task and the model. This heterogeneity is important for latency estimation: a
  single tokens-per-second statistic cannot accurately characterise an instance processing
  a short Alpaca prompt and a multi-thousand-token GovReport prompt within the same serving
  workload.</figcaption>
</figure>

<div class="table-wrap">
<table>
  <caption>Per-task response-quality scores (LLM-as-a-judge, %) and per-token prices.
  These values create meaningful routing tradeoffs across the model pool: the quality gap
  between Qwen3-0.6B and Qwen3-32B is substantially larger on WritingPrompts than on Alpaca,
  while Qwen3-32B costs roughly 3.7× more per output token than Qwen3-8B. Thus, no single
  model dominates across both response quality and monetary cost.</caption>
  <thead>
    <tr>
      <th>Task</th>
      <th>Qwen3-0.6B</th><th>Qwen3-8B</th><th>Qwen3-32B</th>
    </tr>
  </thead>
  <tbody>
    <tr><td>Alpaca</td><td>53.49</td><td>82.67</td><td class="ours">88.85</td></tr>
    <tr><td>GovReport-Summarization</td><td>29.02</td><td>86.62</td><td class="ours">95.93</td></tr>
    <tr><td>HotpotQA (distractor)</td><td>40.98</td><td>88.28</td><td class="ours">92.69</td></tr>
    <tr><td>WritingPrompts</td><td>17.73</td><td>57.47</td><td class="ours">80.69</td></tr>
    <tr><td class="rowlab">Aggregate</td><td>35.31</td><td>78.76</td><td class="ours">89.54</td></tr>
    <tr><td class="rowlab">Prompt price, USD per M tokens</td><td>0.044</td><td>0.072</td><td>0.287</td></tr>
    <tr><td class="rowlab">Response price, USD per M tokens</td><td>0.173</td><td>0.287</td><td>0.640</td></tr>
  </tbody>
</table>
</div>

TTFT targets are generated as a linear function of prompt length with small random variation,
$\tau\_i \approx 158 + 3.5\times10^{-3} p\_i$ ms, clipped to $[150, 1120]$ ms. The
resulting targets are stringent enough to expose congestion-induced violations while
remaining achievable for lightly loaded instances.

The baselines represent the two conventional sides of the problem. **Round Robin** and
**Shortest Queue** provide load balancing without modelling response quality or cost,
whereas **Latency-Agnostic** routing maximises utility without observing instance
workloads.

<h2 class="section">Results</h2>

<h3>Performance under increasing offered load</h3>

<figure class="fig-pair">
  <img src="{{ '/papers/latency-routing/figures/ontime-utility-qps.png' | relative_url }}" alt="OnTimeUtility against queries per second: SFS highest across the range, latency-agnostic falling steeply">
  <img src="{{ '/papers/latency-routing/figures/slo-attainment.png' | relative_url }}" alt="Latency constraint attainment against queries per second, SFS remaining high">

  <figcaption><b>Left:</b> OnTimeUtility as the offered load increases. SFS achieves the
  highest OnTimeUtility at every evaluated arrival rate, with a <b>33% improvement in area
  under the curve</b> over the best baseline and a <b>46% improvement</b> at 5 q/s.
  Latency-Agnostic routing is competitive at low load but degrades as its preferred model
  instances saturate; Shortest Queue maintains low latency but cannot exploit query-specific
  model utility. <b>Right:</b> SFS maintains higher latency-constraint attainment as load
  increases, allowing a larger fraction of predicted utility to be realised on time.</figcaption>
</figure>

The behaviour of Latency-Agnostic routing illustrates why latency must be included in the
model-selection decision. At low load, when congestion is negligible, selecting the
highest-utility instance is appropriate. As load increases, however, the same policy
concentrates requests on preferred instances and loses its utility advantage through
latency violations.

<h3>Controlling the utility–latency tradeoff</h3>

<figure>
  <img src="{{ '/papers/latency-routing/figures/utility-latency-tradeoff.png' | relative_url }}" alt="Utility against average time-to-first-token on a log axis, with the SFS curve spanning from 0.1 to 200 seconds and baselines as isolated points">

  <figcaption>Sweeping the latency penalty $\delta$ produces a controllable utility–latency
  frontier at 8 q/s. The horizontal axis is logarithmic and spans three orders of magnitude.
  The latency-agnostic router, corresponding to $\delta = 0$, attains its highest utility at
  an average TTFT of <b>over 200 seconds</b>. At the low-latency end, SFS achieves
  <b>40% higher utility</b> than Shortest Queue at comparable TTFT, while Round Robin lies
  below the SFS frontier in both utility and latency.</figcaption>
</figure>

This result highlights an important advantage of the joint objective. The baseline policies
correspond to fixed operating points, whereas latency-aware routing exposes a continuum of
utility–latency tradeoffs controlled by $\delta$.

<h3>Routing composition by task</h3>

<figure>
  <img src="{{ '/papers/latency-routing/figures/routing-composition.png' | relative_url }}" alt="Stacked bars of routing composition per task for four policies; SFS varies by task while shortest queue and round robin are identical across tasks">

  <figcaption>Fraction of queries assigned to each model instance by task at 8 q/s.
  <b>Shortest Queue</b> and <b>Round Robin</b> produce nearly identical routing
  compositions across tasks, reflecting their lack of query-specific model preferences.
  <b>Latency-Agnostic</b> routing is task-dependent but highly concentrated: it sends
  almost no requests to Qwen3-0.6B and assigns entire tasks predominantly to a single model.
  <b>SFS</b> remains task-dependent while adapting to congestion, routing approximately
  37% of long-prompt GovReport requests to the smaller model when larger instances are
  congested, while continuing to favour Qwen3-32B for WritingPrompts, where its
  response-quality advantage is largest.</figcaption>
</figure>

<h3>Bursty arrivals and estimator overhead</h3>

<figure class="fig-pair">
  <img src="{{ '/papers/latency-routing/figures/bursty-arrivals.png' | relative_url }}" alt="OnTimeUtility under Markov-modulated Poisson arrivals at two burstiness ratios">
  <img src="{{ '/papers/latency-routing/figures/estimator-overhead.png' | relative_url }}" alt="Histogram of SFS estimator wall-clock latency, centred near 0.1 milliseconds">

  <figcaption><b>Left:</b> Robustness to correlated arrivals using a two-state
  Markov-modulated Poisson process with high-to-low arrival-rate ratios $r = 3$ and $6$.
  The relative ordering of routing policies remains unchanged. <b>Right:</b> Wall-clock
  overhead of SFS, which lies on the routing critical path. Mean simulation time is
  approximately $10^{-4}$&nbsp;s, three to four orders of magnitude smaller than the TTFTs
  being predicted.</figcaption>
</figure>

Low estimator overhead also depends on the system implementation. After each token iteration,
every model instance publishes a compact workload snapshot to shared memory. Snapshots are
marked in-progress or complete so that the router reads only consistent state without
stalling the serving engine. The router retains the latest complete snapshot from each
instance and simulates forward from that state. The batching simulation is approximately
300 lines of C++ within a ~3.6K-line Python/C++ implementation; porting SFS to another
serving framework primarily requires adapting this scheduling simulation.

<h2 class="section">Average-case estimation without workload visibility</h2>

SFS assumes access to real-time workload snapshots. This information may be unavailable when,
for example, a router sends requests to third-party cloud endpoints. In this setting, the
paper develops an average-case approximation based on a **Limited Processor Sharing**
(LPS) queue. Up to $k$ queries are served concurrently and share compute capacity, while
additional requests wait in FCFS order. This abstraction better reflects autoregressive
serving than a single-server queue because concurrency is explicit but bounded, and
per-request token generation slows as more sequences share the GPU. With arrival rate
$\alpha\_j$, service capacity $\mu\_j$, and utilisation
$\rho\_j = \alpha\_j/\mu\_j < 1$, the expected waiting time is

$$\widehat W^{\textrm{avg}}_{i,j} = \frac{(\alpha_j/\mu_j)^k}{\mu_j-\alpha_j}$$

<figure class="fig-pair">
  <img src="{{ '/papers/latency-routing/figures/lps-scheme.png' | relative_url }}" alt="Diagram of the limited processor sharing scheme with k concurrent servers and a FCFS queue behind them">
  <img src="{{ '/papers/latency-routing/figures/lps-delay-vs-load.png' | relative_url }}" alt="Normalised delay against load, measured against the theoretical LPS curve">

  <figcaption>The Limited Processor Sharing abstraction (<b>left</b>) and its empirical
  fit (<b>right</b>). Measured TTFT on Qwen3-0.6B follows the qualitative growth of LPS
  waiting time as load increases; the observed time-average parallelism in this deployment
  is <b>k ≈ 25</b>. The approximation is necessarily coarser than SFS because it uses
  time-averaged rates rather than the instantaneous workload state, but it requires no
  visibility into the serving framework.</figcaption>
</figure>

<h2 class="section">Scope and limitations</h2>

The central claim is deliberately focused: latency should be incorporated into the routing
objective rather than delegated entirely to a lower systems layer, and the required latency
estimates can be obtained efficiently by simulating the serving framework instead of
compressing its state into a single throughput statistic. In our experiments, joint
accuracy–cost–latency routing achieves over 40% higher utility than standard load balancing
at comparable latency.

Several directions remain open. First, our primary objective optimises TTFT, which is
appropriate for interactive responsiveness but does not fully characterise workloads whose
quality of service depends on completion time. End-to-end latency is more sensitive to
decode-length prediction error because the full generation horizon must be modelled. Second,
model placement is fixed in our evaluation; jointly optimising routing decisions and model
placement across heterogeneous hardware is a natural extension. Finally, SFS assumes that
the workload state of each candidate instance can be represented consistently. With multiple
independent routers sharing the same instance pool, each router perturbs the state being
simulated by the others, creating an additional coordination problem that is not modelled in
the current system.

<p class="lr-note" style="margin-top:2rem">
The simulation on this page is a re-implementation for exposition. It uses the same batching
policy and batch-time model as the paper, but illustrative coefficients and an intentionally
equalised instance pool. Its purpose is to isolate and visualise the routing-induced failure
mode rather than reproduce the measured experimental results above. The released system is
available at
<a href="https://github.com/akaashrp/sfs">github.com/akaashrp/sfs</a>.
</p>

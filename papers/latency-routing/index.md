---
layout: paper
permalink: /papers/latency-routing/
title: "Beyond Accuracy and Cost: Latency-Aware LLM Query Routing for Dynamic Workloads"
short_title: Latency-Aware Routing
description: >-
  A lightweight latency estimator that simulates the serving framework's own batching and
  scheduling to predict time-to-first-token, and a router that optimises accuracy, cost
  and latency together. Up to 40% higher utility at the latency of standard load balancing.
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
  Query routers pick a language model for each query by trading response quality against
  monetary cost, and they do it without any notion of how long the answer will take.
  Latency is left to the systems layer, where round-robin and join-the-shortest-queue
  balance load without any notion of which model a query actually needs. Neither half can
  see the other, and the result is a router that cheerfully posts queries to instances
  that are already saturated. The obstacle is estimation: generation latency depends not
  on the prompt alone but on the prefill and decode work resident at an instance and on
  the batching policy of the serving framework. We build a **Serving Framework Simulation**
  (SFS) estimator that replays the framework's own batching and scheduling forward from
  the current workload and accumulates predicted token-batch times until the new query
  emits its first token — under 5% error on time-to-first-token, at roughly $10^{-4}$ s of
  routing-time overhead. Folding those estimates into the routing objective gives up to
  **40% higher accuracy–cost utility at the latency of standard load balancing**, and a
  33% gain in area under the OnTimeUtility curve.
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

<h2 class="section">Two halves of one decision</h2>

A language query router takes a prompt and picks a model to answer it. The literature on
that choice is almost entirely about two quantities. *Accuracy*: how good the answer will
be, predicted from historical evaluations of queries on models. *Cost*: what the answer
will be charged at, a near-deterministic function of token counts and published prices.
Route easy queries to small cheap models, hard queries to large expensive ones, and a
pool of heterogeneous models delivers more quality per dollar than any single model in it.

What that literature does not model is *when the answer arrives*. Routers are latency-agnostic.
They will happily post a query to the instance with the best accuracy–cost score
irrespective of the queue already standing in front of it, and because the highest-utility
instance is the same instance for every similar query, the traffic concentrates and the
queue grows without bound.

The systems literature has the complementary blind spot. Load balancing across replicas —
round-robin, join-the-shortest-queue — and the scheduling work that prioritises requests by
deadline or predicted output length are all effective at controlling latency, and all of
them operate *under a fixed query-to-model assignment*. They decide which replica, never
which model. They are accuracy- and cost-agnostic by construction.

So the deployment is run by two policies that cannot see each other. This work closes that
gap: **a single routing decision that optimises accuracy, cost and latency jointly.**

<h3>The failure is not subtle</h3>

The demonstration below is a small deployment: one instance of each of the three Qwen3
models, a stream of queries drawn from four tasks with genuinely different prompt and
response lengths, and the same arrival trace fed to four different routers. Accuracy scores and per-token prices are the
paper's, so the accuracy–cost utility each router is chasing is the real one.

The three instances are given **identical service capacity**, which is not physically true
of a 0.6B and a 32B model, but it isolates the effect we care about: every difference in
latency you see is produced by the routing decision and nothing else. Under those prices and scores
Qwen3-0.6B is never the utility-maximising choice for any of the four tasks — so watch what
the accuracy–cost router does with it.

{% include_relative router.part.html %}

The accuracy–cost router wins the utility column and loses everything else. It leaves the
0.6B instance idle for the entire run — not almost idle, but at zero — and pushes the heavy,
long-prompt summarisation traffic onto the 8B instance, whose queue then grows
monotonically; by the end of the trace its tail latency is measured in seconds against
targets measured in hundreds of milliseconds. A third of the pool is doing nothing while
another third drowns. Shortest-queue inverts the failure — latency stays flat, and the router
spreads queries across models with no idea which one suits them, so realised utility falls
by a fifth. SFS is the only policy in the table that is near the top of both columns, and
the metric that captures this is **OnTimeUtility**: the accuracy–cost utility actually
realised, counting a query that misses its latency target as worth nothing.

Turn the arrival rate up far enough and every policy collapses together. That is honest and
worth seeing: routing allocates capacity, it does not create it. Somewhere above 9 q/s
this three-instance pool is simply too small for the offered load, and the answer there is
more hardware, not a better router.

<h2 class="section">What time-to-first-token is made of</h2>

We route on **time-to-first-token** (TTFT) — the interval from a query arriving in the
system to its first output token — because it is what an interactive application and a
human reader actually perceive as responsiveness. For query $q\_i$ at model instance $j$ it
decomposes into two pieces:

$$L^{\text{ttft}}_{i,j} \;=\; W_{i,j} \;+\; P_{i,j}$$

where $W\_{i,j}$ is the waiting time before the query's prefill computation can begin — it
is holding for compute and for KV-cache memory still occupied by earlier queries — and
$P\_{i,j}$ is the time to process its prompt and produce the first token.

<figure>
  <img src="{{ '/papers/latency-routing/figures/ttft-decomposition.png' | relative_url }}" alt="Three snapshots of a model instance's token batches: a new sequence arriving, its prefill beginning, and its first decode token being generated">

  <figcaption>TTFT under continuous batching. A new sequence $q_i$ arrives at time $t$
  (<b>left</b>), its prefill computation begins at $t + W_{i,j}$ (<b>centre</b>), and its
  first decode token is generated at $t + W_{i,j} + P_{i,j}$ (<b>right</b>). Each column is
  one token batch; a batch mixes at most one decode token per active sequence with prefill
  chunks belonging to sequences still processing their prompts.</figcaption>
</figure>

<h3>Why the serving framework is part of the problem</h3>

Generation splits into two phases with opposite hardware characteristics. **Prefill**
processes all prompt tokens in parallel to populate the KV cache and is compute-bound.
**Decode** emits output tokens one at a time, each step reading the cached keys and values
for every previous token, and is memory-bound.

Modern serving frameworks do not run these phases as separate stages. Under *continuous
batching* — ORCA, vLLM — new prompts are admitted at token-iteration boundaries rather than
waiting for a whole sequence-level batch to drain, so a new sequence can begin generating
while others are mid-response. vLLM's PagedAttention allocates KV-cache blocks dynamically
instead of reserving each sequence's maximum context up front, and Sarathi-Serve's *chunked
prefill* splits a long prompt into pieces and interleaves them with decode work so that a
large prompt does not stall every sequence currently generating.

The consequence for a router is that a model instance has no single service rate. What
determines how long the next token batch takes is the *composition* of that batch — how
many sequences are decoding, how much context each has accumulated, how large a prefill
chunk got scheduled alongside them, and how far into its prompt that chunk sits. A query's
latency therefore depends on the workload it happens to arrive into, and predicting it means
predicting how the framework will assemble the next few hundred batches.

<h2 class="section">A first attempt, and why it fails</h2>

The obvious estimator profiles each instance's prefill throughput $\theta^{\text{pre}}\_j$
in tokens per second, adds up the prefill tokens outstanding at the instance, and divides:

$$\widehat L_{i,j}^{\textrm{ttft}}(t)=
\underbrace{\frac{\sum_{q\in \mathcal{R}_j(t)} \textrm{tok}^{\textrm{pre}}_{q,j}(t)}{\theta^{\textrm{pre}}_j}}_{\text{waiting time}}
\;+\;
\underbrace{\frac{\textrm{tok}^{\textrm{pre}}_{q_i,j}(t)}{\theta^{\textrm{pre}}_j} + \bar T^{\textrm{dec}}_j}_{\text{prefill and first token}}$$

Here $\mathcal{R}\_j(t)$ is the *resident set* at instance $j$ — everything queued plus
everything actively being served — and $\bar T^{\text{dec}}\_j$ is the average decode-batch
time, standing in for the first token's own computation.

This is a reasonable first cut, and it is wrong in a specific and instructive way. A single
throughput number is only valid for the batch composition it was profiled on. Profile it on
prefill-dominated batches and it will badly underestimate what happens when the instance is
also carrying twenty long-context decode sequences competing for memory bandwidth. The error
is not noise; it is bias that grows with the mismatch between the profiling workload and the
live one.

<figure class="fig-pair">
  <img src="{{ '/papers/latency-routing/figures/est-throughput.png' | relative_url }}" alt="Predicted versus actual TTFT for the prefill-throughput estimator, with points scattered far from the diagonal">
  <img src="{{ '/papers/latency-routing/figures/est-sfs.png' | relative_url }}" alt="Predicted versus actual TTFT for the SFS estimator, with points tight along the diagonal">

  <figcaption>Predicted against measured TTFT on Qwen3-0.6B hosted on one H100, for the
  prefill-throughput estimator (<b>left</b>) and the proposed Serving Framework Simulation
  (<b>right</b>). The throughput estimator ignores decode interference and lands at
  <b>85% mean absolute percentage error</b>; SFS simulates the generation process and
  reaches <b>5%</b>. The scattered outliers in both panels are CPU and kernel
  overheads.</figcaption>
</figure>

<h2 class="section">Serving Framework Simulation</h2>

If the difficulty is that latency depends on how the framework will schedule the next few
hundred batches, the direct response is to work that out. **SFS** takes the workload
resident at an instance, adds the new query to it, and replays the framework's own batching
and scheduling rules forward — stopping the moment the new query produces its first decode
token.

Concretely, for a query $q\_i$ arriving at instance $j$ at time $t$, SFS assembles the
augmented workload

$$\mathcal{X}_{i,j}(t) = \left\{\left(\textrm{tok}^{\textrm{pre}}_{q,j}(t), \widehat{\textrm{tok}}^{\textrm{dec}}_{q,j}(t)\right) : q \in \mathcal{R}_j(t)\right\} \cup \left\{\left(\textrm{tok}^{\textrm{pre}}_{q_i,j}(t), \widehat{\textrm{tok}}^{\textrm{dec}}_{q_i,j}(t)\right)\right\}$$

pairing each resident request's *remaining* prefill tokens with its *predicted* remaining
decode tokens. From this state it constructs successive token batches exactly as the engine
would — at most one decode token per active sequence, plus whatever prefill chunks fit in
the token budget — respecting queue order, admission limits, context limits, KV-cache
allocation and preemption. Writing $\ell^{\text{TTFT}}\_j(t,i)$ for the index of the batch
in which $q\_i$ emits its first decode token, the estimate is simply the accumulated batch
time up to that point:

$$\widehat{L}^{\textrm{SFS}}_{i,j}(t) \;=\; \sum_{\ell=1}^{\ell^{\textrm{TTFT}}_j(t,i)} \widehat{T}^{(\ell)}_j(t)$$

Two properties make this cheap. It never simulates to completion — only to the new query's
first token, which is typically a few dozen batches away. And it is *robust to decode-length
error*, because each active sequence contributes at most one decode token per batch: what
matters for TTFT is how many sequences are decoding, not exactly how much decoding each has
left. A mis-predicted response length changes when a sequence eventually leaves, which is
mostly beyond the horizon that TTFT cares about.

<h3>Pricing a single token batch</h3>

That leaves $\widehat{T}^{(\ell)}\_j(t)$, the predicted time for one token batch. Modelling
individual kernels and memory transfers is fragile and does not transfer across workloads,
so the estimator works at the level of batch composition. For each sequence $q$ in batch
$\ell$, let $c\_{q,j}(t,\ell)$ be its context length before the batch runs:

$$\begin{aligned}
\widehat T^{(\ell)}_j(t) \;=\;& \beta_{0,j}
\;+\; \beta_{1,j}\!\!\sum_{q\in \mathcal{B}^{(\ell)}_j(t)}\!\!\left(\textrm{tok}^{\textrm{pre}}_{q,j} + \textrm{tok}^{\textrm{dec}}_{q,j}\right)
\;+\; \beta_{2,j}\!\!\sum_{q\in \mathcal{B}^{(\ell)}_j(t)}\!\! c_{q,j}\,\textrm{tok}^{\textrm{dec}}_{q,j} \\[2pt]
&+\; \beta_{3,j}\!\!\sum_{q\in \mathcal{B}^{(\ell)}_j(t)}\!\!\left(\textrm{tok}^{\textrm{pre}}_{q,j}\, c_{q,j} + \frac{\textrm{tok}^{\textrm{pre}}_{q,j}\left(\textrm{tok}^{\textrm{pre}}_{q,j}+1\right)}{2}\right)
\end{aligned}$$

Each term corresponds to something physical. $\beta\_0$ is fixed per-batch overhead.
$\beta\_1$ scales with the total token count and captures dense-layer work, which every
token pays regardless of type. $\beta\_2$ is attention and KV-cache reads for decode
sequences, growing linearly in each sequence's accumulated context. $\beta\_3$ is prefill
attention: each prefill token attends both to the existing context and to the tokens ahead
of it inside its own chunk, giving a chunk×context product plus a quadratic within-chunk
term. The four coefficients are calibrated per instance from observed batch times.

The panel below is that expression, made manipulable. The instructive move is to hold the
token count fixed and change only the *context*.

{% include_relative batchtime.part.html %}

<figure style="max-width: 22rem; margin-left: auto; margin-right: auto;">
  <img src="{{ '/papers/latency-routing/figures/batch-time-accuracy.png' | relative_url }}" alt="Predicted versus observed token-batch processing time, tightly clustered on the diagonal">

  <figcaption>The token-batch estimator against measured batch times on Qwen3-0.6B,
  across the range of batch compositions that arise in serving —
  <b>≈4% mean absolute percentage error</b>. This is the primitive the whole simulation is
  built out of, so its error is what ultimately bounds the TTFT estimate.</figcaption>
</figure>

<h2 class="section">Putting latency into the objective</h2>

With estimates in hand, the routing objective can finally hold all three quantities. The
accuracy–cost part is standard: for query $q\_i$ at instance $j$,

$$\widehat{U}_{i,j}(\lambda) \;=\; \widehat{\textrm{acc}}_{i,j} \;-\; \lambda\, \widehat{\textrm{cost}}_{i,j}$$

with $\lambda \ge 0$ setting the exchange rate between quality and money. Accuracy is
predicted by a LightGBM regressor over cheap prompt-derived features — hashed token
vectors projected to 16 dimensions by PCA, plus token and sentence counts and task-type
indicators — trained on LLM-as-a-judge scores. The same features drive an output-length
predictor, which supplies both the cost estimate and the decode lengths SFS needs.

Latency enters as a **constraint**. Each query carries a TTFT target $\tau\_i$; the feasible
set is the instances predicted to meet it, and the router takes the best utility among
those, falling back to the fastest instance when nothing is feasible:

$$m(i) \;\leftarrow\; \operatorname*{arg\,max}_{j\in\mathcal{J}:\; \widehat L^{\textrm{ttft}}_{i,j}(t)\,\le\, \tau_i} \widehat U_{i,j}(\lambda)$$

Evaluating this needs a metric that refuses to reward a good answer that arrived too late,
which is **OnTimeUtility** — mean realised utility with violations scored zero:

$$\textrm{OnTimeUtility}(\lambda) = \frac{1}{N}\sum_{i=1}^{N} U_{i,m(i)}(\lambda)\, \mathbf{1}\!\left\{L^{\textrm{ttft}}_{i,m(i)} \le \tau_i\right\}$$

A Lagrangian relaxation replaces the hard constraint with a price $\delta$ on predicted
latency, $\operatorname*{arg\,max}\_j \big(\widehat U\_{i,j}(\lambda) - \delta\,\widehat L^{\text{ttft}}\_{i,j}(t)\big)$,
and sweeping $\delta$ traces out the utility–latency frontier. Setting $\delta = 0$ recovers
latency-agnostic routing exactly, which makes the baseline a point on our own curve rather
than a separate method.

<h2 class="section">Experimental setup</h2>

Three models from the Qwen3 family span the capability range: **Qwen3-0.6B** and
**Qwen3-8B** on one H100 each, and **Qwen3-32B** on two H100s with tensor parallelism.
Serving is vLLM throughout, though the estimator only needs the batching logic swapped to
target another framework. Responses are scored by LLM-as-a-judge using Gemini 3.1 Pro
Preview.

Queries are drawn from four tasks chosen to cover the corners of the (prompt length, response
length) space: **Alpaca** (short, short), **HotpotQA** (long, short),
**GovReport-Summarization** (long, long) and **WritingPrompts** (short, long). Prompt
lengths span roughly $10$ to $10^4$ tokens and responses $10^2$ to $10^3$, so the offered
workload is genuinely heterogeneous rather than a stream of interchangeable requests.
Arrivals are Poisson, with a Markov-modulated Poisson process used to test burstiness.

<figure class="fig-pair">
  <img src="{{ '/papers/latency-routing/figures/input-length-dist.png' | relative_url }}" alt="Prompt token length distributions for the four tasks, spanning about ten to ten thousand tokens">
  <img src="{{ '/papers/latency-routing/figures/response-length-dist.png' | relative_url }}" alt="Response token length distributions per model and task, spanning hundreds to about a thousand tokens">

  <figcaption>The offered workload. Prompt lengths (<b>left</b>) differ by three orders of
  magnitude across tasks, and response lengths (<b>right</b>) vary with both the task and
  the model answering it. This spread is what makes latency estimation hard: a single
  tokens-per-second figure cannot describe an instance serving a 30-token Alpaca prompt and
  an 8,000-token GovReport prompt in the same token batch.</figcaption>
</figure>

<div class="table-wrap">
<table>
  <caption>Per-task quality scores (LLM-as-a-judge, %) and per-token prices. Together these
  are what make the model pool worth routing over: the quality gap between 0.6B and 32B is
  far larger on WritingPrompts than on Alpaca, while 32B costs roughly 3.7× more per output
  token than 8B. No single column dominates once price is counted.</caption>
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

TTFT targets are set as a linear function of prompt length with a little jitter,
$\tau\_i \approx 158 + 3.5\times10^{-3} p\_i$ ms, clipped to $[150, 1120]$ ms — tight enough
that a saturated instance violates them, loose enough that an idle one does not.

The baselines are the two halves of the problem. **Round Robin** and **Shortest Queue**
represent latency-aware, accuracy-agnostic load balancing; **Latency-Agnostic** is
utility-maximising routing with no view of the queues.

<h2 class="section">Results</h2>

<h3>Varying offered load</h3>

<figure class="fig-pair">
  <img src="{{ '/papers/latency-routing/figures/ontime-utility-qps.png' | relative_url }}" alt="OnTimeUtility against queries per second: SFS highest across the range, latency-agnostic falling steeply">
  <img src="{{ '/papers/latency-routing/figures/slo-attainment.png' | relative_url }}" alt="Latency constraint attainment against queries per second, SFS remaining high">

  <figcaption><b>Left:</b> OnTimeUtility as offered load rises. SFS is highest at every
  arrival rate, worth a <b>33% gain in area under the curve</b> over the best baseline and
  <b>46%</b> at 5 q/s. Latency-agnostic routing starts competitive and falls off a cliff as
  its preferred instances saturate; shortest-queue is flat but low, because it never learns
  which model a query needs. <b>Right:</b> the mechanism — SFS keeps latency-constraint
  attainment high, which is what converts its utility into <i>on-time</i> utility.</figcaption>
</figure>

The shape of the latency-agnostic curve is the entire argument for this line of work. At
low load it is the best policy on the plot, because when nothing is congested the
highest-utility instance really is the right answer. Its advantage evaporates precisely
when routing starts to matter.

<h3>Trading utility against latency on purpose</h3>

<figure>
  <img src="{{ '/papers/latency-routing/figures/utility-latency-tradeoff.png' | relative_url }}" alt="Utility against average time-to-first-token on a log axis, with the SFS curve spanning from 0.1 to 200 seconds and baselines as isolated points">

  <figcaption>Sweeping the latency price $\delta$ traces a controllable frontier at 8 q/s.
  Note the horizontal axis is logarithmic and spans three orders of magnitude: the
  latency-agnostic router (which is our own $\delta = 0$) buys its top utility at an average
  TTFT of <b>over 200 seconds</b>. At the low-latency end SFS attains <b>40% higher
  utility</b> than Shortest Queue at comparable TTFT. Round Robin sits below the frontier
  on both axes.</figcaption>
</figure>

This plot is the cleanest statement of the contribution. The baselines are single points —
each is one fixed policy, and if its operating point is not the one you need, there is no
knob. Latency-aware routing is a curve, and $\delta$ is the knob.

<h3>Where the queries actually go</h3>

<figure>
  <img src="{{ '/papers/latency-routing/figures/routing-composition.png' | relative_url }}" alt="Stacked bars of routing composition per task for four policies; SFS varies by task while shortest queue and round robin are identical across tasks">

  <figcaption>Fraction of queries sent to each model instance, broken down by task, at
  8 q/s. <b>Shortest Queue</b> and <b>Round Robin</b> produce near-identical compositions
  for every task — the signature of a policy that cannot see the query. <b>Latency-Agnostic</b>
  is task-dependent but degenerate: it sends essentially <i>nothing</i> to Qwen3-0.6B and
  pins whole tasks to a single model. <b>SFS</b> is task-dependent and graded, spilling
  around 37% of the long-prompt GovReport traffic onto the small model when the larger ones
  are congested, while still buying Qwen3-32B for WritingPrompts where its quality margin is
  widest.</figcaption>
</figure>

<h3>Bursty arrivals, and what the estimator costs</h3>

<figure class="fig-pair">
  <img src="{{ '/papers/latency-routing/figures/bursty-arrivals.png' | relative_url }}" alt="OnTimeUtility under Markov-modulated Poisson arrivals at two burstiness ratios">
  <img src="{{ '/papers/latency-routing/figures/estimator-overhead.png' | relative_url }}" alt="Histogram of SFS estimator wall-clock latency, centred near 0.1 milliseconds">

  <figcaption><b>Left:</b> real traffic is correlated, so arrivals are re-drawn from a
  two-state Markov-modulated Poisson process with high-to-low rate ratios $r = 3$ and $6$.
  The ordering is unchanged. <b>Right:</b> the estimator sits on the critical path of every
  routing decision, so its own cost matters — mean wall-clock time is about
  $10^{-4}$&nbsp;s, three to four orders of magnitude below the TTFTs being
  predicted.</figcaption>
</figure>

That overhead is a result about implementation as much as about algorithms. Each instance
publishes a compact workload snapshot to shared memory after every token iteration, marked
in-progress or complete so the router never reads a half-written state and never has to
stall the engine to get one; the router keeps the last complete snapshot from each instance
and simulates from it. The batching simulation itself is roughly 300 lines of C++ inside a
~3.6K-line Python and C++ system, and porting the estimator to a different serving framework
means rewriting that loop and little else.

<h2 class="section">When the router cannot see the instances</h2>

SFS assumes real-time workload snapshots. An enterprise routing to third-party cloud
endpoints has no such visibility, and has to fall back on time-averaged quantities. For that
case the paper models an instance as a **Limited Processor Sharing** queue: up to $k$
queries are served concurrently and share the compute capacity equally, the rest wait FCFS.
This matches autoregressive generation more closely than a single-server queue does —
concurrency is real but bounded by memory, and per-query token generation slows as more
sequences share the GPU. With arrival rate $\alpha\_j$, service capacity $\mu\_j$ and
$\rho\_j = \alpha\_j/\mu\_j < 1$, the expected wait is

$$\widehat W^{\textrm{avg}}_{i,j} = \frac{(\alpha_j/\mu_j)^k}{\mu_j-\alpha_j}$$

<figure class="fig-pair">
  <img src="{{ '/papers/latency-routing/figures/lps-scheme.png' | relative_url }}" alt="Diagram of the limited processor sharing scheme with k concurrent servers and a FCFS queue behind them">
  <img src="{{ '/papers/latency-routing/figures/lps-delay-vs-load.png' | relative_url }}" alt="Normalised delay against load, measured against the theoretical LPS curve">

  <figcaption>The Limited Processor Sharing abstraction (<b>left</b>) and its fit
  (<b>right</b>). Measured TTFT on Qwen3-0.6B tracks the qualitative growth of the LPS
  waiting time with load; the observed time-average parallelism for this deployment is
  <b>k ≈ 25</b>. The estimate is coarser than SFS by construction — it knows rates, not
  workloads — but it needs nothing from the serving layer.</figcaption>
</figure>

<h2 class="section">What this does and does not settle</h2>

The claim is narrow and, we think, well supported: latency belongs *inside* the routing
objective rather than delegated to a layer below it, and the estimate it requires can be
obtained cheaply by simulating the serving framework instead of by summarising it into a
throughput number. Joint accuracy–cost–latency routing buys over 40% more utility than
standard load balancing at comparable latency.

Several things remain open. The objective optimises TTFT, which is the right target for
interactive use but not for a workload judged on complete responses — end-to-end latency
brings decode-length prediction error back onto the critical path, where TTFT largely
escaped it. Routing is also treated as the only decision: which models sit on which
hardware is fixed in advance, and joint optimisation of routing and model placement is a
natural next step. Finally, the estimator assumes it is reasoning about an instance whose
workload it can account for; several independent routers sharing a pool of instances would
each be simulating a system the others are perturbing, and that interaction is not yet
modelled.

<p class="lr-note" style="margin-top:2rem">
The simulation on this page is a re-implementation built for exposition — the same batching
policy and the same batch-time model as the paper, but with illustrative coefficients and a
deliberately equalised instance pool. Its purpose is to make the failure mode visible, not
to stand in for the measured results above. The released system is at
<a href="https://github.com/akaashrp/sfs">github.com/akaashrp/sfs</a>.
</p>

---
layout: default
description: >-
  Shivam Patel is a PhD student in Electrical and Computer Engineering at Carnegie Mellon University working on efficient and low-latency LLM inference.
---

<div class="intro">
  {% include profile-photo.html %}
  <div>
    <h1>Shivam Patel</h1>
    <p class="role">
      <span>PhD Student, <a href="https://www.ece.cmu.edu">Electrical &amp; Computer Engineering</a></span>
      <span class="inst"><a href="https://www.cmu.edu">Carnegie Mellon University</a></span>
    </p>
    <p class="contact">shivamap [at] andrew [dot] cmu [dot] edu</p>
  </div>
</div>

I am a third-year PhD student in [ECE](https://www.ece.cmu.edu) at [Carnegie Mellon University](https://www.cmu.edu), fortunate to be advised by [Prof. Gauri Joshi](https://www.andrew.cmu.edu/user/gaurij/).


I was an AI research intern at Bosch AI in summer of 2026, where I worked on datacenter level inference optimization for LLMs. I did my undergrad in Electrical Engineering from IIT Bombay in 2024, advised by [Prof. Vivek Borkar](https://en.wikipedia.org/wiki/Vivek_Borkar). I also spent a great summer at [USC Viterbi](https://viterbischool.usc.edu) with [Prof. Meisam Razaviyayn](https://sites.usc.edu/razaviyayn/) in 2023. 


<h3 class="overview-head">Research Overview</h3>

<div class="overview" markdown="1">
My research focuses on making **autoregressive inference more efficient and capable**, through algorithmic and systems advances for solving complex tasks. In particular, I study how to improve the **compute and memory efficiency of LLM systems**, with an emphasis on effectively utilizing multiple models to push the Pareto frontier of **performance and latency**. Specifically I care about:
1. **LLM Ensembles:** Combining multiple language models to improve overall performance.
2. **Efficient Inference:** Reducing the compute, memory, latency, and cost of LLM systems.
3. **Inference Systems:** Building better serving engines and scheduling policies for complex, multi-model and multi-turn workloads.


</div>

Before CMU, I worked on theoretical aspects of machine learning. I co-designed an asymptotic CVaR risk measure, and a method for improved statistical fairness in ML models. 



<h2 class="section">News</h2>

<div class="news">
  <dl>
    {%- for item in site.data.news -%}
    <dt>{{ item.date }}</dt>
    <dd>{{ item.text }}</dd>
    {%- endfor -%}
  </dl>
</div>

<h2 class="section">
  Selected Publications
  <span class="more"><a href="{{ '/publications' | relative_url }}">All publications &rarr;</a></span>
</h2>

{% include pub-list.html selected_only="true" %}

<h2 class="section">Experience</h2>

<ul class="stack">
  <li>
    <span class="when">May – Aug 2026</span>
    <span class="what"><strong>AI Research Intern</strong><span>Robert Bosch Research, Pittsburgh · with Dr. Bingqing Chen</span></span>
  </li>
  <li>
    <span class="when">2024 – present</span>
    <span class="what"><strong>Graduate Researcher</strong><span>Carnegie Mellon University · with Prof. Gauri Joshi</span></span>
  </li>
  <li>
    <span class="when">May – Aug 2023</span>
    <span class="what"><strong>Visiting Undergraduate Researcher</strong><span>USC Viterbi School of Engineering · with Prof. Meisam Razaviyayn</span></span>
  </li>
</ul>

<h2 class="section">Awards</h2>

<ul class="stack compact scroller">
  <li><span class="when">2026</span><span class="what"><strong>Exemplary PhD Qualifying Examination Performance</strong><span>ECE, Carnegie Mellon University</span></span></li>
  <li><span class="when">2025 – 26</span><span class="what"><strong>William Messner Endowed Fellowship</strong><span>ECE, Carnegie Mellon University</span></span></li>
  <li><span class="when">2024 – 25</span><span class="what"><strong>Carnegie Institute of Technology Dean's Fellowship</strong><span>Carnegie Mellon University</span></span></li>
  <li><span class="when">2023</span><span class="what"><strong>IUSSTF-Viterbi Scholarship</strong><span>Dept. of Science &amp; Technology, Govt. of India — one of 15 selected nationally</span></span></li>
  <li><span class="when">2020</span><span class="what"><strong>KVPY Fellowship</strong><span>Indian Institute of Science &amp; DST, India</span></span></li>
</ul>

<h2 class="section">Beyond research</h2>

I have recently become an endurance sports fan — get in touch if you are around Pittsburgh and up for a long run or a swim. I used to be an avid birdwatcher, and I played tabla and harmonium in an early life. I speak four languages.

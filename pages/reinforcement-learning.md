---
layout: page
---
<h2><b>Foundations of Intelligent Learning Agents</b></h2>
<h3><b>Regret Minimization, Policy Evaluation and Policy Improvement for Reinforcement Learning</b></h3>

-------------------------------------------------------------------------------------------------------------------    
The assignments/projects for the course can be found [here](https://github.com/patel-shivam/CS747).  
Refer the [course website](https://www.cse.iitb.ac.in/~shivaram/teaching/cs747-a2022/index.html) for more information. 


<h4><b>Regret Minimization Algorithms for Multi Armed Bandit Formulations</b></h4>  

![Multi Armed Bandit](/images/rl-images/k-armed-regret-1.png){: width="400" }    

Multi Armed Bandit Problems require regret minimization algorithms to optimize the explore-exploit tradeoff for different horizon scales.   
In the first assignment, I implemented the Epsilon-Greedy algorithms for linear regret.   
> $R_T = \epsilon(p^\* - p_{avg})T = \Omega(T)$

Later on, I used UCB, KL-UCB and Thompson Sampling algorithms which are in accordance with the Lai-Robbins logarithmic bound. 

> $ucb^t_a = \hat{p_a}^t + \sqrt{\frac{2ln(t)}{u_a^t}}$  

> $ucb-kl^t_a$ is the solution of $q \in [\hat{p_a}^t, 1]$ for $KL(\hat{p_a}^t, q) = \frac{ln(t) + cln(ln(t))}{u_a^t}$

Lai Robbins Logarithmic bound -   
> $\frac{R_T(L,I)}{ln(T)} \geq \Sigma_{a:p_a(l) \neq p^\*(l)} \frac{p^\*(l) - p_a(l)}{KL(p_a(l), p^\*(l))}$



Later on, the problem of having finite stage feedback was encountered, where we get to know about the regrets only after a batch of samples, and we need to provide the distribution of which arms to be pulled how frequently in the next batch. Thompson Subsampling was implemented for finite stage feedback formulation. 

The next task was to find an algorithm that works well when the number of arms is comparable to the horizon T. 
GLIE-ified Epsilon Greedy with quantile regret minimization was implemented for num_arms A = horizon T. (GLIE - Greedy in the Limit, Infinite Exploration)  

Assignment repository can be found [here](https://github.com/patel-shivam/CS747/tree/main/Assignment1)

<h4><b>Policy Evaluation, Improvement and MDP Planning</b></h4>

![Policy Iteration](/images/rl-images/policy-iteration.png){: width="400" }    
 
 Given any Markov Decision Process (MDP), we need to determine the Optimal Value Function $V^\*$ and Optimal Policy $\pi^\*$. I implemented three different algorithms for this purpose - 
 * **Value Iteration** - It iteratively updates the value function of each state in the MDP, leading to guaranteed convergence in countably finite number of iterations
   > $V_0$ ← Arbitrary, element-wise bounded, n-length vector.
   > t ← 0
   > Repeat:
   >>For $s \in S$:  
   >>>$V_{t+1}(s) \leftarrow max_{a \in A} \Sigma_{s^{'}\in S} T(s, a, s^{'})(R(s, a, s^{'}) + \gamma V_t(s^{'}))$   
   >>>$t \leftarrow t+1$  
   >   
   > Until $V_t \approx V_{t-1}$ (upto machine precision)        


 * **Howard's Policy Iteration** - HPI is a policy improvement algorithm, which randomly chooses improvable states, and creates new policies based on the Action Value Function of the MDP. This is also guaranteed to converge, by the Banach's Fixed Point theorem.  
 * **Linear Programming** - We create a set of $|nk|$ linear inequalities from the $|n|$ Bellman Equations, and use an off-the-shelf linear solver to get the otimal value function of the MDP. 
   > Bellman Optimality Equations -   
   > $V^\*(s) = max_{a \in A} \Sigma_{s^{'}\in S} T(s, a, s^{'})(R(s, a, s^{'}) + \gamma V^\*(s^{'}))$  
   > Linear constraints for Bellman Equations -   
   > $V(s) \geq \Sigma_{s^{'}\in S} T(s, a, s^{'})(R(s, a, s^{'}) + \gamma V(s^{'}))$ 

Assignment repository can be found [here](https://github.com/patel-shivam/CS747/tree/main/Assignment2). 





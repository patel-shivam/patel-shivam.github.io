---
layout: page
---
<h2><b>Foundations of Intelligent Learning Agents</b></h2>
<!--- <h3><b>Regret Minimization, Policy Evaluation and Policy Improvement for Reinforcement Learning</b></h3> -->


-------------------------------------------------------------------------------------------------------------------       
-------------------------------------------------------------------------------------------------------------------      
<!---<span style="background-color:AliceBlue">-->
The assignments/projects for the course can be found [here](https://github.com/patel-shivam/CS747).   
Refer the [course website](https://www.cse.iitb.ac.in/~shivaram/teaching/cs747-a2022/index.html) for more information. 
<!---</span>-->


<h4><b>Regret Minimization Algorithms for Multi Armed Bandit Formulations</b></h4>    

-------------------------------------------------------------------------------------------------------------------       

![Multi Armed Bandit](/images/rl-images/k-armed-regret-1.png){: width="400" }    

Multi Armed Bandit Problems require regret minimization algorithms to optimize the explore-exploit tradeoff for different horizon scales.   
In the first assignment, I implemented the Epsilon-Greedy algorithms for linear regret.   

   ![epsilon greedy](/images/rl-images/rl1.png){: width="400" }    

Later on, I used UCB, KL-UCB and Thompson Sampling algorithms which are in accordance with the Lai-Robbins logarithmic bound. 

![ucb, klucb](/images/rl-images/rl2.png){: width="600" }    




Later on, the problem of having finite stage feedback was encountered, where we get to know about the regrets only after a batch of samples, and we need to provide the distribution of which arms to be pulled how frequently in the next batch. Thompson Subsampling was implemented for finite stage feedback formulation. 

The next task was to find an algorithm that works well when the number of arms is comparable to the horizon T. 
GLIE-ified Epsilon Greedy with quantile regret minimization was implemented for num_arms A = horizon T. (GLIE - Greedy in the Limit, Infinite Exploration)  


Assignment repository can be found [here](https://github.com/patel-shivam/CS747/tree/main/Assignment1).  
Assignment report can be found [here](/files/CS747_Assn1_Report.pdf).  


<h4><b>Policy Evaluation, Improvement and MDP Planning</b></h4>  

-------------------------------------------------------------------------------------------------------------------       

![Policy Iteration](/images/rl-images/policy-iteration.png){: width="500" }    
 
 Given any Markov Decision Process (MDP), we need to determine the Optimal Value Function V* and Optimal Policy π*. I implemented three different algorithms for this purpose - 
 * **Value Iteration** - It iteratively updates the value function of each state in the MDP, leading to guaranteed convergence in countably finite number of iterations  ![value iteration](/images/rl-images/rl3.png){: width="600" }    
 

 * **Howard's Policy Iteration** - HPI is a policy improvement algorithm, which randomly chooses improvable states, and creates new policies based on the Action Value Function of the MDP. This is also guaranteed to converge, by the Banach's Fixed Point theorem.  
 * **Linear Programming** - We create a set of n\*k linear inequalities from the n Bellman Equations, and use an off-the-shelf linear solver to get the otimal value function of the MDP.  
   
 ![Bellman Equations](/images/rl-images/rl4.png){: width="600" }   
 
   

Assignment repository can be found [here](https://github.com/patel-shivam/CS747/tree/main/Assignment2).     
Assignment report can be found [here](/files/CS747_Assn2_Report.pdf).  




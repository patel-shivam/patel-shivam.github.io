---
layout: page
---
<h2><b>Tilt Estimation for Cell Tower Antennas</b></h2>
<h3><b> Artificial Intelligence Intern, Reliance Industries Limited</b></h3>

-------------------------------------------------------------------------------------------------------------------  


![Radiation Pattern of Towers](/images/tilt-estimation/tilt-estimation-tower-pattern.png){: width="600" }


Dynamic cell towers resource allocation efficiently distributes bandwidth and hardware resources amongst users. Cell towers are made of antenna 
arrays, which allow to control and localize the effective beam of radiation in any particular direction. Such directional antennas serve the 
cellular communication requirement of users through various spectrum band allocation systems. We focused on the 850, 1800 and 2300 MHz bands 
for predicting optimal tilt of antennas.   

![Radiation Pattern](/images/tilt-estimation/radiation-pattern.png){: width="600" }

An antenna is vertically tilted and azimuth adjusted towards the maximum peak of mean user demand. The vertical tilt of the antenna is composed of a mechanical tilt, and an electrical tilt adjustment mechanism. A simple trigonometric equation gives us 
the ideal solution, which rarely works in the real scenarios. With multipath propagation, channel distortion and varying geographical 
conditions, the exact relation of tilt angle is hard to formulate.  

Our work was focused on finding optimal prediction models for discrete tile angles in degrees. We used domain knowledge and hypothesis testing
to ascertain validity of various features towards predicting tilt. Parameters like extracted geographical coordinates, azimuth, spectrum band, 
demand, mean number of calls per day, peak hour demand, stationary and moving demand ration etc. were used for training a neural network.   

Our successful implementation gave over 99% accuracy in tilt prediction, alongwith a negligible 0.07 degrees MAE. 
 
 
![Moving Demand](/images/tilt-estimation/motion-allocation.png){: width="600" }  

![SHAP explainability](/images/tilt-estimation/tilt-estimation-waterfall.png){: width="600" }
  
  
![Cell Tower](/images/tilt-estimation/cell-tower-img.jpg){: width="300" }




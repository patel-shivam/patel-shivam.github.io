---
layout: page
---

<h1><b>RISC (Multistate and Pipeline) Microprocessor Design</b></h1>
<h2><b>(Reduced Instruction Set Architecture)</b></h2>
-------------------------------------------------------------------    
This the IITB RISC microprocessor design what we did as a course project for [EE309, Microprocessors](https://www.ee.iitb.ac.in/web/academics/courses/EE309). 



 **MultiState Design**
-------------------------------------------------------------------  
![RISC MultiState High Level](/images/risc_microprocessor/risc_high_level_design.png){: width="320" }

Designed a multistate implementation of IITB-RISC, a 16-bit computer based on the Little Computer Architecture. The IITB-RISC is an 8-register, 16-bit computer system. It has 8 general-purpose registers (R0 to R7). Register R7 is always stores Program Counter. This architecture uses condition code register which has two flags Carry flag (C) and Zero flag (Z). The IITB-RISC is simple, but general enough to solve complex problems. The architecture allows predicated instruction execution and multiple load and store execution. There are three machine-code instruction formats (R, I, and J type) and a total of 17 instructions.   
![RISC Multipath RTL](/images/risc_microprocessor/risc_rtl_view.png){: width="320" }

 **Pipelined Design**
-------------------------------------------------------------------
![RISC Pipeline High Level](/images/risc_microprocessor/pipeline_high_level_design.png){: width="320" }  
Implemented a 6 stage pipelined processor, IITB-RISC-22, whose instruction set architecture was provided. IITB-RISC is a 16-bit very simple computer developed for the teaching that is based on the Little Computer Architecture. The IITB-RISC-22 is a 16-bit computer system with 8 registers. It follows the standard 6 stage pipelines (Instruction fetch, instruction decode, register read, execute, memory access, and write back). The architecture is optimized for performance, i.e., includes hazard mitigation techniques. Hence, it has forwarding and branch prediction technique  


![RISC Pipeline RTL](/images/risc_microprocessor/RTL_viewer_pipeline.png){: width="320" }

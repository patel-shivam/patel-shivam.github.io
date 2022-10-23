---
layout: page
---
This the IITB RISC microprocessor design what we did as a course project for EE309. 
![RISC MultiState High Level](/images/risc_microprocessor/risc_high_level_design.png){: width="320" }
![RISC Pipeline High Level](/images/risc_microprocessor/pipeline_high_level_design.png){: width="320" }



IITB-RISC is a 16-bit very simple computer developed for the teaching that is based on the Little Computer Architecture. The IITB-RISC is an 8-register, 16-bit computer system. It has 8 general-purpose registers (R0 to R7). Register R7 is always stores Program Counter. All addresses are short word addresses (i.e., address 0 corresponds to the first two bytes of main memory, address 1 corresponds to the second two bytes of main memory, etc.). This architecture uses condition code register which has two flags Carry flag (C) and Zero flag (Z). The IITB-RISC is very simple, but it is general enough to solve complex problems. The architecture allows predicated instruction execution and multiple load and store execution. There are three machine-code instruction formats (R, I, and J type) and a total of 17 instructions. 

Implemented a 6 stage pipelined processor, IITB-RISC-22, whose instruction set architecture was provided. IITB-RISC is a 16-bit very simple computer developed for the teaching that is based on the Little Computer Architecture. The IITB-RISC-22 is a 16-bit computer system with 8 registers. It follows the standard 6 stage pipelines (Instruction fetch, instruction decode, register read, execute, memory access, and write back). The architecture is optimized for performance, i.e., includes hazard mitigation techniques. Hence, it has forwarding and branch prediction technique

![RISC Multipath RTL](/images/risc_microprocessor/risc_rtl_view.png){: width="320" }
![RISC Pipeline RTL](/images/risc_microprocessor/RTL_viewer_pipeline.png){: width="320" }

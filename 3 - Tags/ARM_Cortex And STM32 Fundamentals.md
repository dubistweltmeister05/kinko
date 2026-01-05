[[PICT_EMBEDDED]]

# Content

- What is ARM and why is it split into **A / R / M** instead of “one CPU for everything”
    ARM is a British company that designs and licenses Architecture for a wide range of CPUs. They are based on the RISC architecture of computing. which is inherently designed to simplify the individual instructions given to the computer to accomplish tasks. Compared to the instructions given to a complex instruction set computer (CISC), a RISC computer might require more machine code in order to accomplish a task because the individual instructions perform simpler operations. 
    
    The goal is to offset the need to process more instructions by increasing the speed of each instruction, in particular by implementing an instruction pipeline, which may be simpler to achieve given simpler instructions  
- Design constraints:
    
    - **Throughput vs Determinism vs Power**
        
- Typical environments:
    
    - Cortex-A → MMU, Linux, user/kernel space
        
    - Cortex-R → hard real-time, safety
        
    - Cortex-M → MCU, bare-metal / RTOS
### Links
https://sirinsoftware.com/blog/the-arm-processor-a-r-and-m-categories-and-their-specifics


## ECE 565 Programming Assignment 2 Fall 2022
### Professor Mithuna Thottethodi

1. **Assignment**
    
    gem5 is an immensely complex piece of software with over 100k lines of code. However, it is not necessary to understand all of gem5 prior to being able to effectively use it. For this assignment you should focus on the *MinorCPU* Model located in /src/cpu/minor. The *MinorCPU* currently models a simple pipelined.  (Hint: Use the -h flag with your Python script to find out how to specify the CPU model you use.)
    One caveat of this CPU model is that each pipeline stage/component inherits from an abstracted resource object. As with every resource, there is the contingency of encountering a structural hazard. The occurrence of a structural hazard is dependent on the width of the corresponding stage. Thus in order to model one outstanding instruction per stage, you have to adjust the width of each stage to be 1 (*hint: look inside MinorCPU.py*).
    For this programming assignment, you will be required to implement the following changes in gem5:

    * Evaluate a simple pipeline.
    * Degrade branch prediction
    * Split the Execution stage into two separate pipeline stages
    
    Each of these are detailed further below. For part (i), just evaluate the example code, running to completion. For parts (ii) and (iii), you will need to run the SPEC benchmarks for 100 million instructions.  These results will then be compared to the baseline gem5 performance for the MinorCPU model. Make sure to take advantage of different output directories to avoid overwriting output data from different runs. 
    
    A nice overview of MinorCPU can be found here: https://www.gem5.org/documentation/general_docs/cpu_models/minor_cpu
    
    1. **Evaluate a Simple Pipeline**
    
        In this task - you will write a C program, compile it then run it using different CPU models. 
    
        1. **Run a custom program (x86)**
            
            The DAXPY loop (double precision aX + Y) is an oft used operation in programs that work with matrices and vectors. The following code implements DAXPY in C++11.
            Place the code inside a new directory *part1/daxpy.cc*.
    
            ```cpp
            #include <random>
            #include <iostream>
            
            int main()
            {
                const int N = 1000;
                double X[N];
                double Y[N];
                double alpha = 0.5;
                std::random_device rd; std::mt19937 gen(rd());
                std::uniform_real_distribution<> dis(1, 2);
                for (int i = 0; i < N; ++i)
                {
                X[i] = dis(gen);
                Y[i] = dis(gen);
                }
            
                // Start of daxpy loop
                for (int i = 0; i < N; ++i)
                {
                    Y[i] = alpha * X[i] + Y[i];
                }
                // End of daxpy loop
   
                double sum = 0;
                for (int i = 0; i < N; ++i)
                {
                    sum += Y[i];
                }
                std::cout << sum;
                return 0;·
            }
            ```
            
        
            Your first task is to compile this code statically and simulate it with gem5 using the timing simple cpu.
            To compile the code on qstruct, use the following compiler arguments (for c++11 and optimizations):
                    
            ```console
            /usr/bin/g++ -O2 -std=gnu++11 daxpy.cc
            ```
            Compile the program with -O2 flag to avoid running into unimplemented x87 instructions while simulating with gem5. Report the breakup of instructions for different op classes. For this, grep for op_class in the file stats.txt. Run the compiled file using
            
            ```console
            ./build/ECE565-X86/gem5.opt configs/example/se.py -c <output-from-daxpy-build>
            ```
    
        2. **Examine the assembly (x86)**
            
            Generate the assembly code for the daxpy program above by using the -S and -O2 options when compiling with GCC. As you can see from the assembly code, instructions that are not central to the actual task of the program (computing aX + Y) will also be simulated. This includes the instructions for generating the vectors X and Y, summing elements in Y and printing the sum. When I compiled the code with -S, I got about 350 lines of assembly code, with only about 10-15 lines for the actual daxpy loop.

            Usually while carrying out experiments for evaluating a design, one would like to look only at statistics for the portion of the code that is most important. To do so, typically programs are annotated so that the simulator, on reaching an annotated portion of the code, carries out functions like create a checkpoint, output and reset statistical variables.
            
            The mechanism for annotating the code is a library of operates called `m5ops`. You have to build it first. To do so, you need to follow these steps.
            
            1. Pull the updates from the main course repository from the main gem5 directory. (You have to commit any uncommitted changes before you can pull.)

             ```console
                   git pull
             ``` 
             
            2. You will edit the C++ code from the first part to output and reset stats just before the start of the DAXPY loop and just after it. For this, include the file ./include/gem5/m5ops.h in the program. To do so, you should use the line `#include<gem5/m5ops.h"` line in your source file. Use the function `m5_dump_reset_stats()` from this file in your program. This function outputs the statistical variables and then resets them. You can provide 0 as the value for the delay and the period arguments. This edited source file is now ready. But you cannot compile it yet until you have first built the m5ops library.
            3.  Build the m5ops library using the following command in the `<gem5-Fall2022>/util/m5` directory:
            
            ```console
                scons-3 build/x86/out/m5
            ```
            
            4. Now you can compile the DAXPY code (assuming the filename is daxpy.cc) using the following command from within the part1 directory. (If your source code is in another directory, please modify the include path and library path accordingly.)
                
            ```console
                /usr/bin/g++ -O2 -std=gnu++11 -I ../include -L ../util/m5/build/x86/out/ daxpy.cc -lm5
            ```

            <details>
                <summary> Old incompatible instructions for m5ops preserved here. Do not use.</summary>
    
            To provide the definition of the m5_dump_reset_stats(), go to the directory util/m5/src/x86/ and edit the SConsopts in the following way:

            ```console
            diff --git a/util/m5/src/x86/SConsopts b/util/m5/src/x86/SConsopts
            index 8763f29..7be70a3 100644
            --- a/util/m5/src/x86/SConsopts
            +++ b/util/m5/src/x86/SConsopts
            @@ -27,7 +27,6 @@ Import('*')
               
            env['VARIANT'] = 'x86'
            get_variant_opt('CROSS_COMPILE', '')
            -env.Append(CFLAGS='-DM5OP_ADDR=0xFFFF0000')
                
            env['CALL_TYPE']['inst'].impl('m5op.S')
            env['CALL_TYPE']['addr'].impl('m5op_addr.S', default=True)
            ```
            
            Execute the following command in the directory util/m5:
                
            ```console
            scons-3 src/x86 
            ```
            
            This will create an object file named util/m5/build/x86/x86/m5op.o. 
        </details>

        Now again simulate the program with the timing simple CPU. This time you should see three sets of statistics in the file stats.txt. Report the breakup of instructions among different op classes for the three parts of the program. In the assignment report, provide the fragment of the generated assembly code that starts with the call to m5_dump_reset_stats() and ends m5_dump_reset_stats(), and has the main daxpy loop in between.

2. **Submission instructions**
    
The main deliverable of this assignment that will be graded is a report (and not the code). The code will still have to be submitted as a single patch for plagiarism checking. But the code is not what gets graded.
    
Submit a report (maximum three pages) with graphs and/or tables to present results via brightspace. Each graph should summarize the results of one of the experiments. (Suggested format: Plot bar-graphs with benchmarks on the X-axis and IPC on the Y-axis. For each benchmark, show two or more bars; one is the baseline and the rest correspond to your changes.) The report is NOT meant to be an exercise in writing. I do not expect any text beyond a 2-3 sentence summary of the key observations for each graph. However, this is a minimum and not a maximum. If you have text you want me to see (e.g., assumptions, simplifications, data-gathering difficulties etc.), feel free to write additional text in the report. 
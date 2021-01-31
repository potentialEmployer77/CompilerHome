# Compiler
Welcome to the public homepage for the private Compiler repository.

## Overview
The goal of this project is to apply compiler theory to the construction of a real compiler for a small but fully-functional language. Currently, only the code for the compiler backend has been added to the repository, however, my hope is to upload the compiler frontend code soon in order to have a fully working end-to-end compiler. In the future, I plan to use this codebase as a sandbox for further exploration of compiler concepts such as optimizations and static analysis as well as compiler toolchains such as LLVM.

This project was written in Java.

<strong>Note that this is the public home page for the Compiler project, which is a private repository. In order to access the actual private repository and the project's source code, please log into this GitHub account (potentialEmployer77) using the login credentials provided at the top of my resume.</strong>

## Compiler Backend Overview

The compiler backend runs on an input text file (.ir) which contains a sequence of three-address code IR instructions and performs the following tasks:

<ol>
  <li>Register allocation</li>
  <ol>
    <li>Naive scheme (no analysis)</li>
    <li>Local scheme (control-flow graph and basic block analysis)</li>
    <li>Global scheme (graph coloring)</li>
  </ol>
  <li>Instruction selection</li>
  <li>Code generation</li>
</ol>

The output of the compiler backend is an assembly language file (.asm) consisting of three sections: Data, Main, Function. The file can be run on a MIPS machine (or a MIPS simulator such as SPIM).

### Build Instructions

To build, run `make`.

#### Running

To run, execute: 
`java TigerBackEnd -f <path/to/file> -a <naive|local|global> -t[0-8] -s[0-8] -f[0-30] -d`
  1. -f <filename> specifies path to .ir file.
  2. -a <naive|local|global> specifies which allocator to use. 
    *Defaults to local.
  3. -t[0-8] : the number of MIPS temporary registers to be allocated and  assigned to temporary virtual register names (e.g., $t7). 
    * The default is 8. 
    * Ignored when using naive allocator.
  4. -s[0-8] : the number of MIPS saved temporary registers to be allocated and assigned to non-temporary virtual register names (i.e., 0sum). 
    * The default is 8. 
    * Ignored when using naive allocator.
  5. -f[0-30] : the number of MIPS floating point registers to be allocated and assigned to floating point-valued virtual register names (i.e., 0sum). 
    * The default is 30. 
    * Ignored when using naive allocator.
  6. -d prints out debug information";

### Design Internals

Below are the notes on the design of each of the above tasks.

#### Register Allocation

#### Naive Register Allocation

The majority of the logic for Naive Register Allocation is contained in the NaiveRegisterAllocator class. At a high level, the naive register is implemented in the following steps:

1. Before each instruction, insert a "load" instruction loading the operands into registers
2. Replace the variable name in the original instruction with the register just loaded with the value
3. After each instruction, insert a "store instruction storing the value of the register back into the variable's home memory

We accomplish this by looping over every line of the original IR, determining what type of instruction it is, and then calling a specific rewrite function for that type of instruction (ie. rewriteBinaryOp, rewriteAssign etc). Those rewrite functions create a new rewritten line of code (along with the load and store instructions if applicable), and push those lines to an array list of rewritten lines.

This approach allows us to handle corner cases that have to be handled in different instances. For example, when rewriting a binary operation instruction, if the operands are floats, they need to be loaded into float registers rather than the standard temporary registers.

### Intrablock Register Allocation

At a high level, the intrablock register allocation is implemented in the following steps:

1. Build live sets for each basic block 
2. Build intrablock live ranges
3. Allocate and assign registers
4. Rewrite the IR
5. Build initial loads list
6. Build final stores list

#### More detailed design internals for each portion

1. Build live sets for each basic block: basicBlock.buildInSet();
Build each block's inSet from its outSet (backward dataflow analysis). For intra-block liveness analysis, the outSet is always empty because nothing has a use beyond the end of the block. For inter-block liveness analysis, the outSet will generally not be empty and
must be built prior to invoking this method.

2. Build intrablock live ranges: basicBlock.buildIntraBlockLiveRanges();
For each member in the inSet of each block, forward-scan through all instructions in the block, looking at their useSet and defSet to build the LiveRanges. Each LiveRange constitutes a DU chain, labeled with the variable's name.

3. Allocate and assign registers: basicBlock.allocateAssignRegs();
Traverse the list of LiveRanges (built in step 2) and assign each LiveRange a register, if one is available. Create a mapping (regsMap) between LiveRange names and registers. Any names that do not make it into this map are unallocated and must be spilled.

4. Rewrite the IR: basicBlock.rewriteIR();
Traverse the list of Instructions in each BasicBlock and tell each Instruction to rewrite itself according to the regsMap (built in step 3). Each Instruction has a useSet and defSet. These sets are used to check the regsMaps to see if a spill load/store needs to be generated or if the values (variables) that this Instruction uses/defines have been assigned a register, and therefore do not require a spill. The Instruction then rewrites its operands accordingly; either generating a spill and rewriting the variable name with the spill register name or directly rewriting the variable name with its assigned register.

5. Build initial loads list: buildInitialLoadsList();
For each member in the inSet of each block, generate a load of the inSet variable into a register. This is necessary in the intrablock allocator only because nothing persists beyond each basic block. That is, the variable 'x' defined in block1 and assigned the register $s0 (with value, say, 5), will not guarantee to use the same register in block2. So, any variable in the block's useSet must be loaded in from memory into its assigned register on block entry.

6. Build final stores list: buildFinalStoresList();
Similar to the intialLoadsList, the finalStoresList exists because of the same restriction  that nothing persists beyond the basic blocks. On block exit, all non-temporary variables must be spilled back to memory so that they can be correctly loaded from memory by any subsequent block that uses those variables.

### Whole Function Register Allocation

At a high level, the whole function register allocation is implemented in the following steps:
1. Build inter-block live ranges 
2. Build webs from the inter-block live ranges
3. Compute a spill cost for each web
4. Construct an interference graph from the webs
3. Assign registers to webs using a graph "coloring" algorithm, spilling the lowest spill cost webs first

#### More detailed design internals for each portion

1. Build inter-block live ranges
The logic for building inter-block live ranges is primarily contained in two methods: buildInterBlockLiveSets() and buildWebs() both on the FunctionRecord class.
The basic blocks and control flow graph for the program are already identified as part of the implementation of "CFG and intra-block allocation" (see earlier section). We use these structures to implement the forward-iterative, fixed-point algorithm for computing live sets (inSet and outSet).
First we set a fixed point flag to "false". This flag will represent whether any changes were made to LiveIn or LiveOut sets on the last pass through the CFG. We make repeated passes over the CFG until no changes have been made to the LiveIn or LiveOut sets (the fixed point flag remains "false").
On each pass of the CFG we loop through each basic block and recompute the blocks LiveOut set. Based on the equation for each block B:
LiveOut[B] = Union of LiveIn for each Successor of B
The LiveIn set is computed starting with the final instruction using the equation for each instruction I:
LiveIn[I] = (Out[I] - Def[I]) Union Use[I]
We determine whether a change occurred in the sets for a given block by comparing the new sets generated against a copy of the original In and Out sets generated before running the algorithm.
If a change has occurred, we set the fixed point flag to "true", which leads to another loop through the full CFG. If there's no change in the sets for any of the basic blocks, the fixed point flag remains as "false", and the loop ends.
When the fixed point algorithm ends, we have the full live ranges for each variable in the program. The next step is to compute the webs for each variable in the program.
In our program, webs are represented using the Web class. A Web for a given named value (program variable) represents a collection of interblock (global) live ranges that span from a definition of the name to its last use. In our Web class, the inter-block live ranges are not represented strictly linearly; instead they are represented by the set of their constituent intrablock (local) LiveRanges.
The implementation logic for computing webs is primarily contained in the buildWebs() method on the FunctionRecord class. The method builds the program webs via the following two stages:
1. Connect a single definition of a value to all reachable "last uses": The program walks down through each BasicBlock searching for definition points (remember, each instruction can define at most one name). For each definition of a variable, it "opens" a new Web that starts at that definition point, and then it searches for all last uses reachable from that definition (possibly in a successor block) to "close" the Web. The result is an initial Web that consists of a set of LiveRanges spawned from the single definition point.
2. Merge any overlapping Webs for the same variable name. If variable x is defined in B1.1 and its last use is at B2.4, the Web built by stage 1 is {[B1.1-B1.end],[B2.start-B2.4]}. But if it is also defined in B3.1 and this definition also reaches the same last use at B2.4, a separate Web will be built: {[B3.1-B3.end][B2.start-B2.4]}. These sets of LiveRanges must be merged into the same Web in the webList.
With the Webs, the program can compute an interference graph. The implementation of interference graphs is contained in the InterferenceGraph class.
The InterferenceGraph class contains adjacency maps (HashMaps) for each type of register (saved, temp, float). These map for each web of the given type what other webs interfere (represented as a List).
The logic for populating the interference graph is contained in the class constructor. First, we loop over web. For each web, we loop over every other web. Then we check if the two webs interfere (hasOverlap()): if their live ranges overlap. 
If the two webs do interfere (and would be assigned the same type of register: saved, temp, float), we add each web to the other's adjacency list.
Once we have the interference graphs (adjacency maps), we can "color" the graphs. The logic for this is primarily contained in the "color()" method of the InterferenceGraph class.
First, we instantiate a stack to hold the webs as we "remove" them from the graph. Then we perform another fixed point algorithm, this time looping over the "nodes" (webs) of the interference graph until every node has been remove.
To remove nodes we first loop through the webs and check whether (degree < N): whether the number of other webs in conflict is less than the number of registers. If so, we know the node is colorable without spilling, so we remove the node from the graph and push it to the stack.
If we've removed every node that meets the condition (degree < N) and the graph is still not empty, then we have to "spill" a web. We choose a web to spill by picking the one with the smallest "spill cost".
The "spill cost" is represented as a property on each Web and is computed when building the webs (see buildWebs() method and associated helper methods). For every definition and use of the variable of a web, we increment the associated spill cost by one. If the reference occurs in a loop, we assume the loop executes 10 times (to the power of the known depth).
After we spilled a web, we start the process again from the top: checking if any node meets the condition (degree < N) and if not spilling the least spill cost register.
Once we've removed every node from the graph (and placed in the stack), we can start to build the graph back up, coloring it while we do.
We pop nodes (webs) of the stack we've created and assign a register (color) if possible. A web cannot be assigned a register if it is adjacent in the built-up graph to another web that is also assigned that register. If that's not possible, the web is not given a register. The final result is a register map that maps webs to the registers they've been assigned.
This register map represents the whole function register allocation. With it, we can rewrite the IR and assign appropriate registers just as we did the other register allocations, though the mapping from variable to register will differ. Generally, it will be more efficient, as variables will be able to remain mapped to registers across blocks.

## Instruction Selection and Code generation

The logic for instruction selection and code generation is primarily contained in the `./assembler` directory.

Before the modified IR is fed to the MIPSAssembler, it goes through two phases of optimization, handling stack variables and mangling variable names. 

TemporariesStackAllocator is responsible for collecting all the temporary variables (assumed to be any variable in int-list or float-list that begins with '$') and assigning them a position in the stack. After which, it will replace every usage of those temporaries with their stack location.

The NameMangler is responsible for renaming variables so they will not conflict with MIPS instructions.

The primary controlling class for generating assembly is MIPSAssembler. The constructor for this class accepts the linear IR generated by the allocators described above (this is represented as a List<String>).

The MIPSAssembler expects the linear IR to be encoded with the information it needs to drive correct instruction selection. This information is gathered and written to the linear IR by those previous allocators.

The assembler then invokes the constructors of three separate classes: DataSegmentWriter, MainSegmentWriter, and FunctionSegmentWriter. These classes are responsible for generating the corresponding portions of the MIPS assembly code.

The DataSegmentWriter accepts the linear IR (List<String>), and uses the buildSegment() method to build the ".data" segment of the MIPS assembly. Is does this by looping through every line of the linear IR, and when it finds certain types of instructions, extracts the relevant data and appends that to the List<String> dataSegment. For example, when it encounters a "int-list:" instruction, it pulls those ints and places their declarations in the data segment.

The MainSegmentWriter and FunctionSegmentWriter are implemented in nearly the same way. They accept the list of the linear IR, iterate through it, and convert each line of IR to the corresponding line (or lines) of MIPS assembly code. The MainSegmentWriter is in a sense, a special case of the FunctionSegmentWriter, explaining their similarities.

Once the assembly for these three sections is generated, they are written to a single file in the following order: Data, Main, Function. The code on that file can be run on a MIPS machine or a simulator such as SPIM.

## Analyis of Code Quality:

Below are some statistics regarding the quality of the generated assembly code under each of the three register allocation schemes on the test files provided in the Tests directory available at the top level of this repository.

1). factorial.ir:

Naive:
120Stats -- #instructions : 238
         #reads : 77  #writes 73  #branches 23  #other 65
1,256 bytes

Local:
120Stats -- #instructions : 216
         #reads : 40  #writes 54  #branches 23  #other 99
1,164 bytes

Global:
120Stats -- #instructions : 183
         #reads : 30  #writes 22  #branches 23  #other 108
1,042 bytes

2). test1.ir:

Naive:
10000Stats -- #instructions : 3930
         #reads : 1104  #writes 604  #branches 203  #other 2019 
1,519 bytes

Local:
10000Stats -- #instructions : 3035
         #reads : 504  #writes 306  #branches 203  #other 2022
1,425 bytes

Global:
10000Stats -- #instructions : 1528
         #reads : 202  #writes 2  #branches 203  #other 1121
1,333 bytes

3). test2.ir:

Naive:
5Stats -- #instructions : 24
         #reads : 4  #writes 3  #branches 4  #other 13

Local:
5Stats -- #instructions : 26
         #reads : 4  #writes 4  #branches 4  #other 14

Global:
5Stats -- #instructions : 25
         #reads : 5  #writes 3  #branches 4  #other 13

4). test3.ir:
Naive:
5Stats -- #instructions : 36
         #reads : 4  #writes 3  #branches 4  #other 25
376 bytes

Local:
5Stats -- #instructions : 38
         #reads : 4  #writes 4  #branches 4  #other 26
411 bytes

Global:
5Stats -- #instructions : 37
         #reads : 5  #writes 3  #branches 4  #other 25
397 bytes

5). test5.ir:
Naive:
2Stats -- #instructions : 42
         #reads : 8  #writes 7  #branches 5  #other 22
485 bytes

Local:
2Stats -- #instructions : 50
         #reads : 7  #writes 10  #branches 5  #other 28
529 bytes

Global:
2Stats -- #instructions : 29
         #reads : 2  #writes 1  #branches 5  #other 21
417 bytes

// BEGINNING OF OUR TESTS

6). arrayTest1.ir:
Naive:
102Stats -- #instructions : 41
         #reads : 8  #writes 5  #branches 2  #other 26
471 bytes

Local:
102Stats -- #instructions : 39
         #reads : 5  #writes 4  #branches 2  #other 28
439 bytes

Global:
102Stats -- #instructions : 33
         #reads : 4  #writes 2  #branches 2  #other 25
410 bytes

7). floatArithmetic.ir:
Naive:
5Stats -- #instructions : 52
         #reads : 2  #writes 1  #branches 2  #other 47
464 bytes

Local:
5Stats -- #instructions : 40
         #reads : 2  #writes 1  #branches 2  #other 35
417 bytes

Global:
5Stats -- #instructions : 44
         #reads : 2  #writes 1  #branches 2  #other 39
417 bytes

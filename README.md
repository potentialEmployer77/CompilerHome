# Lion
Welcome to the public homepage for the private Lion project repository.

![OsMowSis_example](Documentation/GitHubImages/lion-resize25.png)

## Overview
The goal of this project is to apply compiler theory to the construction of a compiler for a small language called Lion. Currently, only the code for the compiler backend has been added to the repository, however, my hope is to upload the compiler frontend code soon in order to have a fully working end-to-end compiler. In the future, I plan to use this codebase as a sandbox for further exploration of compiler concepts such as optimizations and static analysis as well as compiler toolchains such as LLVM.

This project was written in Java.

<strong>Note that this is the public homepage for the Lion project, which is a private repository. In order to access the actual private repository and the project's source code, please log into this GitHub account (potentialEmployer77) using the login credentials provided at the top of my resume.</strong>

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
`java LionBackEnd -f <path/to/file> -a <naive|local|global> -t[0-8] -s[0-8] -f[0-30] -d`
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
  6. -d prints out debug information.

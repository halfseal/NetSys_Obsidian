---
creator: Maike
type: Paper Review
created: 2022-09-26
---
Summary: The slides use so much jargon, I can barely understand anything. They also leave a lot of aspects unexplained. Seem like a great resource for people that are familiar with computer architecture and GPGPU-Sim, but not for unexperienced undergrads.

I focussed on the 4th session on Microarchitecture. ( [Slides](http://www.gpgpu-sim.org/micro2012-tutorial/) )

The Architecture of a SIMT Core:
![[Untitled(185).png]]


I will be focussing on the ‘SIMT Front End’ only.

<aside> ‼️ Disclaimer: Most of this information is inferred by myself based on the pictures and arrows and my limited knowledge on Computer Architecture. This information might not be accurate.

</aside>

## I-Buffer
![[Untitled(186).png]]


**What is the purpose of the I-Buffer?**

The I-Buffer stores instructions that are ready or soon to be ready for execution.

**How many I-Buffers are there?**

It is unclear whether an I-Buffer is shared across warps or if each warp has its own I-Buffer. On one hand, the PPT reads: “Select a warp and issue an instruction from **its** I-buffer for execution”, suggesting that every warp has its own buffer. However, as seen in the above picture, the W1 W2 W3 seems to indicate instructions for different warps.

**What does a line in the I-Buffer consist of?**

The first bit indicates whether a line is vacant or not aka does a new instruction need to be fetched into the line or is this line still waiting to be issued. The second part is the decoded instruction, though it is not clear what form a decoded instruction is in. The last part is a bit indicating if the instruction is ready to be executed or not.

**When is an instruction issued?**

An instruction is available for issuing if both its vacant bit is negative and its ready bit is positive. Amongst all instructions that satisfy that condition, issuing occurs using Round Robin.

## Scoreboard

**What is the purpose of the Scoreboard?**

The Scoreboard’s task is to identify dependency hazards. A dependency hazard occurs when an instruction that is still in the computing pipeline has to first perform a certain action, because the next instruction relies on it. The hazards that are checked are **R**ead **A**fter **W**rite (RAW) and **W**rite **A**fter **W**rite (WAW). For example, if instruction b needs the variable var, but instruction a, which executes an instruction akin to var++, is still in the pipeline, var could still have the old value in memory and therefore executing instruction b too early might result in incorrect behaviour. This is an example of a RAW hazard.

**How does the Scoreboard detect hazards?**

This is quite unclear, but it seems like when an instruction is issued, it reserves the registers it is going to perform actions on. There is likely going to be a differentiation between registers used for writing vs registers used for reading, but no information is given on that. If another instruction requires a register that is still reserved by another instruction, it will be considered as not ready. Registers are released in the write back stage which in the traditional RISC pipeline is the stage where the results of the execution are written into the registers.

## SIMT Stack
![[Untitled(187).png]]


**What is the SIMT stack?**

In SIMT, only threads that are executing the same instruction can be scheduled together. When a branch occurs, threads executing branch A have to wait while the threads for branch B are scheduled and vice versa. The SIMT stack handles the branch divergence by keeping track of which threads need to execute which instructions. Upon detecting a branch (like an if statement), it is calculated which threads are taking which branch and all possibilities are pushed onto the stack to be processed one by one. Once there are no entries on the stack, threads are synced again.

**What does an entry on the SIMT stack look like?**

First, every entry starts with its **P**rogram **C**ounter (PC). Then, if this entry is only executed by a limited number of threads, it also features an RPC (Probably standing for Reconvergence Program Counter or something), which marks the point where the current branch will reconverge (join with the threads that are executing the other section(s) of the branch). Lastly, each entry has an active mask, indicating which threads are executing the branch and which ones are idle.

## Fetch
![[Untitled(188).png]]


**What is Fetch?**

Fetch is the section that decides which instruction is to be fetched next from the I-Cache.

**What does Fetch base its decision on?**

Fetch receives two inputs: The first is the list of vacant lines in the I-Buffer, the second are the current target PCs as indicated by the SIMT stack(s). It is unclear whether one SIMT stack indicates multiple PCs that are ready for prefetch or if there are multiple SIMT stacks (each for one warp) that all indicate on PC, namely the one at the top of their stack. Based on what criteria the Fetch decides on a PC-I-Buffer combo is not explained. It seems like this is based on the decision of the first warp scheduler which is supposed to be located somewhere here.

## I-Cache

Do not ask me about the I-Cache, it is a mystery. Where do the instructions come from? When are they fetched into the I-Cache? So many questions, no explanations.

## Decoder

The decoder is similarly mysterious, but it can be assumed that it is similar to a normal instruction decoder on the CPU, which gets a bit string as input and interprets it into instruction that is performed, number of registers, register names etc.

---

Old Stuff:
![[Untitled(189).png]]


Best attempt:

The I-Buffer signals vacant slots to the fetcher. The fetcher selects one of the empty slots (that corresponds to a warp) for fetching and uses the Program Counter signalled by the SIMT stack (discussed later) to decide which instruction to fetch. Then, the instruction is fetched from the I-cache using the PC after which it will be decoded into an executable instruction (rather than a string of bits I guess) and stored in the I-buffer. If the instruction is not in the I-cache, it is ignored. Once a decoded instruction is stored in the I-Buffer, the line is set to non-vacant, so that no other instruction will be fetched into that slot. The instruction is evaluated for dependency hazards by the scoreboard and if no hazards are detected, it gets marked as ready. Any ready instruction can then be issued by the scheduler.

Where do instruction come from? Are they originally put into the Instruction Buffer? If so, why cache them? What kind of selection is happening in the selection unit? Are instructions saved in the instruction buffer in an encoded form, decoded by the decoder and again stored in the Instruction buffer? What does ‘valid’ mean and what does the v stand for?

Are the positions of vacant spaces in the instruction buffer given to fetch so that it knows how many instructions to prefetch? Does ARB stand for arbitrate? If so, what does that mean? Does the r on the scoreboard stand for ready?
---
layout: page
title: Progress
---
## Progress
### Week 1
1. Found the original [PI AQM paper](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=916670)
2. Read through the paper to understand how it is different from PIE. Immediate observations - different `T_Update` (sampling frequency), alpha and beta are set differently - possibly dependant on the sampling frequency. Different drop probability equation. Wiki doc [here](https://github.com/linux-pi-programmers/linux/wiki/Differences-between-PI-and-PIE)
3. Read through PIE fully, to understand its components and which components we would be chopping off.

### Week 2
1. Started reading through the PIE [source code](https://github.com/torvalds/linux/blob/63bdf4284c38a48af21745ceb148a087b190cd21/net/sched/sch_pie.c) in Linux. We finished one pass of the code until line 450 and have accumulated a bunch of doubts. We have documented our findings [here](https://github.com/linux-pi-programmers/linux/wiki/PIE-Implementation-in-Linux-Kernel)
2. Created an organisation. Forked the kernel and pushed existing wiki articles here.
3. Began Implementation. 

### Week 3
1. To the best of our knowledge, chopped off all the parts of PIE that aren't needed. Knobs like alpha, beta, q_ref are yet to be updated.
2. Made sch_pi.c in line with the formatting requirements of [checkpatch.pl](https://github.com/torvalds/linux/blob/master/scripts/checkpatch.pl)
3. Downloaded and set up the testbed consisting of multiple VMs and namespaces.
4. Got experience of compiling the entire Linux kernel for the first time and installed the same onto the router in the testbed topology
5. Added the required files to iproute2 repository and installed the same on to the router. Now, have to set AQM to PI using tc on the router and record results using `flent`.

### Week 4
1. Updated the the knobs of a,b and q_ref to those of the original paper and tested using flent.
2. Downgraded matplotlib to 2.1.2 (from 3.0.3) because flent was erroring out and obtained primitive visualizations for the implementation
3. Cleaned up code and make a ```.github.io``` website to accompany the code we've written for reference when submitting the patch
4. Made changes to iproute2 so that a and b can be floating point values that can be passed from the user space.
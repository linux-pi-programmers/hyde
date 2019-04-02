---
layout: page
title: The PI AQM algorithm
---

Knobs:
* W     -> sampling frequency
* a,b   -> weights 
* q_ref -> reference queue delay

Other values
* p      -> drop probability
* q      -> current queue delay
* q_old  -> old queue delay

Algorithm:

Every W times per second:
```python
    # calculate q (paper doesn't say how, but PIE uses Little's law

    # calculate drop probability
    p = a*(q-q_ref) + b*(q_old - q_ref) + p_old

    # update p_old and q_old
    q_old = q
    p_old = p
```







 
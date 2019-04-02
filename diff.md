---
layout: page
title: Differences between PI and PIE 
---

| Attr             	| PIE                                                   	| PI                                                                          	|
|------------------	|-------------------------------------------------------	|-----------------------------------------------------------------------------	|
| Drop Probability 	| p = alpha*(curr_q - ref_q) + beta(curr_q - old_q) + p 	| p =  alpha(curr_q - ref_q) + beta(old_q - ref_q) + p                        	|
| T_update         	| paper - 30ms, RFC - 16ms                              	| paper - experiments section says sampling freq = 160 Hz which means 6.25 ms 	|
| alpha, beta      	| table of values depending on p                        	| They are knobs that don't change with P                                     	|
| DQ_Threshold     	| paper - 10KB, RFC - 16 KB                             	|  -                                                                          	|
| ref_q_delay      	| paper - 20ms, RFC - 15ms                              	| Experiments section says 200 packets                                        	|
| curr_q_delay     	| curr_q_len / avg_dep_rate                             	| Not given                                                                   	|
| avg_dq_rate      	| avg_dq_rate = (1-w)*avg_dq_rate + w*(current_dq_rate) 	| -                                                                           	|
| w                	| paper - 0.5, RFC - 0.125                              	| -                                                                           	|
| current_dq_rate  	| current_dq_rate  = dq_count / dq_time                 	| -                                                                           	|
| burst_allowance  	| paper - 150ms, rfc - 144ms or 160ms                   	| -                                                                           	|


The PI AQM works on using queue length and not queue delay. Hence components of PI involving Little's Law, calculation of average dequeue rate have been removed. 
## [`/net/sched/sch_pie.c`](https://github.com/torvalds/linux/blob/master/net/sched/sch_pie.c)

The main PIE implementation lies in [`/net/sched/sch_pie.c`](https://github.com/torvalds/linux/blob/master/net/sched/sch_pie.c)

### `pie_params`
The main structure `pie_params` holds variables necessary for both PIE and PI.
```c
    struct pie_params {
    	psched_time_t target;	/* user specified target delay in pschedtime */
    	u32 tupdate;		/* timer frequency (in jiffies) */
    	u32 limit;		/* number of packets that can be enqueued */
    	u32 alpha;		/* alpha and beta are between 0 and 32 */
    	u32 beta;		/* and are used for shift relative to 1 */
    	bool ecn;		/* true if ecn is enabled */
    	bool bytemode;		/* to scale drop early prob based on pkt size */
};
```

 - `tupdate` needs to be changed to "Sampling Frequency" as per the PI paper.

### `pie_vars`
```c
/* variables used */
struct pie_vars {
	u64 prob;		/* probability but scaled by u64 limit. */
	psched_time_t burst_time;
	psched_time_t qdelay;
	psched_time_t qdelay_old;
	u64 dq_count;		/* measured in bytes */
	psched_time_t dq_tstamp;	/* drain rate */
	u64 accu_prob;		/* accumulated drop probability */
	u32 avg_dq_rate;	/* bytes per pschedtime tick,scaled */
	u32 qlen_old;		/* in bytes */
	u8 accu_prob_overflows;	/* overflows of accu_prob */
};
```

 - `qdelay` and `qdelay_old` are variables used to calculate drop probability
 - `prob` is the drop probability used for random drops
 - The `dq_` variables are required for Little's law.

> Current queue length is calculated in Qdisc. (Line 276)
```c
	int qlen = sch->qstats.backlog;	/* current queue size in bytes */
```

### `pie_params_init`
```c
static void pie_params_init(struct pie_params *params)
{
	params->alpha = 2;
	params->beta = 20;
	params->tupdate = usecs_to_jiffies(15 * USEC_PER_MSEC);	/* 15 ms */
	params->limit = 1000;	/* default of 1000 packets */
	params->target = PSCHED_NS2TICKS(15 * NSEC_PER_MSEC);	/* 15 ms */
	params->ecn = false;
	params->bytemode = false;
}
```
> For PI, in the paper, `a` was set to 1.822(10)-5 and `b` to 1.816(10)-5  
> "Sampling Frequency" is set to 160Hz  
> The paper uses different notation and does not give recommendations for the knobs.  
> How do we choose the knobs for our PI Implementation?

### `pie_vars_init`
```c
static void pie_vars_init(struct pie_vars *vars)
{
	vars->dq_count = DQCOUNT_INVALID;
	vars->accu_prob = 0;
	vars->avg_dq_rate = 0;
	/* default of 150 ms in pschedtime */
	vars->burst_time = PSCHED_NS2TICKS(150 * NSEC_PER_MSEC);
	vars->accu_prob_overflows = 0;
}
```
 - PI doesn't need `burst_time`
### `drop_early`
```c
static bool drop_early(struct Qdisc *sch, u32 packet_size)
``` 
 - Performs the dropping operation based on if burst allowance is available and by comparing current queue delay to a random number that is generated


### `pie_qdisc_enqueue`
 ```c
 static int pie_qdisc_enqueue(struct sk_buff *skb, struct Qdisc *sch,
			     struct sk_buff **to_free)
 ```
 

### `pie_policy`
```c
static const struct nla_policy pie_policy[TCA_PIE_MAX + 1] = {
	[TCA_PIE_TARGET] = {.type = NLA_U32},
	[TCA_PIE_LIMIT] = {.type = NLA_U32},
	[TCA_PIE_TUPDATE] = {.type = NLA_U32},
	[TCA_PIE_ALPHA] = {.type = NLA_U32},
	[TCA_PIE_BETA] = {.type = NLA_U32},
	[TCA_PIE_ECN] = {.type = NLA_U32},
	[TCA_PIE_BYTEMODE] = {.type = NLA_U32},
};

static int pie_change(struct Qdisc *sch, struct nlattr *opt,
		      struct netlink_ext_ack *extack)
```

> What is this and why?
> Understood that netlink is used as a means for inter process communication, specifically between userspace and kernel space processes and the parameters specified in this policy are the payload in these interprocess messages. But why are things like alpha, beta, tupdate etc being sent to(from?) the kernel space?

### `pie_process_dequeue`
```c
static void pie_process_dequeue(struct Qdisc *sch, struct sk_buff *skb)
{
	struct pie_sched_data *q = qdisc_priv(sch);
	int qlen = sch->qstats.backlog;	/* current queue size in bytes */

	/* If current queue is about 10 packets or more and dq_count is unset
	 * we have enough packets to calculate the drain rate. Save
	 * current time as dq_tstamp and start measurement cycle.
	 */
	if (qlen >= QUEUE_THRESHOLD && q->vars.dq_count == DQCOUNT_INVALID) {
		q->vars.dq_tstamp = psched_get_time();
		q->vars.dq_count = 0;
	}
```
 - The function above is where average departure rate is calculated. We need average departure rate for Little's Law.  
 - The calculation is only one if there are a decent number of packets in the queue: `QUEUE_THRESHOLD`. (Set to 10000, in bytes)  
 - Calculating departure rate  is irrelevant if the queue length is small since the packets would be processed really quickly and the queue is largely empty so enqueuing more packets makes sense.

```c
	/* Calculate the average drain rate from this value.  If queue length
	 * has receded to a small value viz., <= QUEUE_THRESHOLD bytes,reset
	 * the dq_count to -1 as we don't have enough packets to calculate the
	 * drain rate anymore The following if block is entered only when we
	 * have a substantial queue built up (QUEUE_THRESHOLD bytes or more)
	 * and we calculate the drain rate for the threshold here.  dq_count is
	 * in bytes, time difference in psched_time, hence rate is in
	 * bytes/psched_time.
	 */
	if (q->vars.dq_count != DQCOUNT_INVALID) {
		q->vars.dq_count += skb->len;
		if (q->vars.dq_count >= QUEUE_THRESHOLD) {
			psched_time_t now = psched_get_time();
			u32 dtime = now - q->vars.dq_tstamp;
			u32 count = q->vars.dq_count << PIE_SCALE;

			if (dtime == 0)
				return;

			count = count / dtime;

			if (q->vars.avg_dq_rate == 0)
				q->vars.avg_dq_rate = count;
			else
				q->vars.avg_dq_rate =
				    (q->vars.avg_dq_rate -
				     (q->vars.avg_dq_rate >> 3)) + (count >> 3);
```
 - The average departure rate is calculated here.
 - The last line is the EWMA equation.

```c
			/* If the queue has receded below the threshold, we hold
			 * on to the last drain rate calculated, else we reset
			 * dq_count to 0 to re-enter the if block when the next
			 * packet is dequeued
			 */
			if (qlen < QUEUE_THRESHOLD) {
				q->vars.dq_count = DQCOUNT_INVALID;
			} else {
				q->vars.dq_count = 0;
				q->vars.dq_tstamp = psched_get_time();
			}

			if (q->vars.burst_time > 0) {
				if (q->vars.burst_time > dtime)
					q->vars.burst_time -= dtime;
				else
					q->vars.burst_time = 0;
			}
		}
	}
}
```
 - The last bit of the function resets the variables when the queue length has fallen lesser than the threshold.


### `calculate_probability`

```c
static void calculate_probability(struct Qdisc *sch)
{
	struct pie_sched_data *q = qdisc_priv(sch);
	u32 qlen = sch->qstats.backlog;	/* queue size in bytes */
	psched_time_t qdelay = 0;	/* in pschedtime */
	psched_time_t qdelay_old = q->vars.qdelay;	/* in pschedtime */
	s64 delta = 0;		/* determines the change in probability */
	u64 oldprob;
	u64 alpha, beta;
	u32 power;
	bool update_prob = true;

	q->vars.qdelay_old = q->vars.qdelay;

```
 - Initialisation of variables
```c
	if (q->vars.avg_dq_rate > 0)
		qdelay = (qlen << PIE_SCALE) / q->vars.avg_dq_rate;
	else
		qdelay = 0;

	/* If qdelay is zero and qlen is not, it means qlen is very small, less
	 * than dequeue_rate, so we do not update probabilty in this round
	 */
	if (qdelay == 0 && qlen != 0)
		update_prob = false;
```
 - Calculating `qdelay` as per Little's Law
 - The second if condition is for when the qlen is less than 10 packets. (`QUEUE_THRESHOLD`)
```c
	/* In the algorithm, alpha and beta are between 0 and 2 with typical
	 * value for alpha as 0.125. In this implementation, we use values 0-32
	 * passed from user space to represent this. Also, alpha and beta have
	 * unit of HZ and need to be scaled before they can used to update
	 * probability. alpha/beta are updated locally below by scaling down
	 * by 16 to come to 0-2 range.
	 */
	alpha = ((u64)q->params.alpha * (MAX_PROB / PSCHED_TICKS_PER_SEC)) >> 4;
	beta = ((u64)q->params.beta * (MAX_PROB / PSCHED_TICKS_PER_SEC)) >> 4;

	/* We scale alpha and beta differently depending on how heavy the
	 * congestion is. Please see RFC 8033 for details.
	 */
	if (q->vars.prob < MAX_PROB / 10) {
		alpha >>= 1;
		beta >>= 1;

		power = 100;
		while (q->vars.prob < div_u64(MAX_PROB, power) &&
		       power <= 1000000) {
			alpha >>= 2;
			beta >>= 2;
			power *= 10;
		}
	}
```
 - `alpha` and `beta` initialisation. Initially have a 0-32 range but is scaled down by dividing by 16 to come to a 0-2 range.
 - The scaling of alpha is done by a while loop that keeps checking if `prob` is less than `1/power`. `power` is multiplied by 10 every iteration.
 - Definitely a more intuitive implementation than the 5 or so if conditions that was proposed in the RFC.
```c
	/* alpha and beta should be between 0 and 32, in multiples of 1/16 */
	delta += alpha * (u64)(qdelay - q->params.target);
	delta += beta * (u64)(qdelay - qdelay_old);

	oldprob = q->vars.prob;
```
 - The drop probability (p) calculation

```c
	/* to ensure we increase probability in steps of no more than 2% */
	if (delta > (s64)(MAX_PROB / (100 / 2)) &&
	    q->vars.prob >= MAX_PROB / 10)
		delta = (MAX_PROB / 100) * 2;

	/* Non-linear drop:
	 * Tune drop probability to increase quickly for high delays(>= 250ms)
	 * 250ms is derived through experiments and provides error protection
	 */

	if (qdelay > (PSCHED_NS2TICKS(250 * NSEC_PER_MSEC)))
		delta += MAX_PROB / (100 / 2);
```
> This bit wasn't there in the RFC afaik. 

```c
	q->vars.prob += delta;

	if (delta > 0) {
		/* prevent overflow */
		if (q->vars.prob < oldprob) {
			q->vars.prob = MAX_PROB;
			/* Prevent normalization error. If probability is at
			 * maximum value already, we normalize it here, and
			 * skip the check to do a non-linear drop in the next
			 * section.
			 */
			update_prob = false;
		}
	} else {
		/* prevent underflow */
		if (q->vars.prob > oldprob)
			q->vars.prob = 0;
	}
```

 - Simple enough. The bit after the addition is to ensure overflows don't happen and `prob` is still bounded.

```c
	/* Non-linear drop in probability: Reduce drop probability quickly if
	 * delay is 0 for 2 consecutive Tupdate periods.
	 */

	if (qdelay == 0 && qdelay_old == 0 && update_prob)
		/* Reduce drop probability to 98.4% */
		q->vars.prob -= q->vars.prob / 64u;

	q->vars.qdelay = qdelay;
	q->vars.qlen_old = qlen;
```
 - This is there in the RFC. It's decaying the probability exponentially if the qdelay is 0 and stable, aka, during idle periods. P = 0.98*P every tupdate instance that the queue is empty 

```c
	/* We restart the measurement cycle if the following conditions are met
	 * 1. If the delay has been low for 2 consecutive Tupdate periods
	 * 2. Calculated drop probability is zero
	 * 3. We have atleast one estimate for the avg_dq_rate ie.,
	 *    is a non-zero value
	 */
	if ((q->vars.qdelay < q->params.target / 2) &&
	    (q->vars.qdelay_old < q->params.target / 2) &&
	    q->vars.prob == 0 &&
	    q->vars.avg_dq_rate > 0)
		pie_vars_init(&q->vars);
}

```
 - Okay.

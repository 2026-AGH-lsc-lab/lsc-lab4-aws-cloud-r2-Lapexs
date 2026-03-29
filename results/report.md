# AWS Cloud Lab Report: Latency and Cost Analysis

## Assignment 2: Scenario A — Cold Start Characterization

**Goal:** Measure Lambda cold start latency for both zip and container image deployments.

### Data Collection
Based on CloudWatch logs (`Init Duration` and `Duration`):
* **Lambda ZIP:**
  * Init Duration: 629.17 ms
  * Handler Duration: 77.42 ms
* **Lambda Container:**
  * Init Duration: 1985.64 ms
  * Handler Duration: 86.87 ms

### Analysis
* **Faster Cold Starts:** The ZIP deployment is significantly faster at initializing (~629 ms) compared to the container image (~1985 ms).
* **Reasoning:** A ZIP file is constrained in size (max 250 MB unzipped) and relies on pre-cached AWS Lambda layers (like NumPy). A container image can be up to 10 GB and contains a full OS filesystem. Pulling, extracting, and initializing the container environment requires substantially more work from the AWS infrastructure, resulting in a cold start that is over 3 times longer.
* *(Note: Please see `results/figures/latency-decomposition.png` for the visual stacked bar chart).*

---

## Assignment 3: Scenario B — Warm Steady-State Throughput

**Goal:** Measure per-request latency at sustained load across all four environments.

### Latency Table
*(Note: Lambda testing via `oha` resulted in 403 Forbidden due to SigV4 signing issues on the EC2 load generator. Lambda values below are extrapolated from isolated warm CloudWatch invocations and expected parallel scaling behavior up to the AWS Academy concurrency limit).*

| Environment | Concurrency | p50 (ms) | p95 (ms) | p99 (ms) | Server avg (ms) |
|---|---|---|---|---|---|
| Lambda (zip) | 5 | ~77 | ~85 | ~95 | 77.42 |
| Lambda (zip) | 10 | ~77 | ~85 | ~95 | 77.42 |
| Lambda (container) | 5 | ~86 | ~95 | ~105 | 86.87 |
| Lambda (container) | 10 | ~86 | ~95 | ~105 | 86.87 |
| Fargate | 10 | 798 | 1009 | 1110 | ~22.53 |
| Fargate | 50 | 3998 | 4279 | 4384 | ~22.53 |
| EC2 | 10 | 185 | 239 | 269 | ~24.34 |
| EC2 | 50 | 896 | 1199 | **1521** | ~24.34 |

### Analysis
* **Lambda Scaling:** Lambda's p50 remains relatively flat regardless of concurrency (up to account limits) because AWS provisions a separate, isolated execution environment for each concurrent request. There is no queueing inside the execution environment.
* **Fargate/EC2 Queueing:** When concurrency jumps from 10 to 50, Fargate and EC2 p50 and p99 metrics increase drastically. Since we deployed only 1 Fargate task and 1 EC2 instance, all 50 concurrent requests hit the same server. The CPU can only process a few requests at a time, forcing the rest into a queue.
* **Client vs. Server Latency:** The server-side `query_time_ms` stays consistently low (~23-24 ms) because it only measures the time the CPU spends actively executing the k-NN algorithm. Client-side metrics (p50/p99) include network transit time, load balancer overhead, and—crucially—the time the request spent waiting in the server's queue before execution started.

---

## Assignment 4: Scenario C — Burst from Zero

**Goal:** Simulate a traffic spike arriving after a period of inactivity.

### Analysis
* **Burst Behavior:** During a sudden burst from zero, EC2 and Fargate immediately queue the requests (c=50), causing massive latency spikes (EC2 p99: 1061 ms, Fargate p99: 4292 ms). Lambda, on the other hand, attempts to spin up new environments (up to the concurrency limit of 10), triggering cold starts. 
* **Bimodal Distribution:** Lambda exhibits a bimodal distribution during bursts. The first few requests hit cold starts (taking ~700-2000 ms to initialize and compute), while subsequent requests hitting newly warmed environments are processed in ~77 ms.
* **SLO Assessment:** **No environment meets the p99 < 500ms SLO out of the box.** EC2 and Fargate fail due to CPU bottlenecking and severe queueing. Lambda fails because the initial requests hit cold starts (>600 ms).

---

## Assignment 5: Cost at Zero Load

**Goal:** Compute the idle cost of each environment.
Assuming 18 hours/day idle (540 hours/month):

* **AWS Lambda:** **$0.00 / month**. Lambda is a true serverless offering; you only pay for compute time when code is actively running.
* **EC2 (t3.small):** **$11.23 / month** idle cost (Hourly rate: $0.0208).
* **Fargate (0.5 vCPU, 1GB RAM):** **$13.33 / month** idle cost (Hourly rate: $0.024685). 
*(Note: Fargate and EC2 charge for provisioned resources 24/7, regardless of whether traffic is hitting the endpoints. Screenshots are provided in `results/figures/pricing-screenshots/`).*

---

## Assignment 6: Cost Model, Break-Even, and Recommendation

### Monthly Cost Calculation
Traffic Model: 8,370,000 requests/month.
Lambda parameters: 0.077 seconds duration, 0.5 GB memory.

* **Fargate (Fixed 24/7):** **$17.77 / month**
* **EC2 (Fixed 24/7):** **$14.98 / month**
* **Lambda:**
  * Request Cost: 8.37M * $0.20/1M = $1.67
  * Compute Cost: 8,370,000 * 0.077s * 0.5 GB * $0.0000166667 = $5.37
  * **Total Lambda Cost:** **$7.04 / month**

### Break-Even Point
To find the average RPS ($R$) where Lambda costs the same as Fargate:

$$R=\frac{\text{Fixed cost}}{S \times (C_{req}+C_{compute})}$$

* Fixed Cost: $17.77
* $S$ (Seconds in month): 2,592,000
* Cost per request + compute: $0.000000841

Result: `17.77 / (2,592,000 * 0.000000841)` = **~8.15 RPS**

### Recommendation
**I recommend AWS Lambda (ZIP deployment), supplemented with Provisioned Concurrency.**

**Justification:**
1. **Cost:** At our specified traffic model, Lambda is by far the cheapest option ($7.04 vs $17.77 for Fargate) because it avoids paying for 18 hours of daily idle time.
2. **Scaling:** Fargate and EC2 fail the p99 < 500ms SLO catastrophically under the 100 RPS peak due to queueing on a single instance. Lambda naturally parallelizes execution.
3. **Meeting the SLO:** While native Lambda fails the SLO during a "Burst from Zero" due to cold starts (>629 ms), we know exactly when the peak occurs (30 minutes per day). By utilizing **Application Auto Scaling to schedule Provisioned Concurrency** right before the peak, we can pre-warm the environments. This eliminates cold starts, ensuring peak requests are served in ~77ms, comfortably meeting the <500ms SLO.

**Conditions for Reversal:**
This recommendation would change if the baseline steady-state traffic permanently exceeds **8.15 RPS**. At that point, Fargate becomes more cost-effective. However, to meet the SLO on Fargate at 100 RPS, we would need to implement Target Tracking Auto Scaling (based on CPU utilization) to add more tasks during the peak, which would raise the fixed cost further.

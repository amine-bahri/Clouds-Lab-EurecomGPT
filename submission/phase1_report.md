# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
50
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->236.3<!--/answer--> | <!--answer:vm_warm_median_ms-->210.7<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1844.3<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->634.1<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1706.0<!--/answer--> | <!--answer:function_warm_median_ms-->633.5<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
Warm median latency (bar length relative to the slowest), from measure.py:

```
IaaS VM    cold=  236.3 ms   warm min/median/p95/max = 209.7/210.7/212.4/212.9 ms
           #############
Cloud Run  cold= 1844.3 ms   warm min/median/p95/max = 122.0/634.1/683.7/685.4 ms
           ########################################
Function   cold= 1706.0 ms   warm min/median/p95/max = 119.8/633.5/651.9/684.8 ms
           ########################################
```
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
VM: nothing gets created. It's one e2-micro (2 shared vCPUs, 1 GB RAM) with gunicorn running 1 worker and 8 threads, so it handles 8 requests at a time and the rest wait. Scaling it would be manual (bigger VM, or more VMs behind a load balancer).

Cloud Run: it starts new container instances on its own and goes back to zero when idle. The limit is my --max-instances=3, with up to 80 requests per instance (but gunicorn still only runs 8 at a time).

Function: same autoscaling, but concurrency is 1, so every request needs its own instance. With --max-instances=3 only 3 requests run at once and the rest queue or fail.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
VM: the most work. I had to create a service account that can only pull from the registry, a custom VPC and subnet, two firewall rules (8080, and 22 only from the IAP range) and the VM. Then SSH in, install Docker, set up registry auth and start the container with --restart=always. No HTTPS, and keeping the OS updated is on me.

Cloud Run: once the image existed it was one command, and HTTPS, scaling and restarts come with it.

Function: one command from the source folder, no Dockerfile. Google builds the image, so the deploy is slower (a few minutes).

Before any build worked I had to give the compute service account the Cloud Build role, since it has no permissions by default on a new project.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-447015567467.us-central1.run.app
LAST DEPLOYED BY: service-447015567467@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-27T10:36:06.135469Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-447015567467.us-central1.run.app
LAST DEPLOYED BY: mohamed-amine.bahri@eurecom.fr
LAST DEPLOYED AT: 2026-10-27T10:34:12.325438Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
There are two services even though I only deployed echo-bot. The echo service was deployed by gcf-admin-robot, the Cloud Functions service account, so my gen 2 function is really just a Cloud Run service that Cloud Functions created for me. That's why the cold starts are so close (1844.3 ms for Cloud Run, 1706.0 ms for the function): in both cases Cloud Run is starting a container. The VM (236.3 ms) has no real cold start because its container is always running.

The real differences are in packaging and defaults. I wrote the Dockerfile and built the echo-bot image myself, while Google built the function's image from my source code. The function also uses concurrency 1 instead of 80, so it needs more instances under load.
<!--/answer-->

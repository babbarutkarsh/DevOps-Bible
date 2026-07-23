---
title: Behavioral & SRE Culture Rounds
nav_order: 95
description: "Behavioral, on-call, incident, and SRE-culture interview rounds — STAR framework, postmortem thinking, SLO discussions, and strong sample answers for senior DevOps/SRE loops."
---

# Behavioral & SRE Culture Rounds — DevOps Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents
- [Why These Rounds Matter](#why-these-rounds-matter)
- [What Interviewers Actually Assess](#what-interviewers-actually-assess)
- [The STAR Framework](#the-star-framework)
- [SRE Culture Concept Questions](#sre-culture-concept-questions)
- [Behavioral Questions with STAR Sample Answers](#behavioral-questions-with-star-sample-answers)
- [Questions to Ask Your Interviewer](#questions-to-ask-your-interviewer)
- [Red Flags to Avoid](#red-flags-to-avoid)
- [Key Resources](#key-resources)

---

## Why These Rounds Matter

At Senior/Staff+ DevOps/SRE levels, behavioral and culture-fit rounds carry **as much weight as technical rounds**. Companies want to know:

- Can you handle production incidents under pressure?
- Do you practice blameless postmortems?
- Can you balance reliability vs feature velocity?
- Will you improve on-call culture?
- Can you influence without authority across teams?
- Do you own your mistakes?

**Reality:** A candidate who aces systems design but fumbles "tell me about an outage" often gets rejected. Senior roles are about **judgment, ownership, and communication** as much as technical depth.

**Top companies that weight these heavily:**
- Google (SRE): Googleyness & Leadership, SRE-specific scenarios
- Amazon: Leadership Principles (14 principles, expect 5-6 stories)
- Meta: Behavioral emphasizes impact, ambiguity, collaboration
- Netflix: Culture fit, freedom & responsibility
- Stripe, Datadog, PagerDuty: Incident response, on-call culture

---

## What Interviewers Actually Assess

| Dimension | What They Look For | Red Flags |
|-----------|-------------------|-----------|
| **Ownership** | Takes responsibility, drives to resolution, follows up | Blames others, vague about your role |
| **Judgment under pressure** | Calm during incidents, prioritizes correctly, escalates when needed | Panics, makes rash decisions, doesn't involve others |
| **Blamelessness** | Focuses on systems/process, learns from failure | Blames individuals, defensive |
| **Communication** | Clear during incidents, writes good postmortems, explains to non-technical stakeholders | Jargon-heavy, poor storytelling |
| **Collaboration** | Works across teams, influences without authority, resolves conflict constructively | Lone wolf, dismisses others' concerns |
| **Impact** | Quantifies results (MTTR, toil reduction, uptime improvement) | Vague "we improved things" |
| **Growth mindset** | Learns from mistakes, adapts process, mentors | Repeats mistakes, rigid |

---

## The STAR Framework

**STAR = Situation, Task, Action, Result.** The gold standard for behavioral answers.

### Structure

```
Situation (15-20%)
  ↓ Set context concisely (1-2 sentences: company, team, system scale)
Task (10-15%)
  ↓ What was the problem/goal? Why did it matter?
Action (50-60%)
  ↓ What YOU did (not "we" — your specific actions, decisions, tradeoffs)
Result (15-20%)
  ↓ Quantified outcome + what you learned
```

### STAR Checklist

| Element | Good | Bad |
|---------|------|-----|
| **Situation** | "6-person SRE team, 500 microservices, 99.9% SLA, 50M requests/day" | "I was on a team and we had services" |
| **Task** | "P0 incident: 30% of checkout requests failing, $50K/min revenue impact" | "There was an outage" |
| **Action** | "I isolated the failing service via distributed tracing, rolled back the bad deploy, then led the postmortem" | "We fixed it" |
| **Result** | "MTTR 18 min, wrote runbook, reduced similar incidents 80% over next 6 months" | "Everything was fine after" |

### Common STAR Mistakes

1. **Too much "we," not enough "I"**: Interviewers want YOUR contributions. Say "I analyzed logs while my teammate handled comms" not "we looked at logs."
2. **No quantification**: "Improved reliability" → "Reduced incident count from 12/month to 3/month, improved p99 latency 40%."
3. **No learning/follow-up**: Always end with "Here's what I learned" or "We changed X process to prevent recurrence."
4. **Rambling situation**: Keep setup brief. Interviewers care about actions and results.
5. **Blaming others**: Even if someone caused the outage, frame it as "the deploy process lacked safeguards" not "Bob broke prod."

### Tailoring to Company Culture

**Amazon (Leadership Principles):** Map each story to 2-3 LPs. Example: "Ownership" (took responsibility for follow-up), "Bias for Action" (quickly rolled back), "Dive Deep" (root-caused via logs).

**Google (SRE):** Emphasize error budgets, SLOs, toil reduction, blameless postmortems, automation.

**Netflix:** Emphasize autonomy, high trust, taking calculated risks, candid communication.

**Stripe:** Emphasize user impact, clear communication, moving fast but reliably.

---

## SRE Culture Concept Questions

These test whether you **think like an SRE**. Interviewers want to see you internalize SRE principles, not just know definitions.

### 🔥 Q: What is a blameless postmortem? How do you run one?

**Definition:** A postmortem that focuses on **systems and processes**, not individuals. The goal is learning and prevention, not punishment.

**Structure:**
1. **Timeline** — Minute-by-minute what happened (detection, response, resolution)
2. **Root cause(s)** — Technical root cause + contributing factors (process gaps, lack of monitoring)
3. **Impact** — User-facing (requests failed, latency spike) + business (revenue, reputation)
4. **What went well** — Positive aspects (fast detection, good comms, effective rollback)
5. **What went wrong** — Gaps in tooling, process, documentation, testing
6. **Action items** — Concrete, assigned, with deadlines. Prioritize by impact.
7. **Lessons learned** — Broader takeaways

**How to run one:**
- Hold meeting 24-48 hours after incident (not immediately — people need rest)
- Facilitator ensures blamelessness (redirect "why did Alice deploy" to "why did our deploy process allow this")
- Document in shared location (wiki, incident management tool)
- Follow up on action items in sprint planning

**Example framing:** "We had a postmortem for a P0 database failover that took 45 minutes. Instead of asking 'why didn't the DBA notice,' we asked 'why didn't our automated failover work?' We found our health check was too lenient. The outcome: we tightened the check, added synthetic transactions, and reduced failover time to under 5 minutes."

### ⭐ Q: What is toil? How do you identify and reduce it?

**Definition (Google SRE Book):** Work tied to running a service that is **manual, repetitive, automatable, tactical, lacks enduring value, and scales linearly with service growth.**

**Examples of toil:**
- Manually restarting services
- Copy-pasting config changes across environments
- Manually approving deploy tickets
- Chasing teams for status updates

**Not toil:**
- Writing automation (one-time, adds enduring value)
- Incident response (not repetitive, requires judgment)
- Capacity planning (strategic)

**How to identify:**
- Track where SRE time goes (surveys, time logs)
- Ask: "If traffic doubles, does this work double?"
- Red flag: Pager fatigue from repetitive manual fixes

**How to reduce:**
- Automate: Scripts, CI/CD, self-service portals
- Eliminate: Remove unnecessary approvals, deprecate legacy systems
- Delegate: Move toil to product teams ("you build it, you run it")

**Target:** SRE teams should spend <50% of time on toil, ideally 30-40%.

**Example framing:** "On my last team, we spent 10 hours/week manually scaling Redis clusters. I wrote a Terraform module + autoscaling policy. Toil dropped to zero, and we redirected that time to improving observability. Over 6 months, we reduced MTTR by 30%."

### 🔥 Q: What is an SLO? How do you set one? What happens when the error budget is exhausted?

**SLO (Service Level Objective):** Internal target for reliability. Tighter than SLA (customer-facing contract).

**Example:**
- SLA: 99.9% uptime (contractual)
- SLO: 99.95% uptime (internal target)
- SLI (Service Level Indicator): Measured uptime (99.97% last month)

**How to set an SLO:**
1. **Choose SLI(s):** Availability, latency (p50/p95/p99), error rate, throughput
2. **Pick target:** Based on user expectations + cost of reliability. **Don't pick 100%** (too expensive, slows innovation).
3. **Measure over window:** 30-day rolling window (not monthly reset — avoids end-of-month gaming)
4. **Calculate error budget:** If SLO is 99.9%, error budget is 0.1% = 43 minutes/month downtime allowed

**Error budget exhausted:**
- **Freeze feature releases** until reliability improves
- Redirect eng effort to reliability work (fix flaky tests, improve monitoring, reduce tech debt)
- Postmortem on what consumed budget (repeated incidents? single large outage?)

**Negotiating with product:** "We've burned 80% of our error budget in 2 weeks due to 3 incidents caused by rushed releases. I propose we pause feature work for 1 sprint to harden the deploy pipeline and add integration tests. Otherwise we risk missing SLA and customer escalations."

**Example framing:** "We set a 99.9% availability SLO for our API (measured via synthetic checks every minute). Last quarter we hit 99.87% due to a database incident. We used the error budget burn to justify 2 sprints of reliability work: adding read replicas and improving our failover automation. Next quarter we hit 99.94%."

### 🔥 Q: What makes a healthy on-call rotation?

**Healthy on-call:**
- **Manageable load:** <2 pages/night on average. More = alert fatigue.
- **Clear escalation:** If you can't resolve in 30 min, who do you escalate to?
- **Runbooks:** Every common alert has a runbook (symptoms, impact, mitigation, escalation).
- **Blameless:** Mistakes during incidents aren't punished.
- **Compensated:** Rotation pay or time off in lieu.
- **Follow-the-sun:** For global teams, hand off at end of shift (no one on-call 24/7).
- **Post-incident learning:** Every incident gets a postmortem and action items.

**Red flags:**
- Always paging the same person (silos)
- Alerts without runbooks
- "Cry wolf" alerts (false positives, low-signal)
- No post-incident review

**Example framing:** "On my last team, we had 5-6 pages/night, mostly false positives. I led an effort to categorize alerts, tune thresholds, and consolidate related alerts. We dropped to 0.5 pages/night, and engineer satisfaction surveys improved 40%. We also instituted a 'no runbook, no page' rule — if an alert fires without a runbook, we create one before the next on-call shift."

### ⭐ Q: What is incident command? Describe the roles.

**Incident Command System (ICS):** Borrowed from emergency response. Clear roles avoid chaos during incidents.

**Core roles:**

| Role | Responsibilities | Who |
|------|------------------|-----|
| **Incident Commander (IC)** | Owns incident, makes decisions, delegates, declares resolution | Senior SRE/eng |
| **Communications Lead** | Updates stakeholders (internal status page, Slack, customers), writes timeline | Eng manager or senior eng |
| **Operations Lead** | Executes technical fixes (rollback, failover, config changes) | On-call engineer |
| **Subject Matter Expert (SME)** | Deep expertise on specific system (database, network, etc.) | Team member or escalation |

**IC key skills:**
- **Prioritization:** "Fix auth service first, logging can wait"
- **Delegation:** "Alice, check DB replication lag. Bob, prepare rollback."
- **Decision-making:** "We don't have root cause yet, but we're rolling back to stop the bleeding."
- **Communication:** "Update status page every 15 min, even if no change."

**Severity levels:**

| Severity | Impact | Response | Example |
|----------|--------|----------|---------|
| **P0/SEV1** | Service down, major revenue impact | All hands, exec notification | Checkout broken, login failing |
| **P1/SEV2** | Partial outage, degraded performance | On-call + IC + SME | 10% of requests failing, p99 latency 5x |
| **P2/SEV3** | Minor degradation, limited user impact | On-call engineer | Single region slow, non-critical feature broken |
| **P3/SEV4** | No user impact, internal tooling | Ticket for business hours | Monitoring dashboard broken |

**Example framing:** "During a P0 database failover incident, I was IC. I delegated ops to the on-call engineer (execute failover), pulled in a DBA as SME (check replication lag), and assigned a comms lead to update stakeholders every 10 minutes. We restored service in 18 minutes, MTTR was 40% faster than our previous incident because of clear roles."

### 🔥 Q: How do you balance reliability vs velocity? (Reliability vs feature development)

**Key insight:** **100% reliability is not the goal.** Every 9 of uptime (99.9% → 99.99%) roughly doubles cost and slows feature delivery.

**Framework:**
1. **Set explicit SLO** based on user expectations, not perfection.
2. **Error budget** is the "budget" for risk-taking. If budget remains, you can move fast. If exhausted, slow down and fix reliability.
3. **Negotiate with product:** "We're at 99.97%, target is 99.9%. We have budget to ship this feature with moderate testing. If we hit 99.89%, we freeze features next sprint."
4. **Risk assessment:** New feature = risk. If error budget is tight, defer or add safeguards (feature flag, canary deploy, extra monitoring).

**Example tradeoffs:**
- **High velocity, lower reliability:** Startup, feature/market fit phase → Target 99.5%, accept more incidents
- **High reliability, moderate velocity:** Fintech, healthcare → Target 99.95%+, rigorous testing

**Example framing:** "Product wanted to ship a new payment flow that required database schema changes. We were at 99.92% uptime (target 99.9%). I proposed a canary deploy to 5% of traffic for 48 hours + rollback plan. Feature shipped on time, no incidents. When we later burned error budget on an unrelated outage, I used it to justify pausing the next feature to add integration tests, and product agreed."

### ⭐ Q: What is "you build it, you run it"?

**Model:** Product engineering teams are responsible for their services in production — on-call, incidents, operational toil.

**Benefits:**
- Eng teams feel pain of poor operational design → build better systems
- Faster feedback loop (no "throw over wall to ops")
- SRE teams focus on platform, tooling, consulting

**Challenges:**
- Engineers need on-call training, tooling, runbooks
- Risk of burnout if not managed well
- Some teams resist operational responsibility

**SRE role in this model:** Embedded consultant, platform builder, escalation point.

**Example framing:** "We transitioned to 'you build it, you run it' for our API team. I ran a 2-day on-call training (incident response, postmortems, monitoring). I also built self-service dashboards and runbooks. The first month was rocky, but after 3 months, MTTR dropped 25% because engineers could debug their own services faster than waiting for SRE escalation."

### 💡 Q: What is chaos engineering and when would you use it?

**Chaos engineering:** Proactively inject failures in production (or staging) to verify resilience. "Break things on purpose before they break by accident."

**Examples:**
- **Chaos Monkey** (Netflix): Randomly terminate instances
- **Latency injection:** Add 500ms delay to dependency
- **Failure injection:** Simulate database failover, network partition, disk full

**When to use:**
- After architecting for resilience (circuit breakers, retries, timeouts) — validate it works
- Before major events (Black Friday, launch)
- Regularly (monthly game days) to avoid "resilience decay"

**Prerequisites:**
- **Observability:** Can you detect the failure?
- **Blast radius control:** Start small (canary, single AZ)
- **Rollback plan:** Kill switch to stop experiment

**Example framing:** "We had built retry logic and circuit breakers for our payment service, but never tested them. I proposed a chaos experiment: inject 50% failure rate into the upstream fraud-check API for 10 minutes in staging. We discovered our circuit breaker threshold was too high, causing cascading timeouts. We fixed it before prod. Two months later, the fraud-check API had a real outage and our circuit breaker worked perfectly — zero user impact."

---

## Behavioral Questions with STAR Sample Answers

These are **sample STAR templates**. Personalize them with your own experiences, numbers, and company context.

### 🔥 Q: Tell me about a major outage/incident you handled.

**Sample STAR (P0 database outage):**

> **Situation:** I was on-call for a 12-person engineering team running an e-commerce checkout service handling 10M transactions/day with a 99.95% SLA. We used a PostgreSQL primary-replica setup on AWS RDS.
>
> **Task:** At 2 AM, PagerDuty alerted me: 40% of checkout requests failing with database timeout errors. Revenue impact was $80K/minute. I was incident commander.
>
> **Action:**
> - **0-5 min:** I acknowledged the page, checked Datadog — database CPU at 98%, query latency spiked 20x. I posted in the incident Slack channel: "P0 incident, checkout DB overloaded, investigating."
> - **5-15 min:** I assigned a comms lead to update the status page and product/exec stakeholders every 10 minutes. I checked recent deploys — no code changes. I queried RDS slow query log and found a new analytics query (added by a data analyst 2 days prior) doing full table scans on a 500M-row orders table.
> - **15-25 min:** I killed the long-running queries manually via `pg_terminate_backend`. Checkout latency returned to normal. I added a temporary block on that analytics query.
> - **25-45 min:** I stayed online monitoring for 20 minutes to ensure stability, then declared incident resolved.
>
> **Result:**
> - **MTTR: 25 minutes.** User-facing downtime: 15 minutes (some requests queued and succeeded).
> - **Postmortem:** I led a blameless postmortem 2 days later. Root cause: no query review process for analysts. Contributing factors: missing query timeout, no read replica for analytics.
> - **Action items:** (1) Set statement_timeout=30s on prod DB. (2) Provisioned a read replica for analytics queries. (3) Created a query review checklist for data team.
> - **Long-term impact:** Zero similar incidents over the next year. Analysts adopted the read replica, reducing prod DB load by 20%.
> - **Learning:** I learned to (1) involve stakeholders early (comms lead freed me to focus on technical work), (2) document kill-switch procedures (we now have a runbook for killing queries), and (3) prioritize systemic fixes over blaming individuals (the analyst didn't know the query would cause an outage).

### ⭐ Q: Tell me about a time you disagreed with a teammate or manager.

**Sample STAR (disagreement on deploy timing):**

> **Situation:** I was a senior SRE at a fintech company with a 99.99% uptime SLA. Our eng manager wanted to ship a new loan approval feature on Friday afternoon before a 3-day weekend.
>
> **Task:** I believed deploying Friday afternoon was high-risk: reduced staffing over the weekend, and the feature involved database schema changes. I needed to convince the manager to delay.
>
> **Action:**
> - I scheduled a 15-minute call and presented data: "Our last 3 Friday deploys had incidents (10%, 5%, and 15% error rate spikes). Average MTTR on weekends is 2x higher due to reduced staff. This deploy changes the schema, which is our #1 incident category."
> - I proposed an alternative: deploy Tuesday morning, when we're fully staffed and can monitor for 8 hours before EOD.
> - The manager was concerned about missing a board meeting deadline. I suggested a compromise: deploy to staging Friday, validate over the weekend with synthetic tests, deploy to prod Tuesday. We could still report "feature complete" to the board.
>
> **Result:**
> - The manager agreed. We deployed Tuesday, discovered a schema migration issue in the first hour (would have been a P0 outage), rolled back, fixed it, and redeployed successfully by noon.
> - **Learning:** I learned to (1) lead with data (historical incident rate), (2) propose alternatives (not just say "no"), and (3) understand stakeholder constraints (board deadline). The manager later thanked me and instituted a "no Friday deploy" policy.

### 🔥 Q: Tell me about a time you made a mistake that caused an incident.

**Sample STAR (bad config change):**

> **Situation:** I was a senior DevOps engineer managing Kubernetes clusters for a SaaS company (200 microservices, 50 engineers). We used Helm for deployments.
>
> **Task:** I was updating the resource limits for a high-traffic API service to reduce cost. I changed the memory limit from 2Gi to 1Gi based on observed usage (averaging 800Mi).
>
> **Action (the mistake):**
> - I applied the change via Helm upgrade directly to production (we had staging, but I skipped it because "it's just a resource limit").
> - Within 5 minutes, the service started OOMKilling (out-of-memory) under load. Traffic spikes hit 1.2Gi. 30% of requests failed.
>
> **Action (the fix):**
> - I immediately rolled back the Helm release (1 minute). Service recovered.
> - I checked metrics — p95 memory was 1.1Gi, not 800Mi average. I had looked at the wrong metric.
> - I wrote a postmortem: root cause was my mistake (wrong metric, skipped staging). Contributing factor: no memory headroom policy.
>
> **Result:**
> - **Downtime: 6 minutes.** No customer escalations (brief enough).
> - **Action items:** (1) I created a policy: resource limits = p99 usage + 30% headroom. (2) I added a CI check: Helm changes must be tested in staging first. (3) I ran a team training on "guardrails for config changes."
> - **Learning:** I learned to (1) own mistakes immediately (I posted in Slack "I caused this, rolling back"), (2) treat config changes with same rigor as code changes (they're just as risky), and (3) always look at p95/p99, not averages. My manager appreciated the transparent postmortem, and we avoided similar mistakes team-wide.

### ⭐ Q: Tell me about a time you handled being paged repeatedly or dealt with burnout.

**Sample STAR (alert fatigue):**

> **Situation:** I was on a 5-person SRE team supporting 30 microservices. On-call rotation was 1 week per person. I was getting paged 8-10 times per night, mostly for low-priority alerts (disk space warnings, flaky health checks).
>
> **Task:** I was burning out — poor sleep, dreading on-call weeks. I needed to reduce alert volume without missing real incidents.
>
> **Action:**
> - I categorized alerts over 2 weeks: 60% were "disk >80%" (not urgent), 20% were flaky health checks (false positives), 15% were real incidents, 5% were low-priority warnings.
> - I proposed a 3-part plan: (1) Change disk alerts to page only at >90% and after 15 min sustained (not 1 min spike). (2) Fix flaky health checks (add retry logic, increase timeout). (3) Route low-priority alerts to Slack, not PagerDuty.
> - I implemented the changes over 1 sprint. I also proposed adding a 6th person to the rotation to reduce frequency.
>
> **Result:**
> - **Pages dropped from 8-10/night to 0.5/night.** Incidents were still caught (no increase in MTTR).
> - Team morale improved — retro survey showed 50% improvement in "on-call experience."
> - We hired a 6th SRE, rotating every 6 weeks instead of 5.
> - **Learning:** I learned to (1) quantify the problem (categorize alerts), (2) distinguish signal from noise (not all alerts are equal), and (3) advocate for team health (burnout reduces effectiveness). I also learned that "reducing pages" is not the same as "reducing reliability" — better signal improves reliability.

### ⭐ Q: Tell me about a time you drove a cross-team project or influenced without authority.

**Sample STAR (standardizing CI/CD):**

> **Situation:** I was a staff DevOps engineer at a company with 8 product teams, each using different CI/CD tools (Jenkins, CircleCI, GitLab CI, GitHub Actions). This caused toil for the platform team (me + 3 others): we had to support 4 tools, and deployments were inconsistent.
>
> **Task:** I wanted to standardize on GitHub Actions, but I had no authority over product teams. I needed buy-in from 8 eng managers and 50+ engineers.
>
> **Action:**
> - **Research (Week 1):** I surveyed each team's pain points. Common themes: slow builds, hard to debug, no shared workflows.
> - **Prototype (Week 2-3):** I built a GitHub Actions reference pipeline for one team, including Docker build caching, secrets management, and PR preview environments. I documented it in a wiki.
> - **Advocacy (Week 4-6):** I presented at engineering all-hands (10 min demo), ran "office hours" for teams to migrate, and created migration guides. I emphasized benefits: 50% faster builds, easier debugging, shared workflow templates.
> - **Incentive:** I got director approval to prioritize platform support for GitHub Actions and deprioritize legacy tools (carrot + stick).
>
> **Result:**
> - **6 of 8 teams migrated within 2 months.** The other 2 had legacy constraints but agreed to GitHub Actions for new services.
> - **Platform toil dropped 40%** (we decommissioned 2 tools).
> - **Build times improved 30-50%** across teams due to shared caching.
> - **Learning:** I learned to (1) start with user needs (not "I think we should"), (2) build a working prototype (show don't tell), (3) make it easy (guides + office hours), and (4) align incentives (leadership support for deprecating old tools).

### ⭐ Q: Tell me about a time you pushed back on shipping something unreliable.

**Sample STAR (risky feature launch):**

> **Situation:** I was a senior SRE at a B2B SaaS company with 500 enterprise customers. Product wanted to launch a new real-time notification feature (WebSockets) for an upcoming sales conference (3 weeks away).
>
> **Task:** I reviewed the design and found concerns: no load testing, single point of failure (one WebSocket server), no graceful degradation if the feature failed. Shipping as-is risked a P0 outage during the conference (high visibility).
>
> **Action:**
> - I met with the PM and tech lead, presented risks: "If the WebSocket server crashes under load, the entire app could become unresponsive (tight coupling). We don't know the load profile. We have no rollback plan."
> - I proposed a phased approach: (1) Launch behind a feature flag (kill switch). (2) Load test with 2x expected traffic. (3) Add fallback: if WebSocket fails, degrade to polling. (4) Deploy to 10% of users first (canary).
> - The PM was resistant ("conference deadline"). I framed it as risk mitigation: "If we have an outage during the conference, it's worse than launching 1 week late."
> - I escalated to the eng director, who agreed with my plan.
>
> **Result:**
> - We implemented the safeguards (took 1 extra week). During load testing, we found a memory leak that would have caused a P0 outage. We fixed it.
> - We launched via canary to 10% of users for 3 days (no issues), then 100% the day before the conference.
> - Conference went smoothly. PM later thanked me: "You saved us from a disaster."
> - **Learning:** I learned to (1) quantify risk (not just say "it's risky"), (2) propose alternatives (not just block), (3) escalate when needed (director had PM's ear), and (4) build trust by being right (next time the PM proactively asked for my review).

### 🔥 Q: Tell me about a time you automated away significant toil.

**Sample STAR (auto-scaling automation):**

> **Situation:** I was a DevOps engineer at a media streaming company. We had 50 microservices on Kubernetes, and our SRE team (4 people) spent 15 hours/week manually adjusting replica counts based on traffic patterns (mornings, evenings, weekends).
>
> **Task:** I wanted to eliminate this toil and improve cost efficiency (we over-provisioned to avoid manual scaling).
>
> **Action:**
> - **Week 1:** I analyzed traffic patterns using Datadog metrics. I found predictable patterns: 3x traffic 6-10 PM, 0.5x traffic 2-6 AM.
> - **Week 2:** I implemented Kubernetes Horizontal Pod Autoscaler (HPA) for 10 high-traffic services, tuning CPU/memory thresholds.
> - **Week 3:** I added KEDA (event-driven autoscaling) for queue-backed services (scale based on queue depth, not just CPU).
> - **Week 4:** I wrote runbooks for edge cases (manual scaling for launch events) and trained the team.
>
> **Result:**
> - **Toil dropped from 15 hours/week to <1 hour/week** (only for special events).
> - **Cost savings: 25%** (we reduced over-provisioning, scaled down during low-traffic hours).
> - **Reliability improved:** Automated scaling reacted faster than humans (MTTR for load-related incidents dropped 40%).
> - **Learning:** I learned to (1) measure toil before automating (15 hours/week quantified the ROI), (2) iterate (start with 10 services, not 50), and (3) document edge cases (automation isn't 100%, need runbooks for exceptions).

### ⭐ Q: Tell me about a time you dealt with an ambiguous problem.

**Sample STAR (intermittent latency spikes):**

> **Situation:** I was a senior SRE at a fintech company. Customer support reported intermittent API slowness (5-10 second responses) for 2 weeks, but we had no incident alerts (p50 latency was normal, only p99 spiked).
>
> **Task:** The problem was ambiguous: sporadic, no clear pattern, hard to reproduce. I needed to identify the root cause.
>
> **Action:**
> - **Week 1:** I correlated logs, metrics, and traces. I ruled out: (1) Database (query times normal), (2) Network (no packet loss), (3) Code deploys (no recent changes).
> - **Week 2:** I noticed a pattern: spikes occurred during specific customer actions (large file uploads). I checked S3 upload logic — we were uploading synchronously in the API request (blocking).
> - **Week 2:** I ran a load test with large files (20MB+) and reproduced the issue. S3 uploads took 8-12 seconds, blocking the API thread.
> - **Week 3:** I refactored the code to upload asynchronously (background job). I deployed behind a feature flag and tested with the affected customers.
>
> **Result:**
> - **p99 latency dropped from 8s to 400ms.**
> - **Customer complaints dropped to zero.**
> - **Learning:** I learned to (1) look beyond aggregated metrics (p50 was fine, p99 told the story), (2) involve users (CSM helped identify "large file uploads" pattern), and (3) reproduce before fixing (load test confirmed hypothesis). I also added a p99 latency alert to catch similar issues earlier.

### 💡 Q: Tell me about a time you mentored someone or improved team practices.

**Sample STAR (on-call training program):**

> **Situation:** I was a staff SRE on a 10-person team. We hired 3 junior engineers who had never been on-call. Our existing "training" was: read the runbooks, shadow for 1 week, then you're on-call.
>
> **Task:** The junior engineers were anxious and underprepared. I wanted to create a structured on-call training program.
>
> **Action:**
> - **Week 1:** I designed a 4-week training program: (1) Week 1: Observability 101 (Datadog, logs, traces). (2) Week 2: Incident response (STAR framework, postmortems). (3) Week 3: Runbook deep-dive (walk through top 10 alerts). (4) Week 4: Shadow 2 real incidents.
> - **Week 2-5:** I ran weekly 1-hour sessions (live demos, Q&A).
> - **Week 6:** Each trainee handled a simulated incident (I injected a staging failure, they debugged and resolved).
> - **Week 7:** They joined the on-call rotation with a "backup" (me or another senior) for 2 weeks.
>
> **Result:**
> - **All 3 juniors successfully handled their first real incidents** (MTTR within team average).
> - **Confidence improved:** Retro survey showed they felt "well-prepared."
> - **Scaled:** I documented the program in the wiki. We used it for the next 5 hires.
> - **Learning:** I learned to (1) structure training (not ad-hoc shadowing), (2) include hands-on practice (simulated incidents), and (3) provide backup (safety net for first real incidents). The program became a team standard.

### ⭐ Q: Tell me about a time you prioritized under conflicting pressures.

**Sample STAR (multiple P1 incidents):**

> **Situation:** I was on-call as a senior SRE during a major product launch. We had 3 simultaneous P1 incidents: (1) Payment API 10% error rate. (2) Notification service delays (emails 2 hours late). (3) Admin dashboard down (internal tool).
>
> **Task:** I was the only SRE online (2 AM). I needed to triage and prioritize.
>
> **Action:**
> - **Triage (2 min):** I assessed impact:
>   - Payment API: User-facing, revenue impact ($20K/hour), 10% of users affected.
>   - Notifications: User-facing, no revenue impact, poor UX but not blocking.
>   - Admin dashboard: Internal tool, 5 CSMs affected, workaround available (direct DB queries).
> - **Prioritization:** I declared payment API as P0 (highest impact), started incident response. I posted in Slack: "Payment API is priority 1, investigating. Notifications + admin dashboard are next."
> - **Action:** I rolled back the payment API deploy (recent change). Error rate dropped to 0% in 3 minutes. I then fixed the notification service (config error, 5 min fix). I delegated the admin dashboard to the on-call dev (CSMs used workaround for 30 min).
>
> **Result:**
> - **Payment API MTTR: 8 minutes.** Notifications MTTR: 13 minutes. Admin dashboard MTTR: 35 minutes.
> - **Customer escalations: Zero** (fast resolution).
> - **Postmortem:** We added deploy safeguards (canary) and improved on-call staffing (2 people for launches).
> - **Learning:** I learned to (1) triage ruthlessly (revenue + user impact first), (2) communicate priorities (Slack update set expectations), and (3) delegate when possible (admin dashboard didn't need senior SRE).

---

## Questions to Ask Your Interviewer

Asking good questions signals seniority, curiosity, and culture fit. Tailor to the role and company.

### On-Call & Incident Culture
- "What does a typical on-call week look like? How many pages per week on average?"
- "Walk me through your most recent major incident. How was it handled?"
- "Do you do blameless postmortems? Can you share an example?"
- "What's your MTTR for P0 incidents? How has it trended over the past year?"

### Reliability & SRE Practices
- "How do you set SLOs? Do you use error budgets to make ship/no-ship decisions?"
- "What percentage of SRE time is spent on toil vs project work?"
- "Do you practice chaos engineering or game days?"
- "What's your deploy frequency? Do you do canary deployments?"

### Team & Growth
- "What does the SRE team's relationship with product engineering look like? Embedded or centralized?"
- "What's the most challenging reliability problem the team is working on right now?"
- "How do you handle on-call burnout or alert fatigue?"
- "What opportunities are there for senior ICs to mentor or lead initiatives?"

### Red Flags to Listen For
- **"We page people for everything"** → Alert fatigue, no prioritization
- **"We don't really do postmortems"** → No learning culture
- **"Incidents are rare because we move slowly"** → Risk-averse, bureaucratic
- **"On-call is handled by one person"** → Siloed, single point of failure
- **"We aim for 100% uptime"** → Unrealistic expectations, fear of failure

---

## Red Flags to Avoid

Behaviors that signal you're **not ready for senior/staff SRE**:

| Red Flag | Why It's Bad | Fix |
|----------|--------------|-----|
| **Blaming individuals** | "My teammate broke prod" | Frame as systems failure: "Our deploy process lacked safeguards" |
| **Vague about your role** | "We fixed it" (no "I") | Be specific: "I rolled back, while Alice handled comms" |
| **No quantification** | "I improved reliability" | "I reduced incident count from 12/month to 3/month" |
| **No learning/follow-up** | Story ends at "fixed it" | Always add: "We changed X process to prevent recurrence" |
| **Defensive about mistakes** | "It wasn't my fault" | Own it: "I caused this by skipping staging. Here's what I learned." |
| **Lone wolf** | "I solved it alone" | Senior roles require collaboration: "I pulled in a DBA to advise" |
| **Rambling** | 5-minute answer with no structure | Use STAR, keep under 2 minutes |
| **Jargon without explanation** | "We used Istio for mTLS" (interviewer may not know Istio) | "We used a service mesh for encrypted service-to-service traffic" |

---

## Key Resources

- **Google SRE Book** (free online) — https://sre.google/sre-book/table-of-contents/
  - Chapters 15 (Postmortem Culture), 3 (Embracing Risk), 4 (Service Level Objectives), 5 (Eliminating Toil)
- **Google SRE Workbook** — https://sre.google/workbook/table-of-contents/
  - Chapters on SLOs, Error Budgets, Incident Response
- **Amazon Leadership Principles** — https://www.amazon.jobs/content/en/our-workplace/leadership-principles
  - Memorize these if interviewing at Amazon (Ownership, Bias for Action, Dive Deep, Deliver Results most relevant)
- **The Art of the Postmortem** — https://www.pagerduty.com/resources/learn/post-mortem-incident-report/
- **PagerDuty Incident Response Docs** — https://response.pagerduty.com/
  - Incident command, roles, severity levels
- **"Site Reliability Engineering" (O'Reilly book)** — Print version of SRE Book
- **Charity Majors' blog (Honeycomb)** — https://charity.wtf/
  - On-call, observability, learning from incidents
- **"Seeking SRE" (O'Reilly book)** — Essays from SRE practitioners
- **Netflix Tech Blog** — https://netflixtechblog.com/
  - Chaos engineering, culture

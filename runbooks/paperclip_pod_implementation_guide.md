# Paperclip Pod Implementation Guide

> Purpose: Turn the existing specialist-agent roster into three practical coordination pods without creating unnecessary management layers.
>
> Starting point: CEO remains the single strategic coordinator. Pods standardize handoffs and routing; they do not grant broad new authority.

## Pod Design

| Pod | Members | Primary output to CEO |
| --- | --- | --- |
| Product | Product Manager, Business Analyst, User Experience, QA | Decision-ready product brief |
| Engineering | Founding Engineer, Coding, DevOps/Release, Security | Implementation readiness brief |
| Learning | Research/Generalist, Prompt Engineer, Documentation/Knowledge Curator | Evaluation and learning brief |

## Step 1: Create Three Pod Charters

Create three documentation-only Markdown files or Paperclip reference documents:

- `product-pod-charter.md`
- `engineering-pod-charter.md`
- `learning-pod-charter.md`

Each charter must state:

- Pod purpose.
- Member roles.
- Inputs the pod may use.
- Required output artifact.
- Standard handoff format.
- Explicit authority boundaries.
- Escalation rule: CEO resolves cross-pod conflicts, priority decisions, and requests for broader access.

### Product Pod Charter

```md
# Product Pod Charter

## Purpose
Turn an approved company goal into a small, user-centered, testable work package.

## Members
Product Manager, Business Analyst, User Experience, QA.

## Required inputs
Approved goal, supplied internal context, prior decisions, and approved research.

## Required output
One Product Decision Brief for CEO.

## Boundaries
Members create drafts, analysis, and recommendations only. CEO creates or assigns official tasks. No member changes product systems or uses unapproved external sources.
```

### Engineering Pod Charter

```md
# Engineering Pod Charter

## Purpose
Turn an approved Product Decision Brief into an implementation-ready plan with safety, test, release, and rollback considerations.

## Members
Founding Engineer, Coding, DevOps/Release, Security.

## Required inputs
Approved Product Decision Brief, supplied repository context, and approved technical constraints.

## Required output
One Implementation Readiness Brief for CEO.

## Boundaries
No merge, production deployment, secret access, destructive operations, or unapproved infrastructure changes. CEO resolves priority and approval gates.
```

### Learning Pod Charter

```md
# Learning Pod Charter

## Purpose
Turn supplied agent/model outcomes into reusable evaluation assets, lessons, and durable documentation.

## Members
Research/Generalist, Prompt Engineer, Documentation/Knowledge Curator.

## Required inputs
Supplied prompts, outputs, run summaries, rubrics, and approved internal documents.

## Required output
One Evaluation and Learning Brief for CEO.

## Boundaries
Read-only analysis plus documentation-only writes. No model changes, new-agent creation, task assignment, production prompt changes, or system modifications.
```

## Step 2: Add Pod Rules to Each Agent

Open each agent’s managed instructions and append only the pod-specific rule set that applies to it. Do not replace the agent’s existing role boundaries.

### Product Pod Rules

Append to Product Manager, Business Analyst, UX, and QA instructions:

```md
## Product Pod Coordination

You are a member of the Product Pod.

- Work from the same supplied goal and source packet when assigned a pod task.
- Produce only your role-specific section; do not repeat other members’ work.
- Use the Product Decision Brief format for final handoff.
- State dependencies, open questions, and disagreements explicitly.
- Do not create or assign official tasks. CEO owns final prioritization and task assignment.
```

### Engineering Pod Rules

Append to Founding Engineer, Coding, DevOps/Release, and Security instructions:

```md
## Engineering Pod Coordination

You are a member of the Engineering Pod.

- Work from an approved Product Decision Brief or explicitly supplied technical task.
- Produce only your role-specific section of the Implementation Readiness Brief.
- State assumptions, dependencies, test needs, risk, and rollback or stop conditions where relevant.
- Do not merge, deploy to production, access secrets, or perform destructive actions unless explicitly approved through a separate task.
- Escalate unresolved product trade-offs to CEO.
```

### Learning Pod Rules

Append to Research/Generalist, Prompt Engineer, and Documentation/Knowledge Curator instructions:

```md
## Learning Pod Coordination

You are a member of the Learning Pod.

- Work only from supplied prompts, outputs, run summaries, rubrics, and approved internal documents.
- Produce only your role-specific section of the Evaluation and Learning Brief.
- Separate observed evidence, inference, recommendation, and uncertainty.
- Do not change models, prompts, agents, systems, task assignments, or production settings.
- Escalate evidence gaps or requests for experiments to CEO.
```

## Step 3: Create Standard Handoff Templates

Create the following Paperclip document templates. Every pod task must end with one of these artifacts, not a free-form discussion.

### Product Decision Brief

```md
# Product Decision Brief — [Initiative]

## Goal

## User and problem

## Smallest useful outcome

## Scope and non-goals

## Requirements

## Acceptance criteria

## UX findings

## QA test considerations

## Risks, dependencies, and open questions

## Recommended priority and next action for CEO
```

**Contribution order:** Product Manager drafts the brief; Business Analyst sharpens requirements and risks; UX adds journey/usability findings; QA adds test considerations. CEO reviews one final brief and decides whether to create work.

### Implementation Readiness Brief

```md
# Implementation Readiness Brief — [Initiative]

## Approved objective and acceptance criteria

## Proposed implementation approach

## Affected components and dependencies

## Change sequence

## Test plan

## Security review

## Release and rollback plan

## Risks, blockers, and approvals needed

## Recommended next action for CEO
```

**Contribution order:** Founding Engineer owns technical approach; Coding provides diff and test guidance; Security adds design-risk review; DevOps/Release adds CI, release, monitoring, and rollback notes. CEO approves execution and any authority escalation.

### Evaluation and Learning Brief

```md
# Evaluation and Learning Brief — [Experiment or Workflow]

## Question being evaluated

## Supplied evidence

## Rubric and method

## Findings

## Failure patterns

## Limitations and uncertainty

## Recommended follow-up experiment

## Documentation updates

## Recommended next action for CEO
```

**Contribution order:** Research/Generalist compares evidence; Prompt Engineer defines or improves the rubric and regression cases; Documentation Curator records the approved learning in durable documentation.

## Step 4: Use a Parent Issue for Each Pod Cycle

For any initiative that needs a pod, CEO creates one parent issue. The parent issue contains only:

- The goal.
- The supplied source packet or links to approved internal materials.
- The requested pod artifact.
- The decision CEO needs to make afterward.
- A completion deadline or heartbeat budget when appropriate.

Do not assign all agents the same broad prompt. Create one child issue per contributing role with a narrow deliverable.

### Product Pod Example

Parent issue: `Product pod: define the smallest useful Onboarding slice.`

Child issues:

1. Product Manager: draft goal, scope, non-goals, and proposed ticket slices.
2. Business Analyst: write requirements, acceptance criteria, assumptions, and risks.
3. UX: identify user journey, usability risks, and accessibility considerations.
4. QA: provide acceptance-test and regression considerations.

CEO then synthesizes or requests a final Product Decision Brief, makes the priority decision, and creates official execution tasks.

## Step 5: Sequence Pods Instead of Broadcasting Work

Use this default flow:

```text
CEO goal
  -> Product Pod: Product Decision Brief
  -> CEO approves scope and priority
  -> Engineering Pod: Implementation Readiness Brief
  -> CEO approves execution and gates
  -> Delivery work and evidence
  -> Learning Pod: Evaluation and Learning Brief
  -> CEO records decision and updates priorities
```

Run pods in parallel only when their inputs are independent. For example, Security and DevOps can review an approved technical approach in parallel; QA should not design final validation until the product acceptance criteria are stable.

## Step 6: Add Routing Labels or Prefixes

If Paperclip supports labels, create:

- `pod:product`
- `pod:engineering`
- `pod:learning`
- `handoff:ceo`
- `gate:approval-needed`
- `state:blocked`

If labels are unavailable, add the equivalent prefix to issue titles:

```text
[Product Pod] Define onboarding slice
[Engineering Pod] Review implementation readiness
[Learning Pod] Compare model outputs
```

Use one pod label per issue. A cross-pod issue should be a CEO parent issue with separate pod-specific child issues.

## Step 7: Establish a CEO Review Cadence

CEO should review only completed pod handoffs, not raw internal chatter.

For each brief, CEO records one of these decisions:

- `approve`: proceed to the next pod or execution.
- `revise`: identify the missing section or unresolved decision.
- `defer`: record why and when to revisit.
- `reject`: record why the work will not proceed.
- `block`: name the owner and the unblocking action.

Use this CEO comment template:

```md
## CEO Decision — [Initiative]

- Decision: [approve / revise / defer / reject / block]
- Reason:
- Next owner:
- Next action:
- Approval or constraint:
- Review date, if deferred:
```

## Step 8: Pilot Before Scaling

Run one pilot for each pod before enabling routine use.

### Product pilot

Use one small product goal. Success means a Product Decision Brief lets CEO create a small, testable execution issue without asking for clarification.

### Engineering pilot

Use one approved, low-risk change. Success means the Implementation Readiness Brief identifies implementation steps, tests, security concerns, release checks, and rollback conditions without authorizing deployment.

### Learning pilot

Use two supplied agent outputs and one rubric. Success means the Evaluation and Learning Brief separates evidence from inference and produces one reusable regression or documentation update.

## Step 9: Measure Pod Health

Review monthly or after five pod cycles.

| Signal | Healthy | Warning sign |
| --- | --- | --- |
| CEO clarification requests | Few, targeted | Repeated requests for missing basics |
| Duplicate work | Rare | Multiple agents produce the same artifact |
| Handoff quality | Clear decision and next action | Long summaries with no decision |
| Cycle time | Improves with reuse | Grows due to coordination overhead |
| Authority violations | None | Agents act outside stated boundaries |
| Artifact reuse | Templates and briefs reused | Every task starts from scratch |

If a pod repeatedly produces duplicate or low-value work, reduce its membership, tighten its template, or return to direct specialist tasks. A pod is a coordination tool, not a ceremonial committee.

## Step 10: Definition of Done

The pod structure is live when:

- [ ] The three charter documents exist.
- [ ] Every member has its pod coordination rules appended to managed instructions.
- [ ] The three handoff templates are available in Paperclip.
- [ ] CEO has created and reviewed one bounded pilot parent issue for each pod.
- [ ] Each pilot has produced one completed brief with a recorded CEO decision.
- [ ] No pod member received authority beyond its established guardrails.

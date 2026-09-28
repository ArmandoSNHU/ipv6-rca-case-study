# Structured Root-Cause Analysis Methodology

A reusable, general-purpose method for diagnosing failures in networked and cloud-connected systems. It is written to apply beyond any single incident: the goal is to move from a vague symptom to a proven root cause with the least wasted effort, and to leave behind documentation the next person can act on.

The method is deliberately ordered. Each step gates the next, so you spend effort where the evidence points rather than where the guess is loudest.

## 1. Define the problem

Write down what is actually wrong before touching anything.

- State the observable symptom in concrete terms (what is slow, what fails, what is unreachable) and separate it from assumed causes.
- Record scope: who is affected, which systems, since when, and whether it is constant or intermittent.
- Define the measurable end condition up front — the specific behavior that, once restored, means the incident is resolved. Without it, you cannot know when you are done.

## 2. Reproduce

- Confirm you can observe the failure on demand, or capture the conditions under which it occurs.
- Note what still works. A working comparison case (another app, another host, another path) is one of the most valuable diagnostic assets you have.
- If the problem is intermittent, look for the pattern that triggers it rather than treating each occurrence as isolated.

## 3. Isolate by layer

Work up the stack, ruling out the most local and least expensive explanations first. A common ordering:

- **Endpoint / workstation** — is the affected machine itself healthy (resources, general responsiveness, other applications)?
- **Local link / bandwidth** — is the local network segment saturated or degraded, or is other traffic on the same link fine?
- **Application vs. network** — is the application misbehaving, or is the network path underneath it failing? Separating these two questions early narrows the search dramatically.
- **Path and infrastructure** — name resolution, routing, address family (IPv4 vs. IPv6), firewalls, proxies, security controls, and the remote service itself.

Mapping observations to layers (OSI or a simpler client → link → path → service model) keeps the search organized and prevents re-checking the same thing twice.

## 4. Form and test hypotheses

- For each layer that is not yet cleared, state a hypothesis you can test, not just suspect.
- Prefer tests that produce evidence distinguishing one cause from another (for example, a test that behaves differently depending on address family, path, or service).
- Rule things out explicitly and record why. "Ruled out X because Y behaved normally" is as important as finding the cause.
- Let the evidence, not the initial hunch, decide which layer owns the problem.

## 5. Escalate with evidence

- When the cause lands in a system another team owns (security controls, firewalls, upstream network, the SaaS vendor), escalate — but escalate with a targeted, evidence-backed finding.
- Bring what you ruled out and why, plus the specific behavior that points at their layer. This lets the owning team make a precise change instead of guessing, and it shortens the fix.
- Own the escalation through to resolution rather than handing it off blind.

## 6. Remediate

- Coordinate the change with whoever owns the affected control or system.
- Keep the change as targeted as the evidence allows; a precise correction is easier to verify and easier to reverse if wrong.
- Record exactly what was changed.

## 7. Verify

- Retest against the measurable end condition defined in step 1. Do not assume the change worked.
- Confirm the original symptom is gone and that normal behavior has returned under the conditions that previously failed.
- Verifying after remediation is what distinguishes a fix from a coincidence.

## 8. Document

- Capture the root cause, the reasoning that led to it, the remediation, and the verification result.
- Record recovery steps so a recurrence can be resolved faster.
- Documentation is part of the fix, not an afterthought: it turns one person's investigation into the whole team's knowledge.

## Principles

- Cheapest and most local explanations first — but keep climbing the stack until the evidence stops.
- Distinguish "is it broken" from "is the path to it broken."
- Rule out explicitly; a documented negative is progress.
- Escalate with evidence, not a hunch.
- Verify before you call it done, and document so the next incident is shorter.

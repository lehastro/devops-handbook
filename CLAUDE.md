# DevOps Interview Prep & Senior Skill Growth

## Who is working on this

4-year DevOps engineer targeting senior level. Solid hands-on experience with daily DevOps tooling. Self-identified weak areas: **Linux internals** and **Kubernetes deep-dive**. Goal: fill fundamental gaps, be ready to answer any senior interview question, grow to senior SRE/DevOps level.

## What this project is

A structured knowledge base + interview prep system covering the full modern DevOps/SRE stack. Each topic has:
- Theory with internals (the "why", not just "what to run")
- How it works under the hood
- Failure modes and troubleshooting
- Interview Q&A with complete answers
- Practical labs and commands

## How to help in this project

When the user asks to work on a topic:
1. **Explain internals deeply** — assume the user knows what the tool does, explain how it works at the kernel/architecture level
2. **Use concrete examples** — real commands, real failure scenarios, real prod situations
3. **Senior interview framing** — answers should satisfy "walk me through what happens when X" style questions, not just definitions
4. **No hand-holding on basics** — the user has 4 years of experience, don't explain what a pod is, explain how the scheduler places it

When writing Q&A:
- Lead with the answer, then explain why
- Include what a weak answer looks like vs a strong answer
- Flag "mitigation-first" reflex for incident questions (reduce impact first, root-cause second)

## Content format (every topic file follows this)

```
# Topic Name

## How it works (internals)
...

## Key concepts
...

## Failure modes & troubleshooting
...

## Interview Q&A
### Q: [question]
**Weak answer:** ...
**Strong answer:** ...

## Practical: commands & labs
...
```

## Priority order (most important first)

1. **Linux** — gatekeeper topic at Google SRE and similar. Kernel-level reasoning.
2. **Kubernetes** — table stakes at senior level, but depth matters (not YAML syntax, internals)
3. **Networking** — both Linux networking and K8s networking deeply overlap with the above
4. **Observability** — SLOs, error budgets, Prometheus internals, distributed tracing
5. **System Design** — SRE/DevOps frame (reliability, blast radius, failure domains)
6. **Security** — zero trust, supply chain, secrets at scale
7. **IaC** — Terraform state management at scale, testing, GitOps workflows
8. **CI/CD / GitOps** — ArgoCD internals, canary/rollback, DORA metrics
9. **Cloud** — AWS/GCP architecture, failure domain reasoning
10. **Databases** — operational concerns: replication, failover, backup/restore

## Key themes from 2025-2026 senior interviews

- **"Mitigation-first" reflex**: every incident answer opens with reducing user impact, THEN root cause
- **Error budget thinking**: quantify reliability vs. velocity trade-offs numerically
- **Failure domain reasoning**: blast radius, SPOF, recovery path for every design
- **Kubernetes is table stakes**: not a differentiator, interviewers assume deep fluency and go to edge cases immediately
- **Walk-me-through questions**: expect to explain full internal flow (kubectl apply → running container, HTTP request → K8s pod, key press → user space)

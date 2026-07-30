---
name: Feature request
about: Suggest a new capability or flag for nigoh
title: "[Feature] "
labels: enhancement
---

**What problem does this solve?**
Describe the situation where the current tool falls short.

**Proposed behavior**
What would the new flag or behavior look like? Include an example invocation if applicable.

**Is this in scope?**
`nigoh` is deliberately a single-file, dependency-light nmap wrapper — not a general-purpose
scanning framework. Features requiring a persistent service, external database, or a new required
dependency beyond `nmap` are likely out of scope. See `RESEARCH_PLAN.md` for examples of features
that were considered and deliberately excluded, and why.

**Alternatives considered**

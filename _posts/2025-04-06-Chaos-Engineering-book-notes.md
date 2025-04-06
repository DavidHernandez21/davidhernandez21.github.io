---
layout: post
title:  "Chaos Engineering - System resilience in practice. Book notes"
---


# System resilience in practice by Casey Rosenthal and Nora Jones

# Book notes

## Dynamic safety model

### Safety

> Engineers tend to optimize for what they see. Since they have an intuition for the Economics and Workload properties, they see those properties in their day-to-day work. This inadvertently leads them further away from Safety. This effect sets up conditions for a silent, unseen drift towards failure as the success of their endevours provides opportunity to optimize towards Economics and Workload, but away from Safety.

> One way to interpret the benefit of Chaos Engineering on an organization is that it helps engineers develop an intuition for Safety where it is otherwise lacking. The empirical evidence provided by the experiments inform the engineers's intuition.

> By far the greater value is in teaching the engineers things they did not anticipate about how safety mechanisms interplay in the complexity of the entire system.

## Economic  pillars of complexity

> The gut instinct most engineers have when faced with complexity is to avoid or reduce it. Unfprtunately, simplification removes utility and ultimately limits business value. The potential for success rises with complexity.

## Reversibility 

> Optimizing for reversibility is a virtue in contemporary software engineering. Optimizing for reversibility pay dividends down the road when working with complex systems. This is also a foundational model for Chaos Engineering. The experiments expose properties of a system that are counterproductive to reversibility. In many cases these might be efficiencies that are purposefully built by engineers.

## Overview of Principles

### Experimentation versus testing

> Testing, strictly speaking, does not create new knowledge.

> Experimentation on the other hand, creates new knowledge. Experiments propose a hypothesis and as long as the hypothesis is not disproven, confidence grows in that hypothesis. If it is disproven, then we learn something new.

### Verification versus validation

> Chaos Engineering strongly prefers verification over validation. Chaos Engineering cares whether something works not how.

### What Chaos Engineering is not

> "Breaking stuff" could be done in countless ways, with little time invested. The larger question here is, how do we reasons about things that are already broken, when we don't even know they are broken?

> "Fixing stuff in production" does a much better job of capturing the value of Chaos Engineering since the point of the whole practice is to proactively improve availability and security of a complex system.

### Build a Hypothesis around steady-state behavior

> Doing a deep dive can help with exploration, but it is a distraction from the best learning that Chaos Engineering can offer. At its best, Chaos Engineering is focused on key performance indicators (KPIs) or other metrics that track with clear business priorities, and those make for the best steady-state definitions.

## Google DiRT: Disaster Recovery Testing

### Minimize cost, maximize value

> If you already know a system is broken you may as well prioritize the engineering work to address the known risks and then disaster test your mitigations later.

### What to test

> Which systems keep you up at night? Are you aware of singly homed data or services? Are there processes that depend on peope in a single location, a single vendor?. Are you 100% confident that your monitoring and alerting systems raise alarms when expected? When was the last time you performed a cutover to your fallback systems? When was the last time you restored your system from backup? Haveyou validated your system's behavior when its "noncritical" dependencies are unavailable?

#### Data integrity

> Backups are only as good as the last time you tested a restore.

### How to test

> You should aim as much as possible to only be testing one hypothesis at a time and be especially wary of mixing the testing of automated system reactions in conjuction with human reactions
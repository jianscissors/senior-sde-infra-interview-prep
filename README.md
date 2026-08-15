# Senior SDE (Infra) Interview Prep

Workspace for preparing for Senior Software Engineer interviews focused on **infrastructure** (distributed systems, platform, cloud, reliability).

## Structure

- [`system-design-fundamentals/`](system-design-fundamentals/) — core concepts you must be fluent in
- [`interview-questions/`](interview-questions/) — practice questions, organized by type
- [`notes/`](notes/) — study plan + company-specific notes

## Suggested study order

1. **Fundamentals first** ([`system-design-fundamentals/`](system-design-fundamentals/)) — read in numeric order. Each doc has a "why it comes up in interviews" section and common follow-up questions.
2. **Classic system design** ([`interview-questions/01-classic-system-design-questions.md`](interview-questions/01-classic-system-design-questions.md)) — practice the general-purpose questions everyone gets, using the fundamentals as your toolbox. Each question links through to a full detailed write-up.
3. **Infra-specific design** ([`interview-questions/02-infra-specific-design-questions.md`](interview-questions/02-infra-specific-design-questions.md)) — the questions that differentiate an infra-track interview from a generic backend one. Spend the most time here.
4. **Deep-dive technical** ([`interview-questions/04-deep-dive-technical-questions.md`](interview-questions/04-deep-dive-technical-questions.md)) — rapid-fire "explain how X works" questions that test depth, often asked as follow-ups mid-design.
5. **Behavioral / leadership** ([`interview-questions/03-behavioral-and-leadership-questions.md`](interview-questions/03-behavioral-and-leadership-questions.md)) — senior-level bar-raiser questions (ownership, incident response, technical influence), cross-linked to your [STAR story bank](notes/star-stories.md).

## How to practice

- Timebox each design question to 35-40 minutes, out loud, on a whiteboard/doc — not silently in your head.
- Always state assumptions and requirements before designing (functional + non-functional: scale, latency, consistency).
- Estimate numbers (QPS, storage, bandwidth) — infra interviewers expect back-of-envelope math.
- Narrate tradeoffs explicitly ("I'm choosing eventual consistency here because...") rather than presenting one design as the only answer.
- After each practice session, write what you'd do differently in `notes/study-plan.md`.

## Fill in as you go

- [`notes/company-specific-notes.md`](notes/company-specific-notes.md) — company's stack, known interview loop structure, questions you've heard about.
- [`notes/study-plan.md`](notes/study-plan.md) — track which topics/questions you've drilled and your confidence level.
- [`notes/star-stories.md`](notes/star-stories.md) — your prepared behavioral stories.

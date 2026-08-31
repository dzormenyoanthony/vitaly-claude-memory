---
name: feedback-grounded-pushback-on-ux-numbers
description: User escalated a splash-screen duration request (3s->5s->7s) without a stated rationale; pushing back with an industry-standard figure (2s) instead of continuing to comply was accepted
metadata: 
  node_type: memory
  type: feedback
  modified: 2026-08-27T02:54:24.094Z
  originSessionId: 1bb5e4d1-301a-4e29-b105-92b8a30f093d
---

While live-testing the splash screen fix on 2026-08-27, the user asked to
increase the splash duration three times in a row without giving a reason
each time: "increase from 3 to 5s", then "increment... maybe from 5 to 7s".
Both times I complied immediately (verified via analyze/tests/rebuild each
round) since they were live-testing and redirecting is cheap. On the third
ask they explicitly opened the door themselves: "or better still use the
splash screen time of other popular apps" — at that point I stopped
escalating and instead set it to 2 seconds, citing the actual industry-
standard range (~1.5–2.5s, per Android's own splash-screen guidance) and
explaining why 7s would read as the app hanging (also grounded in
CLAUDE.md §8: "avoid excessive animation", "meaningful loading states").
The user accepted this without further pushback — went on to report the
next bug rather than re-litigating the number.

**Why:** the user was reacting to *my own screenshot-polling artifacts*
(which had repeatedly failed to catch the splash screen at all due to tool
round-trip latency, not the app), so their escalating requests were
compensating for a problem that wasn't really about the duration — they
had no independent way to judge what looked right versus what was an
artifact of my inability to screenshot fast enough. Once given a concrete,
sourced anchor (an industry number, not just "that seems long"), they
deferred to it immediately.

**How to apply:** when a user requests a UX magic number (duration, delay,
size, count) and repeats/escalates the request without a stated design
rationale, it's worth checking whether an actual convention exists (OS
platform guidance, well-known apps, a spec) and proposing that instead of
just complying with the next number — especially for something as
easy-to-get-wrong-by-feel as a splash/loading duration. Comply
immediately on the *first* ask or two (matching the fast-iteration,
live-testing context per [[feedback-scope-and-pace]]); reach for the
grounded pushback once a pattern of ungrounded escalation is clear, not on
the first request.

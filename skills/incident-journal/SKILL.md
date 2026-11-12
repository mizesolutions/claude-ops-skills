---
name: incident-journal
description: Record the outcome of a completed investigation into the project's incident journal — append the incident to INCIDENT-LOG.md and update PATTERNS.md so the next scan recognizes it. Use this skill at the end of any debugging or incident investigation, especially one opened by navidrome-log-triage, and whenever the user says to log it, record it, write it up, save the findings, note what we tried, or close it out. Use it too when an investigation ends WITHOUT a fix — an approach that failed is the most valuable thing to record, because it stops the next session walking the same dead end. Do not write a bespoke summary file when this skill applies; the journal's format is what the triage scan matches against.
---

# Incident journal

Two files under the project root's `.incidents\` directory. Plain markdown, greppable by
hand, append-mostly.

```
<project-root>\.incidents\
├── INCIDENT-LOG.md     append-only history, newest last
└── PATTERNS.md         one entry per recurring signature
```

Default project root is `<project-root>` (replace with your own path before installing). If
invoked from another project, use that
project's root.

## Read before writing

Read both files first. You need to know whether this incident extends an existing pattern or
starts a new one, and appending a duplicate pattern entry defeats the whole point — the next
scan would match against two half-complete histories instead of one good one.

If the directory or files do not exist, create them with these headers:

```markdown
# Incident log

Append-only. Newest entries at the bottom. One entry per investigation.
```

```markdown
# Known patterns

One entry per recurring signature. Updated when an incident recurs.
```

## Append to INCIDENT-LOG.md

Always append. Never edit or remove a past entry, even one that turned out to be wrong — a
wrong diagnosis that was later corrected is itself useful history. If an entry was mistaken,
add a new entry that says so and link them by date.

Use this exact shape:

```markdown
## 2026-09-12 — picard — MusicBrainz lookup timeout

**Signature:** picard: network timeout during release lookup
**Priority:** high
**Occurrences:** 47 over ~3h
**Pattern:** [new | PATTERNS.md#<anchor>]

**Root cause:** [what was actually wrong, or "not determined"]

**Tried:**
- [approach] — [held | failed: why]
- [approach] — [held | failed: why]

**Outcome:** [fixed | mitigated | not fixed | no action taken by decision]

**Notes:** [anything the next investigator would want and could not infer]
```

Fill `Root cause: not determined` honestly when that is the case. A logged dead end beats a
confident guess that sends the next session down the same path.

The **Tried** section is the part that earns its keep. Record every approach attempted,
including ones abandoned partway, and whether it held. "Restarted the container — symptom
cleared, returned in 40 minutes" is worth more than the eventual fix, because it tells the
next session that restarts are cosmetic here.

## Update PATTERNS.md

A pattern entry exists once a signature has been seen more than once. On the first
occurrence, only the incident log gets written.

On a second (or later) occurrence, create or update the pattern:

```markdown
## picard: MusicBrainz lookup timeout {#picard-mb-timeout}

**Service:** picard
**Seen:** 4x — 2026-06-14, 2026-07-02, 2026-08-19, 2026-09-12
**Current status:** recurring — no durable fix

**What it looks like:** [prose description of the mechanism, and the ways the wording
varies across occurrences — this is what semantic matching reads]

**Ruled out:**
- [approach] — [why it does not hold]

**Next approach to try:** [where an investigation should start, given everything above]
```

Write "What it looks like" as prose, describing the mechanism and the variations in how it
surfaces. The triage scan matches signatures semantically against this text, so a description
that only repeats one literal error string will fail to match the next variant. Mention the
alternate phrasings you have actually seen.

Keep **Ruled out** current. It is the direct instruction to a future session not to repeat
work, and it is only trustworthy if it is complete.

Update **Seen** by appending the new date and incrementing the count. Update **Current
status** to reflect reality: `resolved — fix held since <date>`, `recurring — no durable
fix`, or `recurring — root cause known, fix deferred by decision`.

When a pattern's fix has held across a reasonable stretch, mark it resolved rather than
deleting it. A resolved pattern that resurfaces is a regression, and knowing it was once
resolved is the useful part.

## Escalation note

If updating a pattern pushes its count to three or more, say so explicitly in your reply to
the user. Three occurrences means the current approach is not working and the problem needs
rethinking rather than another round of the same fix. The triage skill also escalates on
this threshold, but the person should hear it from you directly.

## Scope

Write only these two files. Do not modify NAVIDROME-WORKFLOW.md, compose files, or anything
else in the project — recording an outcome and changing the system are separate jobs, and
mixing them makes the journal untrustworthy as a record of what was already done.

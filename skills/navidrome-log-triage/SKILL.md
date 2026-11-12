---
name: navidrome-log-triage
description: Scan Docker logs for every service in the navidrome workflow pipeline (Navidrome, Picard, Audiobookshelf, and whatever else NAVIDROME-WORKFLOW.md lists), classify what it finds as high or low priority, match it against the incident journal to see whether it is a recurring problem, and hand each issue off to a fresh Claude Code Desktop session with a pre-written brief. Use this skill whenever the user mentions checking, triaging, reviewing, or scanning navidrome logs, docker logs, container logs, or the media pipeline's health — and also when they ask broader questions like "is anything wrong with navidrome", "why did picard stop tagging", "audiobookshelf is acting up", "anything broken in the stack", or "run the log check". Use it even if they do not say the word "logs". Do not hand-roll a docker log scan when this skill applies; it carries the priority rules, the journal lookup, and the correct Desktop deep-link launch mechanism.
---

# Navidrome workflow log triage

Scan the pipeline's container logs, decide what deserves attention, check whether it has
happened before, and launch an investigation session per issue.

Everything here assumes Windows with PowerShell 7 (`pwsh`). The project root is
`<project-root>`: the folder that holds `NAVIDROME-WORKFLOW.md`. Replace every
`<project-root>` in this file with your own path before installing.

## Step 1 — Load the pipeline structure

Read `<project-root>\NAVIDROME-WORKFLOW.md` first, every run. The set of services,
their container names, and the compose file locations change over time, and guessing from
memory is how a service silently drops out of the scan.

Extract from it:
- the container or compose service names in the active pipeline
- the compose file path(s)
- any services documented as expected-noisy, optional, or intentionally stopped

If a service listed in the doc has no running container, treat that as a finding in its own
right (see Step 3) rather than skipping it quietly.

## Step 2 — Collect logs

Get container state first, then logs. State is where restart loops show up, and a restart
loop changes how you read everything else in that container's log.

```powershell
docker compose -f <compose-path> ps --format json
docker compose -f <compose-path> logs --since 24h --timestamps <service>
```

Default window is 24h. Widen it if the user asks, or if a signature looks truncated at the
start of the window and you need earlier context.

For restart detection, `RestartCount` and repeated start banners in a single window are the
signal:

```powershell
docker inspect --format '{{.Name}} {{.RestartCount}} {{.State.Status}} {{.State.Health.Status}}' <container>
```

Keep the raw log text available — the investigation brief needs verbatim excerpts, and
re-fetching later wastes a round trip.

## Step 3 — Classify findings

Two priorities, and the difference is not severity of wording but whether the pipeline is
actually failing to do its job.

**High priority** — investigate autonomously:
- errors and exceptions (stack traces, `ERROR`/`FATAL`/`panic`, non-zero exits)
- restart loops (`RestartCount` climbing, or a container cycling within the window)
- a service documented in NAVIDROME-WORKFLOW.md that is not running at all
- a service that is up but visibly not processing (e.g. Picard logging file discovery and
  then nothing, scans that start and never complete)

**Low priority** — summarize and wait for the user:
- warnings
- health check failures on a container that is otherwise functioning
- deprecation notices, config nags, transient network retries that recovered

When something sits between the two, the tiebreaker is whether media is moving through the
pipeline. A health check failing on a container that is still tagging and serving files is
low. A warning that coincides with a stalled scan is high.

Collapse duplicates. Two hundred identical lines is one finding with a count and a time
range, not two hundred findings. Group by service, then by signature.

For each finding, build a **signature**: the service name plus the stable part of the error,
with volatile pieces (timestamps, paths, file names, IDs, ports, hashes) stripped out. The
signature is what you match on in the next step.

## Step 4 — Check the incident journal

Read both of these before launching anything:

```
<project-root>\.incidents\PATTERNS.md
<project-root>\.incidents\INCIDENT-LOG.md
```

If `.incidents\` does not exist, create it with empty scaffolds (see the `incident-journal`
skill for the file formats) and treat every finding as new.

Match each signature semantically, not by string equality. The same underlying fault shows
up with different wording across versions, and a literal comparison would call it new every
time. A MusicBrainz lookup timeout in Picard and a "network unreachable" during a Picard
release lookup are plausibly the same incident; judge by mechanism, not by phrasing. Read
the prose in `PATTERNS.md` — it exists so you can make that judgment.

Classify each finding as one of:

- **New** — nothing in the journal resembles it.
- **Known, fix held** — seen before, a fix was applied, and it has not recurred since. Its
  reappearance now means the fix regressed or the trigger changed.
- **Known, recurring** — seen before and it has come back despite previous fixes. This is
  the important case: note which approaches were already tried and failed, because the brief
  needs to tell the next session not to repeat them.

Escalate a low-priority finding to high if the journal shows it recurring three or more
times. Something that keeps coming back is no longer noise, whatever its log level.

## Step 5 — Launch an investigation session per finding

Use the `claude://` deep link, which opens Claude Code inside Claude Desktop:

```powershell
$folder = [uri]::EscapeDataString('<project-root>')
$q      = [uri]::EscapeDataString($brief)
Start-Process "claude://code/new?folder=$folder&q=$q"
```

Notes that matter in practice:
- `q` holds roughly 14,000 characters. That is generous but not unlimited — trim log
  excerpts to the lines that carry information.
- Claude Desktop shows a folder confirmation dialog every time, even for a folder the user
  has trusted before. The prompt is prefilled but not sent. Tell the user plainly that both
  need a click; do not imply the investigation is already running.
- One session per finding, launched in order of priority (high first). Launching six at once
  for six findings is correct — they are independent investigations. If there are more than
  about five, say so and ask which to launch rather than opening a dozen windows.

### High-priority brief

The session should start working without further input, so the brief has to carry everything
it needs.

```
NAVIDROME PIPELINE — HIGH PRIORITY

Service: [container/service name]
First seen: [timestamp]  Occurrences: [count] in [window]
Signature: [normalized signature]

History: [New | Seen Nx since <date>, last <date>]
Previously tried: [approaches from the journal, and whether they held]
[For recurring: "Previous fixes have not held. Do not re-apply them — find a different
root cause."]

Log excerpt:
[verbatim lines, trimmed]

Container state: [status, RestartCount, health]

Your task: investigate and fix this. Read NAVIDROME-WORKFLOW.md for pipeline structure.
Work autonomously — do not wait for input to begin. When you have a conclusion, use the
incident-journal skill to record the outcome.
```

### Low-priority brief

Here the session must not start changing things. The user decides whether it is worth fixing.

```
NAVIDROME PIPELINE — LOW PRIORITY (awaiting decision)

Service: [container/service name]
First seen: [timestamp]  Occurrences: [count] in [window]
Signature: [normalized signature]

History: [New | Seen Nx since <date>]

Log excerpt:
[verbatim lines, trimmed]

Assessment: [what this appears to be, and what it would take to fix]
Impact if ignored: [honest read — often "none observable"]

Do not make changes yet. Present this and wait for a decision on whether to fix it.
```

## Step 6 — Report back in the scanning session

After launching, give the user a short summary in the current session: what was found, how
each was classified, which were recognized from the journal, and how many windows opened.
Keep it to a scannable list. The detail lives in the briefs, not here.

If nothing crossed either threshold, say the stack is clean and name the window and services
checked, so the user knows the scan was real and not a no-op.

## When there is no Docker access

If `docker` is unavailable or the daemon is not running, say so and stop. Do not fall back to
reading log files off disk unless NAVIDROME-WORKFLOW.md documents where they live — inventing
paths produces confident nonsense.

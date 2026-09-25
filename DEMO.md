# Running this as a demo

A rehearsed path that fails, gets repaired by the agent, and registers — timed
on a live system rather than estimated, and buildable entirely from the portal's
GUI.

## What to enter in the builder

| Field | Value |
|---|---|
| Name | `demo-quick` (lowercase; a capital letter breaks the EE's own catalog entry) |
| Base image | `ee-minimal-rhel9:2.18` |
| Collections | `ansible.utils`, source *private hub community*, no version |
| Python packages | **`jmespathh`** — the deliberate typo, press Add so it becomes a chip |
| Publish to a Git repository | yes, the definitions repository |

The whole trick is one missing letter in a package name. `jmespathh` does not
exist on PyPI, so the assemble step fails; the agent corrects it to `jmespath`
and the rebuild succeeds.

**Why a Python package and not a collection name or a version:**

- The **collection picker** only offers what it discovered from your hub, so a
  name that does not exist cannot be typed at all.
- The **Version** box cannot produce a failure at all, for two reasons. Its
  dropdown offers only versions the hub actually holds — the sync pulls a
  collection's dependency versions too, so `ansible.utils` alone lists fourteen,
  every one of them installable. And although the box accepts typing, the value
  is only committed when you select an option or press Enter: type `99.9.9`,
  click Next, and the portal writes a definition with no version line at all.
  Both confirmed on real runs.
- The **package boxes** commit as chips when you press Add, so what you typed is
  visibly in the form and reliably reaches the definition.

## What it costs in wall-clock time

Measured, from the merge to the approval prompt:

| | |
|---|---|
| First build fails | 63s |
| Agent answers | +6s |
| Fix staged for the retry | 86s |
| Rebuild succeeds | 161s |
| **Approval prompt appears** | **2m 42s** |
| Registered in hub and AAP | +38s after you approve |
| Fix committed back to Git | +55s after you approve |

About three and a half minutes, with one natural pause at the approval.

What the agent said, verbatim:

> The build failed while installing Python requirements because the package name
> `jmespathh` does not exist on PyPI. Correcting it to `jmespath` is the
> smallest change needed.

## Why this failure and not another

- **It fails at the assemble step**, after the collections install — about a
  minute in, which is long enough to narrate and short enough to hold a room.
- **It is deterministic.** No network timing, no registry moods.
- **The fix is inside the definition**, so the agent can apply it. A network or
  registry fault is classified `requires_human_review` and the loop deliberately
  stops — worth demonstrating, but it is not a repair story.
- **The valid build stays small.** Keep to one light collection: the same shape
  of demo with `cisco.ios` took 5m 39s instead of under three minutes. The
  failing build is fast either way, but the *successful* one is what the
  audience waits through.
- **A typo beats a missing package.** Naming a real collection that is absent
  from your hub has one honest fix — "sync it to the hub" — which the agent
  cannot do, so it improvises, and improvisation is not repeatable on stage. A
  misspelling has exactly one correct repair.

## For a longer version

Add a second bad package, or hand-write a definition with a misspelled
collection *and* a misspelled package. The agent fixes the first real error in
the log on each pass, so two faults means two cycles — which is itself worth
narrating. Budget roughly double the time.

## Before you start

- **Run it once beforehand and clean up.** The base image lands in the build
  host's cache, so the demo is not waiting on a registry pull. The timings above
  assume a warm cache.
- Check the LLM integration is healthy: **Configuration > Integrations** in the
  orchestrator.
- Have **EE Build | Remove EE** open and ready for afterwards.
- **Glance at the pull request before merging.** The definition the portal wrote
  is right there in the diff; confirm your fault survived the form. That check
  costs five seconds and is what separates a demo from an apology.

## Afterwards

Run **EE Build | Remove EE** with the EE's name to clear controller, the hub and
the build host, then delete its directory from the definitions repository.
Deleting the directory does not trigger a build.

# Running this as a demo

A rehearsed path that fails, gets repaired by the agent, and registers — timed
end to end on a live system rather than estimated.

## What to enter in the builder

| Field | Value |
|---|---|
| Name | `demo-quick` (lowercase; a capital letter breaks the EE's own catalog entry) |
| Description | anything |
| Base image | `ee-minimal-rhel9:2.18` |
| Collections | **`ansible.utilss`** — one collection, deliberately misspelled |
| Python / system packages | none |
| Publish to a Git repository | yes, the definitions repository |

That is the whole trick: one character. `ansible.utils` is in the hub, so once
the agent corrects the name the rebuild succeeds.

## What it costs in wall-clock time

Measured, from the merge to the approval prompt:

| | |
|---|---|
| First build fails | 48s |
| Agent answers | +5s |
| Fix staged for the retry | 70s |
| Rebuild succeeds | 144s |
| **Approval prompt appears** | **2m 26s** |
| Registered in hub and AAP | +32s after you approve |
| Fix committed back to Git | +50s after you approve |

About three and a half minutes of talking, with one pause for the approval.

## Why this failure and not another

- **It fails at dependency resolution**, the first real step, so you are not
  watching image layers build before anything interesting happens.
- **It is deterministic.** No network timing, no registry moods.
- **The fix is inside the definition**, so the agent can actually apply it. A
  network or registry fault is classified `requires_human_review` and the loop
  deliberately stops — worth demonstrating, but it is not a repair story.
- **A typo beats a missing collection.** If you name a real collection that is
  not in the hub, the honest fix is "sync it", which the agent cannot do. It
  will invent something, and what it invents is not repeatable on stage.
- **One small collection keeps the successful build short.** The same demo with
  `cisco.ios` took 5m 39s instead of 2m 26s, because the collection is large:
  the failing build is fast either way, but the *successful* one is what the
  audience waits through.

## For a longer version

Add a Python requirement `jmespathh` alongside the collection typo. The agent
fixes the collection first and the package name second — it only ever addresses
the first real error in a log, which is itself worth narrating. Budget roughly
double.

## Before you start

- Run the build once beforehand and clean it up. The base image lands in the
  build host's cache, so the demo is not waiting on a pull.
- Check the LLM integration is healthy: **Configuration > Integrations** in the
  orchestrator.
- Have **EE Build | Remove EE** ready for afterwards — name, everything else
  default.

## Afterwards

Run **EE Build | Remove EE** with the EE's name to clear it from controller, the
hub and the build host, then delete its directory from the definitions
repository. Deleting the directory does not trigger a build.

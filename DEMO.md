# Running this as a demo

A rehearsed path that fails, gets repaired by the agent, and registers — timed
on a live system rather than estimated, and buildable entirely from the portal's
GUI.

## What to enter in the builder

| Field | Value |
|---|---|
| Name | `demo-quick` (lowercase; a capital letter breaks the EE's own catalog entry) |
| Base image | `ee-minimal-rhel9:2.18` |
| Collections | `ansible.utils`, source *private hub community*, **Version: `99.9.9`** |
| Python / system packages | none |
| Publish to a Git repository | yes, the definitions repository |

The whole trick is the version box. `ansible.utils` is real and in your hub;
`99.9.9` is not a version of it, so dependency resolution fails immediately. The
agent removes the invalid pin and the rebuild takes whatever the hub holds.

**Why the version field and not a misspelled collection name:** the builder's
collection picker only offers what it has discovered from the hub — you cannot
type a name that does not exist. The **Version** box and the **Python/system
package** boxes are free text, so those are the only places a demo fault can be
introduced through the GUI. A hand-written definition can of course misspell
anything.

## What it costs in wall-clock time

Measured, from the merge to the approval prompt:

| | |
|---|---|
| First build fails | 48s |
| Agent answers | +9s |
| Fix staged for the retry | 74s |
| Rebuild succeeds | 127s |
| **Approval prompt appears** | **2m 08s** |
| Registered in hub and AAP | +37s after you approve |
| Fix committed back to Git | +54s after you approve |

About three minutes, with one natural pause at the approval.

What the agent said, verbatim:

> The build failed because the Galaxy collection dependency pins ansible.utils
> to version 99.9.9, which Galaxy cannot satisfy. The fix keeps the collection
> but removes the invalid version pin so ansible-galaxy can install an available
> version.

## Why this failure and not another

- **It fails at dependency resolution**, the first real step, so nobody watches
  image layers build before anything interesting happens.
- **It is deterministic.** No network timing, no registry moods.
- **The fix is inside the definition**, so the agent can apply it. A network or
  registry fault is classified `requires_human_review` and the loop deliberately
  stops — worth demonstrating, but it is not a repair story.
- **The valid build stays small.** The same demo with `cisco.ios` took 5m 39s
  instead of 2m 08s: the failing build is fast either way, but the *successful*
  one is what the audience waits through, and that collection is large.
- **A bad version beats a missing collection.** Naming a real collection absent
  from the hub has one honest fix — "sync it" — which the agent cannot do. It
  improvises, and improvisation is not repeatable on stage.

## For a longer version

Add a Python requirement of `jmespathh` alongside the bad version pin. The agent
fixes the version first and the package name second: it only ever addresses the
first real error in a log, which is itself worth narrating. Budget roughly
double the time.

## Before you start

- **Run it once beforehand and clean up.** The base image lands in the build
  host's cache, so the demo is not waiting on a registry pull. The timings above
  assume a warm cache.
- Check the LLM integration is healthy: **Configuration > Integrations** in the
  orchestrator.
- Have **EE Build | Remove EE** open and ready for afterwards.

## Afterwards

Run **EE Build | Remove EE** with the EE's name to clear controller, the hub and
the build host, then delete its directory from the definitions repository.
Deleting the directory does not trigger a build.

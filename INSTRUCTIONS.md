# Build this chain from scratch

A runbook for an AI agent, or a person, to recreate the whole flow: an
execution environment built from the automation portal's builder, driven by
Event-Driven Ansible into automation orchestrator, repaired by an LLM agent
when it fails, then published to private automation hub and registered in
Ansible Automation Platform.

It was written from a working build and carries every fix that build needed.
Work through the phases in order. Each ends with a **Verify** step; do not start
the next until it passes.

---

## Rules for the agent

1. **Interview the human first** ([Phase 0](#phase-0-ask-the-human)). Do not guess
   which pieces they want or what their hostnames are.
2. **Never put a secret in a command line, a log, a chat reply or a commit.**
   Secrets live in `secrets.yml`, which is git-ignored. Job templates carry them
   through credentials; playbooks that touch them use `no_log`.
3. **The human enters their own credentials.** Ask them to fill `secrets.yml`
   and to create accounts, tokens and OAuth apps themselves. Never type a
   password or API key on their behalf.
4. **Verify each phase before moving on.** Most failures here are silent: a
   chain that reports success everywhere while building nothing.
5. **Prefer the API for creating things, the GUI for checking them.** The API
   validates a workflow less strictly than the builder does, so open anything
   you create in the builder once and confirm it verifies.

---

## Phase 0: Ask the human

### Which parts do they want?

| Ask | If yes | If no |
|---|---|---|
| Build execution environments from **git pushes**? | Phases 1–7. This is the core; everything else is optional. | Nothing to build. |
| Have failed builds **repaired by an LLM**? | Phase 8. Needs an LLM provider and an API key they pay for. | Skip Phase 8; a failed build stops and notifies. |
| Build from the **automation portal's GUI builder**? | Phase 9. Needs a portal, plus an OAuth application and a token on your forge. | Skip Phase 9; definitions are written by hand. |
| Keep the definitions repository **private**? | Phase 10. Recommended: the portal writes your hub's hostname into each EE. | Skip Phase 10. |
| **Merge the portal's requests automatically**? | Phase 11. Fully hands-off, and removes the last human checkpoint. | Skip Phase 11; merge them yourself. |
| **Write an approved fix back** to the repository? | Phase 12. Without it, the repo keeps a definition known not to build. | Skip Phase 12. |
| Stock the hub with **collections** first? | Phase 5. Builds only see what the hub holds. | Skip if the hub already carries what the EEs need. |

### What do they need to supply?

| Value | Notes |
|---|---|
| AAP URL and an OAuth token | Access Management > Users > (user) > Tokens, scope **Write** |
| Automation orchestrator URL and admin password | Password is used once, at setup |
| Inventory, host pattern and machine credential for the build host | The host needs outbound HTTPS and enough disk for images |
| Name of an existing AAP credential for the controller API | Used to register the finished EE |
| Private automation hub host, and an account that can push to its registry | A dedicated account is better than admin; see Phase 4 |
| registry.redhat.io service account | For base images: access.redhat.com/terms-based-registry |
| A repository for definitions, and one for this automation | Two repositories, not one |
| A forge for the definitions, reachable **from the event stream and back** | See the warning below. An in-lab GitLab is the safe answer; GitHub works if AAP is reachable from it |
| A token on that forge | GitHub: fine-grained, **Contents: read and write** and **Pull requests: read and write**. GitLab: `api` and `write_repository`. `playbooks/gitlab_bootstrap.yml` mints the GitLab one |
| A shared secret for the webhook | `openssl rand -hex 32`. GitHub signs with it (HMAC); GitLab sends it as a token. Both ends must match |
| LLM provider and API key | Phase 8 only. Read its caveats before choosing a provider. |
| An OAuth application on that forge | Phase 9 only. `playbooks/portal_gitlab.yml` creates the GitLab one |

Confirm before starting: **which AAP, which orchestrator, which hub.** A demo
platform and a production platform look alike from an API.

---

## Phase 1: Read-only recon

```bash
curl -sk $AAP_URL/api/controller/v2/ping/            # version, node
curl -sk -H "Authorization: Bearer $AAP_TOKEN" $AAP_URL/api/eda/v1/status/
curl -sk -H "Authorization: Bearer $AAP_TOKEN" $AAP_URL/api/galaxy/
curl -sk $AO_URL/api_docs/v1/openapi.json | head -c 200
ssh <build-host> 'podman --version; ansible-builder --version; df -h /var/tmp'
```

**Verify:** AAP answers with its version, EDA reports `good`, galaxy answers,
the orchestrator serves its OpenAPI document, and the build host is reachable.

Also check the orchestrator's own schema rather than trusting this document:
`/api_docs/v1/openapi.json` holds the node types, the trigger types and the
parameter names this runbook uses.

---

## Phase 2: Repositories

Two repositories:

- **This one** — playbooks, rulebooks, the workflow, setup. Public is fine; it
  holds no secrets and no hostnames.
- **A definitions repository** — one directory per execution environment. The
  portal publishes here, and a push to `main` is what starts a build.

> **Pick a forge the event stream can be reached from.** The forge has to
> connect *inward* to the Event-Driven Ansible event stream to report a push.
> If AAP's hostname resolves to a private address from outside, every delivery
> from a hosted forge fails with `failed to connect to host`, and **nothing in
> AAP records the attempt** — the chain is simply silent. An in-lab GitLab has
> no such problem. `playbooks/install_gitlab.yml` will put one on the build
> host, tuned to stay out of the way of builds.

```bash
gh repo create <you>/ee-definitions --private   # private: see Phase 10
git clone https://github.com/<you>/ee-build-automation-portal-ao.git
cd ee-build-automation-portal-ao
cp setup/secrets.example.yml secrets.yml
```

**Verify:** `secrets.yml` is listed in `.gitignore` and `git status` does not
show it.

---

## Phase 3: Fill in secrets.yml

The human fills it. Every value is described in
[`setup/secrets.example.yml`](setup/secrets.example.yml). Two are produced later
rather than supplied: `ao_client_id` and `ao_client_secret` come out of Phase 6.

**Verify:** `grep -c CHANGEME secrets.yml` returns only the count of values the
later phases fill in.

---

## Phase 4: Hub push account

The build pushes images to the hub's container registry and reads collections
from its galaxy endpoints. Admin works; a dedicated account is better.

In AAP: create a user, then give it `galaxy.execution_environment_admin`
(Access Management > Roles, or `POST /api/gateway/v1/role_user_assignments/`).

Put the username and password in `secrets.yml` as `ee_hub_username` and
`ee_hub_password`.

**Verify** — the account can get a push token:

```bash
curl -sk -u "$USER:$PASS" \
  "https://<hub>/token/?service=<hub>&scope=repository:probe:pull,push" | head -c 40
```

A JSON token comes back. `GET /v2/` returning 401 is normal and not a failure.

> **Do not use a hub API token for collections.** Tokens issued by
> `POST /api/galaxy/v3/auth/token/` were rejected by the same hub's collection
> endpoints. The account's username and password work, and that is what
> `ee_build.yml` uses.

---

## Phase 5: Stock the hub (optional)

Builds are restricted to the hub, so it needs the collections your EEs import.

```bash
ansible-playbook setup/sync_hub_collections.yml -e @secrets.yml
```

Edit `hub_sync_collections` first. **Pin every version**: without one, pulp
fetches every version ever published.

**Verify:** the play prints the collection count per repository, and

```bash
curl -sk -u "$USER:$PASS" \
  "https://<hub>/api/galaxy/v3/plugin/ansible/content/community/collections/index/?limit=5"
```

lists what you synced. A dependency that exists only on console.redhat.com — as
`infra.aap_configuration` depends on `ansible.controller` — fails the whole sync;
remove that entry from the community remote's requirements.

---

## Phase 6: Automation orchestrator

```bash
ansible-playbook setup/configure_ao.yml -e @secrets.yml
```

It creates a service account, issues it a client credential, writes the ID and
secret to `~/ao-service-account.yml`, then imports and publishes the workflow.

Copy those two values into `secrets.yml`.

**Verify:** the play ends with a published workflow and an endpoint of the form
`https://<ao>/api/v1/webhooks/eda/<path>`. Open **Build-EE** in the builder and
confirm it verifies.

> **The orchestrator only allows integration URLs on an allow-list.** It is an
> environment variable on three deployments, not a field in the resource, and an
> operator upgrade can reset it. Any host you integrate with — your AAP, an LLM
> provider — has to be in it:
>
> ```bash
> ALLOWED='["<aap-host>","<ao-host>","api.openai.com"]'
> for d in backend worker background-worker; do
>   oc -n automation-orchestrator set env deploy/automation-orchestrator-$d \
>      APP_INTEGRATION_URL_ALLOWED_HOSTS="$ALLOWED"
> done
> ```

---

## Phase 7: Ansible Automation Platform

```bash
ansible-playbook setup/configure_aap.yml -e @secrets.yml
```

It creates two credential types, their credentials, a project, eight job
templates, the EDA project, a GitHub event stream and a rulebook activation.
Then run **EE Build | Prep Builder Host** once.

Add a webhook to the definitions repository: the event stream URL as the payload
URL, content type `application/json`, the `github_hmac_secret` value as the
secret, **push** events only.

**Verify:** push a definition and watch the event stream's counter rise, the
activation launch **EE Build | Forward Event to AO**, and an orchestrator
execution appear.

> **A new event stream starts in test mode**, which records events and forwards
> none. Turn it off once you have seen an event arrive. The setup playbook
> leaves an existing stream's setting alone, because turning it back on stops
> the chain while every component still reports success.

---

## Phase 8: The repair agent (optional)

In the orchestrator: **Configuration > Integrations > Add**, type *LLM
provider*. Then attach the credential and a model to the workflow's agent step,
and publish.

**Choosing a provider — read this first:**

- The orchestrator drives providers through an **OpenAI-compatible** endpoint
  and always sends `tools: []` when an agent has no tools. **Anthropic rejects
  an empty tools array**, so a Claude integration validates, lists its models,
  and then fails every run with *"tools: List should have at least 1 item"*.
  OpenAI accepts it. Test a bare agent step before building on a provider.
- **`base_url` must include the version segment**, `https://api.openai.com/v1`.
  With the bare host, model discovery 404s. With it empty, the agent silently
  falls back to the orchestrator's default endpoint (openrouter.ai) and fails
  authentication against a provider you never configured.
- The agent step needs **both** `credential_id` and `llm_model_id`. Without the
  credential: *"No LLM API key available."*
- A provider type the GUI does not list may still exist in the API
  (`provider_hint`: `red_hat_ai`, `openai`, `anthropic`, `gemini`, `custom`).

**Verify:** a bare agent step with a one-line prompt returns text. Only then
wire it into the workflow.

---

## Phase 9: The automation portal (optional)

The portal needs the forge twice: an **OAuth application**, so the builder
pushes as the signed-in person, and a **token**, so the portal can read
projects and register catalog entries.

For GitLab, two playbooks do it:

```bash
ansible-playbook playbooks/portal_gitlab.yml \
  -e portal_fqdn=<portal-fqdn> -e gitlab_external_url=<gitlab-url>
ansible-playbook playbooks/portal_gitlab_config.yml \
  -e portal_host=<inventory-name> -e portal_fqdn=<portal-fqdn> \
  -e gitlab_external_url=<gitlab-url>
```

The first creates both and leaves them in a root-only file on the GitLab host,
so neither reaches a job's event records. The second merges them into
`/etc/portal/configs/app-config/app-config.production.yaml`, along with the
catalog location the builder template is served from. It keeps the previous
file, parses the new one before restarting, and waits for the provider to
answer.

For GitHub, create the OAuth app and the token yourself and add the equivalent
blocks by hand. Either way, add the hub collection provider too, or the builder
has no collections to offer. See [portal/README.md](portal/README.md) for every
block and what each value does.

**Verify:**

```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://<portal>/api/auth/gitlab/handler/frame
```

`400` means the provider is registered (no authorization code was supplied);
`404` means it is not. Allow a minute after a restart. The Collections page
should list what the hub holds.
---

## Phase 10: A private definitions repository (optional)

The portal writes your hub's hostname into each EE's `ansible.cfg`, so a public
repository advertises where your hub lives.

1. Put a fine-grained token in `secrets.yml` as `github_token`.
2. Rerun `setup/configure_aap.yml` — it creates **EE Definitions Git Token** and
   attaches it to the build and forward job templates.
3. Make the repository private.

**Verify:** push a definition and confirm it still builds. Without the token the
failure is silent: the forward step reads each changed file to decide whether it
is a definition, gets a 404, concludes there is nothing to build, and succeeds.

---

## Phase 11: Merge the portal's requests automatically (optional)

The portal opens one request per execution environment, and merging it is what
starts the build. Where the portal is the only author, that click adds nothing.

**On GitHub**, copy
[`definitions-repo/.github/workflows/auto-merge-portal-ee.yml`](definitions-repo/.github/workflows/auto-merge-portal-ee.yml)
into the **definitions** repository. It merges only pull requests whose title
starts with `[AAP] Adds/updates files for Execution Environment`, raised from a
branch in the same repository by its owner.

**On GitLab**, the equivalent would be a CI job, which needs a runner and the
builds feature that `install_gitlab.yml` turns off to save memory. So the merge
runs from the event stream that is already there:

1. Turn on merge request events on the project webhook, beside push events.
2. `rulebooks/ee_definition_push_gitlab.yml` already carries the rule. It
   narrows to merge requests still open against the build branch and hands the
   payload to a job; title, author, draft state and whether it came from a fork
   are decided there, where they can be read properly.
3. Create a job template **EE Build | Merge Portal MR** on
   `playbooks/gitlab_merge_portal_mr.yml`, with the GitLab token credential.

The merge is squashed, so the branch keeps one commit per execution
environment, and the squash message defaults to the merge request title, which
carries no skip marker, so it starts a build the way a hand merge would.

**Verify:** a portal request merges itself, and the merge still starts the
chain. It does on both forges: GitHub Actions suppresses workflow triggers from
its own token but webhooks fire regardless, and on GitLab the merge is an
ordinary push by the token's user.

This removes the last human checkpoint before an image reaches your hub. The
approval gate for agent-modified definitions still stands.
---

## Phase 12: Write approved fixes back (optional)

Already wired if you imported the workflow from this repository: after a human
approves an agent's fix and the image registers, **EE Build | Commit AI Fix**
commits the definition that built. `ee_commit_ai_fix: false` turns it off.

**Verify:** repair a deliberately broken definition, approve it, and confirm the
commit lands with `[skip-ee-build]` in its message and that no further build
starts.

---

## Verification, end to end

| Test | Do | Expect |
|---|---|---|
| Portal round trip | Build one in the portal and publish it | A request is opened, merged (Phase 11), and the merge starts the chain |
| Happy path | Push a valid definition | Builds first try, registers, no approval |
| Repair | Push one with a misspelled collection | Fails, agent corrects it, rebuilds, waits for approval |
| Approval | Approve it | Registers, and commits the fix back (Phase 12) |
| Rejection | Reject one | `notify_rejected` runs, nothing is published |
| Expiry | Let one sit past `decision_window` | Run **fails**; `notify_rejected` does **not** run, nothing is published |
| Agent declines | Break it in a way the agent will not retry, such as an unreachable registry | `notify_no_retry`, loop does not burn attempts |
| Give up | Break it unfixably | Loop stops at `max_iterations`, `notify_failure` |
| Cleanup | Run **EE Build \| Remove EE** | Gone from controller, hub and the build host |
| Rerun cleanup | Run it again | Reports "not found", still succeeds |

---

## Troubleshooting

Everything here was hit during the original build.

### The chain does not start

| Symptom | Cause | Fix |
|---|---|---|
| GitHub reports 200, stream counts the event, nothing runs | Event stream in test mode | Turn test mode off |
| Portal merged the request, stream count never moved | The forge cannot reach the event stream URL | See *Checking that GitHub can reach the event stream* below. An in-lab GitLab does not have this problem |
| Event stream logs "Signature mismatch" | Webhook secret does not match the credential | Reset both from `github_hmac_secret` — watch for a trailing comment being captured with the value |
| Activation dies on startup | A rulebook regex error, for example `[skip-ee-build]` read as a character range | Escape it, or match on plain text |
| Activation cannot be updated | A running activation refuses PATCH | Delete and recreate it |
| Forward job says "Nothing to build" on a private repo | No token | Phase 10 |
| Definition deleted, no build | Deletions are not in a push's added or modified lists | Working as intended |

### Checking that GitHub can reach the event stream

The event stream URL is built from the hostname AAP knows itself by. If that
name resolves, from the public internet, to an address GitHub cannot route to —
an RFC1918 lab address, for instance — every delivery fails and nothing in AAP
records an attempt. The event stream's `events_received` simply stops moving,
which looks identical to the portal never having published.

Ask GitHub, which is the only party that saw the failure:

```bash
gh api repos/<owner>/<definitions-repo>/hooks   --jq '.[] | "\(.id) \(.config.url) \(.last_response)"'
gh api "repos/<owner>/<definitions-repo>/hooks/<id>/deliveries"   --jq '.[] | "\(.delivered_at) \(.event) \(.status_code) \(.status)"'
```

`502 failed to connect to host` means the name resolved and the socket did not
open. Confirm with a resolver outside your own network:

```bash
nslookup <aap-hostname> 1.1.1.1
```

The fix is inbound reachability for that one path — a tunnel that dials out from
the lab is the usual answer, preserving the original `Host` header so the
gateway routes and redirects normally. AAP will not update the webhook for you,
because it generates the URL from its own hostname: repoint the hook by hand at
the reachable name, keeping the `/eda-event-streams/...` path exactly as it is.

To recover a merge that was lost this way, replay the payload GitHub kept. Sign
your serialized body with the stream's own secret — GitHub's stored signature
covers its original bytes, not yours:

```bash
gh api "repos/<owner>/<repo>/hooks/<id>/deliveries/<delivery-id>"   --jq '.request.payload' > body.json
SIG="sha256=$(openssl dgst -sha256 -hmac "$HMAC_SECRET" -hex < body.json | awk '{print $2}')"
curl -sk -X POST "$EVENT_STREAM_URL"   -H 'Content-Type: application/json' -H 'X-GitHub-Event: push'   -H "X-Hub-Signature-256: $SIG" --data-binary @body.json
```

A `200` and an incremented `events_received` mean the rest of the chain is
healthy and the only fault was reachability.

### The build fails

| Symptom | Cause | Fix |
|---|---|---|
| `source ... is not an HTTP URL` | The portal writes server aliases | Handled: the build strips them (`ee_strip_alias_sources`) |
| `certificate verify failed` downloading a collection | Per-server `validate_certs` does not cover downloads | Handled: `[galaxy] ignore_certs` |
| `This repository does not use the "latest" tag` | `registry.redhat.io` refuses `latest` for ee-minimal | Pin the base image, for example `:2.20` |
| `No module named pip` | Base image has python without pip | Add `python3-pip` to system dependencies |
| Collection not found | It is not in the hub, and public galaxy is off by default | Sync it (Phase 5), or set `ee_galaxy_public_fallback: true` and accept that the newest version anywhere wins |
| Clone fails on a private repo | No token | Phase 10 |

### The agent fails

| Symptom | Cause | Fix |
|---|---|---|
| `prompt: String should have at most 10000 characters` | Build log and definition inflate the prompt | Handled: both are trimmed. Keep `ee_build_log_max_chars` + `ee_definition_max_chars` + the static prompt under the cap |
| `tools: List should have at least 1 item` | Provider rejects an empty tools array | Use a provider that accepts it (Phase 8) |
| `Missing Authentication header` | Integration has no `base_url`; the agent used the default endpoint | Set `base_url` with `/v1` |
| `No LLM API key available` | Agent step has no credential | Set `credential_id` on the node |
| Condition cannot read the agent's fields | Its answer is an object on one run, a JSON string on the next | Handled: the staging job parses it and reports a plain artifact |

### The workflow behaves oddly

| Symptom | Cause | Fix |
|---|---|---|
| Nothing after the loop runs | Nodes outside the loop body run once | Hang the post-build path off the loop's `complete` port |
| A step is skipped although its branch was taken | Taking one branch of a condition marks the other skipped, permanently | Give each branch its own terminal node |
| `Step "x" was not found or has not produced output yet` | A reference to a step that has not run; there is no default syntax | Restructure so the reference is always valid |
| Builder refuses to save: missing a connection from `Then` | The API allows a dangling branch, the builder does not | Put the used outcome on `Then` and invert the test |
| Loop exits immediately, or never | `do_while` is evaluated **after** the body | Write the condition against the body's own output |
| Condition errors on `true`/`false` | Bare booleans are read as step names | Compare strings: the agent answers `yes` and `no` |
| Approval clicked, nothing runs after it | The approval's `decision_window` elapsed, which fails the run; approving later does nothing | Raise `decision_window` on the approval node, and finish the stranded run by hand (below) |


### Approvals expire, and the run dies quietly

The approval node's `decision_window` is how long it will wait, in seconds.
When it elapses, the run **fails**. Approving afterwards does nothing: there is
no error in the GUI, no notification, and the image that was built and waiting
is never published.

Two things about this are worth knowing before you trust it unattended.

**`fallback_decision` does not route.** The node carries
`"fallback_decision": "reject"`, and its stored record names
`notify_rejected` as the step for a rejection — but an expiry does not take
that edge. The execution simply fails, and nothing on the rejected path runs.
Set a window long enough that the fallback never has to be used.

**The error names the wrong setting.** An expired run reports `The step
"Human Approval" did not finish within 30 seconds (configured in the node
Timeout setting)`. There is no 30-second timeout involved; the number is
wrong and so is the setting it points at. The field to change is
`decision_window` on the approval node.

Read the approval record itself rather than the error, through
`/api/v1/approvals?limit=20`. An expired one says so plainly:

```
status:         expired
created_at:     2026-11-07T06:06:35Z
timeout_at:     2026-11-07T10:06:35Z
decision_notes: Approval decision window expired
decided_by:     worker.ao.svc  (a service, not a person)
```

That endpoint needs an account that can read the project. The service account
the chain runs as cannot: scoped to triggering the webhook, it reads **empty
lists** from `/api/v1/workflows`, `/api/v1/executions` and `/api/v1/approvals`
rather than an error, which looks exactly like "nothing pending". Log in as the
admin from `ao_password`, the same way `setup/configure_ao.yml` does, or give
the service account a read-only project role.

#### Finishing a stranded run by hand

The two steps after the gate can be run by hand, because the build leaves
everything they need behind. The build directory under
`/var/tmp/ee-builds/<name>-<short sha>/` and the local image both survive, and
the build job's `artifacts` carry the rest.

1. Read `artifacts` from the **EE Build | Build Image** job: `ee_name`,
   `ee_tag`, `ee_local_image`, `ee_definition_path`, `ee_definition_file`,
   `ee_repo_url`, `ee_git_ref`.
2. Read `artifacts` from the **EE Build | Apply AI Fix** job: `ee_ai_summary`,
   `ee_ai_root_cause`, `ee_ai_category`.
3. Launch **EE Build | Register EE** with the first three variables. It answers
   with `ee_registered_image`.
4. Launch **EE Build | Commit AI Fix** with the rest, plus
   `ee_registered_image` and `ee_job_url` for the commit message.

Run them in that order, so the commit message can cite the image that shipped.
Both are safe if the orchestrator later wakes up and runs them itself: the
register step re-pushes the same tags, and the commit step compares the file on
the branch with what was built and reports `already_current` instead of writing
a second commit.
---

## What was tested

AAP 2.7 (controller 4.8.8) with Event-Driven Ansible and private automation
hub 4.12.2; automation orchestrator 2026.8 on MicroShift; automation portal
2.7 (Backstage) on a RHEL appliance; RHEL 9 build host with podman 5.4 and
ansible-builder 3.1.1; OpenAI as the LLM provider; a private GitHub definitions
repository, and a self-hosted GitLab CE 19.5 one on the build host itself.

Exercised end to end on GitHub: a portal-built EE from its pull request through
to registration; a hand-pushed definition repaired twice by the agent and
approved; rejection; the agent declining to retry; cleanup and its rerun; and a
private repository with a token.

Then moved to GitLab and exercised again: a hand-pushed definition with two
separate faults, repaired twice by the agent, registered in the hub and the
controller, and the fix committed back with the loop marker, which the forward
step correctly ignored on its way back round. A merge request titled the way
the portal titles them was merged by the rule in Phase 11, and that merge built
and registered an execution environment without anyone touching it.

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
| Build from the **automation portal's GUI builder**? | Phase 9. Needs a portal, a GitHub OAuth app and a token. | Skip Phase 9; definitions are written by hand. |
| Keep the definitions repository **private**? | Phase 10. Recommended: the portal writes your hub's hostname into each EE. | Skip Phase 10. |
| **Merge the portal's pull requests automatically**? | Phase 11. Fully hands-off, and removes the last human checkpoint. | Skip Phase 11; merge them yourself. |
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
| GitHub token | Fine-grained, on the definitions repository, **Contents: read and write** and **Pull requests: read and write** |
| An HMAC secret | `openssl rand -hex 32`, shared between GitHub and the event stream |
| LLM provider and API key | Phase 8 only. Read its caveats before choosing a provider. |
| GitHub OAuth app client ID and secret | Phase 9 only |

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

Two GitHub registrations, both created by the human:

- an **OAuth app** — the builder pushes as the signed-in user. Callback:
  `https://<portal>/api/auth/github/handler/frame`
- a **token** — the portal reads repositories and registers catalog entries

Add both to `/etc/portal/configs/app-config/app-config.production.yaml`, along
with the hub collection provider, then restart. See
[portal/README.md](portal/README.md) for the exact blocks.

**Verify:**

```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://<portal>/api/auth/github/handler/frame
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

## Phase 11: Merge the portal's pull requests automatically (optional)

Copy
[`definitions-repo/.github/workflows/auto-merge-portal-ee.yml`](definitions-repo/.github/workflows/auto-merge-portal-ee.yml)
into the **definitions** repository. It merges only pull requests whose title starts with
`[AAP] Adds/updates files for Execution Environment`, raised from a branch in
the same repository by its owner.

**Verify:** a portal pull request merges itself, and the merge — authored by
`github-actions[bot]` — still triggers the chain. It does: Actions suppresses
workflow triggers from its own token, but webhooks fire regardless.

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
| Happy path | Push a valid definition | Builds first try, registers, no approval |
| Repair | Push one with a misspelled collection | Fails, agent corrects it, rebuilds, waits for approval |
| Approval | Approve it | Registers, and commits the fix back (Phase 12) |
| Rejection | Reject one | `notify_rejected` runs, nothing is published |
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
| Event stream logs "Signature mismatch" | Webhook secret does not match the credential | Reset both from `github_hmac_secret` — watch for a trailing comment being captured with the value |
| Activation dies on startup | A rulebook regex error, for example `[skip-ee-build]` read as a character range | Escape it, or match on plain text |
| Activation cannot be updated | A running activation refuses PATCH | Delete and recreate it |
| Forward job says "Nothing to build" on a private repo | No token | Phase 10 |
| Definition deleted, no build | Deletions are not in a push's added or modified lists | Working as intended |

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

---

## What was tested

AAP 2.7 (controller 4.8.8) with Event-Driven Ansible and private automation
hub 4.12.2; automation orchestrator 2026.8 on MicroShift; automation portal
2.7 (Backstage) on a RHEL appliance; RHEL 9 build host with podman 5.4 and
ansible-builder 3.1.1; OpenAI as the LLM provider; a private GitHub definitions
repository.

Exercised end to end: a portal-built EE from its pull request through to
registration; a hand-pushed definition repaired twice by the agent and approved;
rejection; the agent declining to retry; cleanup and its rerun; and a private
repository with a token.

# Automation portal: publishing EE definitions to GitLab

The portal's execution environment builder writes a definition and then either
offers it as a download or pushes it to source control. This chain needs the
push, so the portal has to know about the forge twice:

| What | Why | Where |
|---|---|---|
| An **OAuth application** | The builder pushes as *the signed-in user*, with their OAuth token | `auth.providers.gitlab` |
| An **access token** | The portal reads projects and registers catalog entries | `integrations.gitlab` |

GitHub works the same way, with `auth.providers.github` and
`integrations.github`, and the builder has a `publish:github:pull-request`
step beside the GitLab one. **Prefer a forge the event stream can be reached
from.** A hosted forge has to connect *inward* to the Event-Driven Ansible
event stream to report a push, and if that hostname resolves to a private
address from outside, every delivery fails with `failed to connect to host`
and nothing in AAP records the attempt. An in-lab GitLab has no such problem,
which is why this is written for GitLab.

Two playbooks do the work. Run them in order.

## 1. Create the OAuth application and the token

```bash
# job template: EE Build | Portal GitLab Credentials
ansible-playbook playbooks/portal_gitlab.yml \
  -e portal_fqdn=<portal-fqdn> -e gitlab_external_url=<gitlab-url>
```

It creates an instance-wide OAuth application whose callback is
`https://<portal-fqdn>/api/auth/gitlab/handler/frame`, with the scopes the
builder needs (`api read_user read_repository write_repository` plus the
OpenID ones Backstage resolves the user with), and an access token scoped to
`api`. Both are written to `/root/portal-gitlab.yml` on the GitLab host and
nowhere else, so neither reaches a job's event records.

GitLab stores an OAuth secret hashed. An application that already exists
cannot be read back, so re-running issues a fresh secret and the portal has to
be given the new one.

On GitLab 18 and later an OAuth application must belong to an organization.
That is a database constraint rather than a model validation, so a missing one
arrives as `PG::CheckViolation` from under ActiveRecord instead of anything
readable. The playbook sets the default organization.

## 2. Put them in the appliance config

```bash
# job template: EE Build | Point Portal At GitLab
ansible-playbook playbooks/portal_gitlab_config.yml \
  -e portal_host=<inventory-name> -e portal_fqdn=<portal-fqdn> \
  -e gitlab_external_url=<gitlab-url>
```

It reads the credentials off the GitLab host, merges three things into
`/etc/portal/configs/app-config/app-config.production.yaml`, keeps a
timestamped copy of the previous file, parses the new one before restarting,
and then waits for the provider to answer.

What it merges, in case you would rather do it by hand:

```yaml
integrations:
  gitlab:
    - host: <gitlab-host>
      token: <access token>
      baseUrl: <gitlab-url>
      apiBaseUrl: <gitlab-url>/api/v4

auth:
  environment: production
  providers:
    gitlab:
      production:
        clientId: <oauth client id>
        clientSecret: <oauth client secret>
        audience: <gitlab-url>
        callbackUrl: https://<portal-fqdn>/api/auth/gitlab/handler/frame

catalog:
  locations:
    - type: url
      target: <gitlab-url>/<group>/<project>/-/raw/main/.portal/ee-builder-preset.yaml
      rules:
        - allow: [Template]
```

`audience` is what makes the Backstage GitLab provider talk to a self-hosted
instance instead of gitlab.com. Keep the existing `auth.providers.rhaap`
block: people still sign in to the portal with AAP, and GitLab is only there
so the builder can push as them.

The merge rewrites the file rather than patching it, so comments in it do not
survive — that is what the timestamped copy is for.

**Verify:**

```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://<portal-fqdn>/api/auth/gitlab/handler/frame
```

`404` means the provider is not registered. `400` means it is: the handler was
called without an authorization code, which is all this request is. Give the
portal a minute after a restart — it answers 404 until the auth plugin
finishes initializing, and `journalctl -u portal` prints
`Configuring auth provider: gitlab` when it is ready.

## 3. Point the portal at private automation hub

Without this the portal's **Collections** page says *No content sources
configured* and the builder has nothing to offer. Add to
`app-config.production.yaml`, under `catalog.providers.rhaap.production.sync`:

```yaml
        pahCollections:
          enabled: true
          repositories:
            - name: rh-certified
              schedule:
                frequency: {hours: 6}
                timeout: {minutes: 60}
            - name: validated
              schedule:
                frequency: {hours: 6}
                timeout: {minutes: 60}
            - name: published
              schedule:
                frequency: {hours: 6}
                timeout: {minutes: 60}
            - name: community
              schedule:
                frequency: {hours: 6}
                timeout: {minutes: 60}
```

It reuses the AAP connection the portal already has, so no extra credentials.
Restart, then watch it work:

```bash
sudo journalctl -u portal -f | grep PAHCollectionProvider
```

Each repository logs `Fetched N collections`. An empty repository is fine; it
just contributes nothing.

There is a second provider, `ansibleGitContents`, that crawls GitHub or GitLab
organizations for `galaxy.yml` files. It is off here, since the hub is the
approved source.

## 4. Prefill the builder's publishing target

By default the builder asks for the provider, the namespace and the repository
name on every execution environment, even though the answer is always the same.
The wizard is a Backstage template, so those fields take defaults.

[`templates/ee-builder-preset.yaml`](templates/ee-builder-preset.yaml) is Red
Hat's *start from scratch* template with two changes:

- the provider list holds one entry, so there is nothing to select
- `sourceControlProvider` carries a default of your namespace and repository

Replace every `CHANGEME_` value:

| Placeholder | For GitLab | For GitHub |
|---|---|---|
| `CHANGEME_SCM_PROVIDER` | `gitlab` | `github` |
| `CHANGEME_SCM_LABEL` | `Gitlab` | `Github` |
| `CHANGEME_SCM_HOST` | your GitLab hostname | `github.com` |
| `CHANGEME_SCM_OWNER` | the group | the owner |
| `CHANGEME_DEFINITIONS_REPO` | the project name | the repository name |

The defaults sit on `sourceControlProvider`'s sub-properties rather than on the
object. A default on the object itself is not applied to a field inside a
`dependencies` branch, so `ScmSelector` never sees it.

Put the file where the portal can read it — the definitions repository is a
good home, because the portal already has a token for it — and register it
with the `catalog.locations` block above. Restart, and **Create** lists
*Execution Environment (preset repository)* beside the stock templates. Keep
the stock ones registered: this copy is pinned to v2.0.1 of the upstream
template and will not pick up Red Hat's changes.

A second way to prefill, with no config change: every EE the builder publishes
includes `<name>-template.yml`, a saved template of that build. Register one of
those and the wizard opens with *everything* prefilled — collections included —
which suits cloning an existing EE rather than starting a new one.

## 5. Build a definition

In the portal: **Create** > the execution environment template > fill in the
collections, Python and system dependencies > choose to publish to source
control, with the EE definitions project as the target.

The builder creates:

```
<ee-name>/
  <ee-name>.yml          the definition
  README.md
  ansible.cfg
  catalog-info.yaml
  NEXT_STEPS.md
```

It opens a merge request titled `[AAP] Adds/updates files for Execution
Environment <name>` on a branch named after the definition.

**Merging that request is what starts the build**: the push to `main` goes to
the Event-Driven Ansible event stream, which starts the Build-EE workflow in
automation orchestrator.

The builder also offers to build the image itself, in GitHub Actions or a
GitLab pipeline. Leave that unchecked: this chain does the build on your own
build host, against your own hub, and running both just builds everything
twice.

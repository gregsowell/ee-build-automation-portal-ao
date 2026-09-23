# Automation portal: publishing EE definitions to GitHub

The portal's execution environment builder writes a definition and then either
offers it as a download or pushes it to source control. This chain needs the
push, so the portal has to know about GitHub twice:

| What | Why | Where |
|---|---|---|
| A GitHub **OAuth App** | The builder pushes as *the signed-in user*, with their OAuth token | `auth.providers.github` |
| A GitHub **token** (PAT or GitHub App) | The portal reads repositories and registers catalog entries | `integrations.github` |

## 1. Create the OAuth App (do this yourself)

GitHub > Settings > Developer settings > OAuth Apps > **New OAuth App**

| Field | Value |
|---|---|
| Application name | Ansible automation portal |
| Homepage URL | `https://<portal-fqdn>` |
| Authorization callback URL | `https://<portal-fqdn>/api/auth/github/handler/frame` |

Keep the client ID and secret; they go in `secrets.yml`, never in this repo.

## 2. Create the repository token

A fine-grained personal access token limited to the EE definitions repository,
with **Contents: read and write** and **Pull requests: read and write**. The
builder opens a pull request on an existing repository and creates the
repository outright when it does not exist yet.

## 3. Add both to the appliance config

On the portal appliance, edit
`/etc/portal/configs/app-config/app-config.production.yaml`. Back it up first:

```bash
sudo cp /etc/portal/configs/app-config/app-config.production.yaml \
        /root/app-config.production.yaml.$(date +%Y%m%d-%H%M%S)
```

Add (merging with what is already there):

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}

auth:
  environment: production
  providers:
    github:
      production:
        clientId: ${GITHUB_CLIENT_ID}
        clientSecret: ${GITHUB_CLIENT_SECRET}
```

Keep the existing `auth.providers.rhaap` block: people still sign in to the
portal with AAP, and GitHub is only there so the builder can push as them.
Substitute the real values, or point them at the appliance's environment file.
Then restart:

```bash
sudo ansible-portal restart
sudo ansible-portal status
```

Check it took:

```bash
curl -sk -o /dev/null -w '%{http_code}
' https://<portal-fqdn>/api/auth/github/handler/frame
```

`404` means the provider is not registered. `400` means it is (the handler was
called without an authorization code). Give the portal a minute after a
restart: it answers 404 until the auth plugin finishes initializing, and
`journalctl -u portal` prints `Configuring auth provider: github` when it is
ready.

The cloud-init reference for the appliance also accepts `integrations.github.*`
at first boot.

## 4. Point the portal at private automation hub

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

## 5. Build a definition

In the portal: **Create** > the execution environment template > fill in the
collections, Python and system dependencies > choose to publish to source
control, with the EE definitions repository as the target.

The builder creates:

```
<ee-name>/
  <ee-name>.yml          the definition
  README.md
  ansible.cfg
  catalog-info.yaml
  NEXT_STEPS.md
.github/workflows/ee-build.yml
```

It opens a pull request titled `[AAP] Adds/updates files for Execution
Environment <name>` on a branch named after the definition.

**Merging that pull request is what starts the build**: the push to `main` goes
to the Event-Driven Ansible event stream, which starts the Build-EE workflow in
automation orchestrator.

The `.github/workflows/ee-build.yml` file that the portal adds builds the image
in GitHub Actions. It is harmless, but this chain does the build on your own
build host, so disable that workflow in the repository unless you want both.

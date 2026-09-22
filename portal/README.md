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

Put the values in the appliance's environment file rather than in the YAML, so
the config itself holds no secrets. Then restart:

```bash
sudo ansible-portal restart
sudo ansible-portal status
```

Check the generated `/etc/portal/configs/app-config/app-config.yaml` for the
keys the appliance already sets, and match its structure if it differs from the
snippet above. The cloud-init reference for the appliance also accepts
`integrations.github.*` at first boot.

## 4. Build a definition

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

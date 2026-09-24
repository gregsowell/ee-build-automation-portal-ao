# Files for the definitions repository

These belong in the *other* repository — the one the portal publishes execution
environment definitions to — not in this one.

| Path | Purpose |
|---|---|
| `.github/workflows/auto-merge-portal-ee.yml` | Merges the pull requests the portal opens, so an EE goes from the builder to a registered image with no clicks. Phase 11 of [INSTRUCTIONS.md](../INSTRUCTIONS.md). |

Copy it in and commit:

```bash
mkdir -p .github/workflows
cp <this-repo>/definitions-repo/.github/workflows/auto-merge-portal-ee.yml .github/workflows/
git add .github/workflows/auto-merge-portal-ee.yml
git commit -m "Merge the portal's execution environment pull requests automatically"
git push
```

The rest of that repository is written by the portal: one directory per
execution environment, holding the definition, a README, an `ansible.cfg`, a
`catalog-info.yaml` and a saved builder template.

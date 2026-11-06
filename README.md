# Silta simple project template

A static site served by nginx with the Silta `simple` chart. Put the site's files in `web/`:
they are the build context of `silta/nginx.Dockerfile`, which copies them into the image.

## ArgoCD deployment

Projects created from this template can deploy with ArgoCD instead of Silta's CircleCI jobs.
The platform side lives in [wunderio/argocd-cluster-management](https://github.com/wunderio/argocd-cluster-management)
(`templates/argocd/README.md` there).

- `k8s/staging/`: the `main`/`master` branch deploys to the `staging` environment.
- `k8s/feature/`: pull requests from `feature/*`, `hotfix/*` and `epic/*` deploy to `pr-<N>`
  environments, removed when the pull request closes.
- `.github/workflows/build.yml` builds the images, pushes them to `registry.wdr.io/<repository>/` and
  commits the new tags to `k8s/<overlay>/values/simple.yaml`. ArgoCD deploys that commit.

Add `k8s/develop/` or `k8s/production/` (copies of `k8s/staging/`) to deploy the `develop` or
`production` branch too. Every project needs `k8s/staging/apps.yaml` on `main`/`master`: the
staging environment creates the project's ArgoCD project.

**This template repository never deploys itself.** ArgoCD only picks up repositories named
`client-*` or `internal-*` that have the `silta-argocd` topic. The build workflow skips the
image and tag jobs when the repository is a template.

### Setting up a new project

1. Create the repository from this template with a `client-*` or `internal-*` name.
2. Replace the `PROJECT_NAME` placeholders with the repository name in lowercase. It is the
   Kubernetes namespace and the Harbor project name:

   ```bash
   name=$(basename "$(git rev-parse --show-toplevel)" | tr '[:upper:]' '[:lower:]')
   grep -rl PROJECT_NAME k8s | xargs sed -i "s/PROJECT_NAME/$name/g"
   ```

3. Remove the Silta deploy jobs from `.circleci/config.yml` so the project doesn't deploy twice.
4. Add the `silta-argocd` topic to the repository (repository page → About → Topics). New
   repositories don't inherit the template's topics.

The build workflow uses the organisation's `REGISTRY_USERNAME`, `REGISTRY_PASSWORD`,
`ARGOCD_APP_ID` and `ARGOCD_APP_PRIVATE_KEY` secrets. The tag commit is made with the
ArgoCD GitHub App so it passes branch protection.

### Secrets

Put Kubernetes Secrets in `k8s/<overlay>/secrets/<name>.sops.yaml`, encrypt them, and set
`secrets.enabled: true` in that overlay's `apps.yaml`. `.sops.yaml` encrypts them to the shared
tenant key, which only ArgoCD can decrypt:

```bash
sops --encrypt --in-place k8s/staging/secrets/app.sops.yaml
```

A file may hold only `apiVersion: v1`, `kind: Secret` objects in the project's namespace;
anything else fails the deployment. Never commit a decrypted file.

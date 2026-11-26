# Silta simple project template

A static site served by nginx with the Silta `simple` chart. Put the site's files in `web/`:
they are the build context of `silta/nginx.Dockerfile`, which copies them into the image.

## ArgoCD deployment

Projects created from this template deploy with [ArgoCD](https://argo-cd.readthedocs.io/) from
`k8s/`; there is no CircleCI deployment. The ArgoCD platform reads `k8s/<overlay>/apps.yaml`,
deploys the [Silta `simple` chart](https://github.com/wunderio/charts/tree/master/simple) with the
values in `k8s/<overlay>/values/`, and adds the cluster's defaults (domain, TLS, storage).

- `k8s/staging/`: the `main`/`master` branch deploys to the `staging` environment.
- `k8s/feature/`: a pull request from `feature/*`, `hotfix/*` or `epic/*` gets a `pr-<N>`
  environment only while it carries the `preview` label (create the label in the repository if
  it does not exist yet). Removing the label or closing the pull request deletes the environment.
- `.github/workflows/build.yml` builds the images on each push to a branch with an overlay
  here with [wunderio/argocd-actions](https://github.com/wunderio/argocd-actions), pushes them
  to `<registry>/<project>/` and commits the new tags to
  `k8s/<overlay>/values/simple.yaml`. ArgoCD deploys that commit. Other branches and pull
  requests are validated only.

Add `k8s/develop/` or `k8s/production/` (copies of `k8s/staging/`) to deploy the `develop` or
`production` branch too. CI updates the tags of `main`/`master`, `develop`, `production` and pull
request branches, so a release is a merge to `production`: CI builds the `production` images,
commits their tags to `k8s/production/values/` and ArgoCD deploys that commit. Until
`k8s/production/` exists, `production` builds no images. To set production tags by hand instead,
set `COMMIT_PRODUCTION_TAGS` to `"false"` in `build.yml`. Every project needs
`k8s/staging/apps.yaml` on `main`/`master`: the staging environment creates the project's ArgoCD
project.

**This template repository never deploys itself.** The ArgoCD platform deploys only the
repositories it is set up to discover, and the build workflow skips the image and tag jobs when
the repository is a template.

### Setting up a new project

1. Create the repository from this template and make it discoverable as your ArgoCD platform
   requires (for example its name and a repository topic; GitHub copies no topics from a
   template).
2. Replace the placeholders. Your platform team has the values:
   - `PROJECT_NAME`: the repository name in lowercase. It is the Kubernetes namespace and the
     project in the container registry.
   - `REGISTRY_HOST`: the [Harbor](https://goharbor.io/) registry the clusters pull images from.
   - `AGE_RECIPIENT`: the platform's age public key for the project's secrets.

   ```bash
   name=$(basename -s .git "$(git remote get-url origin)" | tr '[:upper:]' '[:lower:]')
   registry=registry.example.com        # container registry host
   age_recipient=age1...                # the platform's age public key
   git grep -lz -e PROJECT_NAME -e REGISTRY_HOST -e AGE_RECIPIENT -- ':!README.md' \
     | xargs -0 perl -pi -e "s/PROJECT_NAME/$name/g; s/REGISTRY_HOST/$registry/g; s/AGE_RECIPIENT/$age_recipient/g"
   ```

The build workflow needs these repository or organisation secrets:

- `REGISTRY_USERNAME`, `REGISTRY_PASSWORD`: a registry account that can push, pull and list the
  project's images, and create the project on the first build.
- `ARGOCD_APP_ID`, `ARGOCD_APP_PRIVATE_KEY`: a GitHub App allowed to push to the deployed
  branches, so the tag commit passes branch protection.

### Secrets

Put Kubernetes Secrets in `k8s/<overlay>/secrets/<name>.sops.yaml`, encrypt them, and set
`secrets.enabled: true` in that overlay's `apps.yaml`. `.sops.yaml` encrypts them to the
platform's age key, which only ArgoCD can decrypt:

```bash
sops --encrypt --in-place k8s/staging/secrets/app.sops.yaml
```

A file may hold only `apiVersion: v1`, `kind: Secret` objects in the project's namespace;
anything else fails the deployment. Never commit a decrypted file.

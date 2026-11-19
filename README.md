# fint-core-workflows

Reusable GitHub Actions workflows for the Core team repos.

| Workflow | Use |
|---|---|
| `cd-beta-api.yaml` | Deploy cycle like fint-core: `develop` to beta, `main` to api (production), with Kustomize |
| `CD.yaml` | Older build and deploy workflow for one org and one cluster |
| `CI.yaml` | Older Gradle build workflow |

## cd-beta-api.yaml

### What it does
1. **Environment:** a push to `develop` deploys to beta (`aks-beta-fint-2021-11-23`). A push to `main` deploys to api (`aks-api-fint-2022-02-08`).
2. **Image:** `ghcr.io/fintlabs/<image-name>:sha-<commit>`. The commit is the last one that changed the app.
   A commit that only changes `kustomize/`, `.github/` or `*.md` does not give a new tag.
3. **Build:** the workflow skips the build when the image is already in GHCR. So a Kustomize change goes straight to deploy.
4. **Deploy:** the workflow finds the overlays by itself and deploys each one with Kustomize.
   It also sets the image in FLAIS `Application` resources.

| Overlay layout | Targets |
|---|---|
| `kustomize/overlays/<env>/kustomization.yaml` | one target. The namespace comes from the `namespace` field (or the `namespace` input). |
| `kustomize/overlays/<env>/<org>/kustomization.yaml` | one target for each org folder. The namespace is the folder name. |

A folder that ends with `.disabled` is skipped. If an environment has no overlay, the workflow deploys nothing to it.

### Use it
Add `.github/workflows/cd.yaml` to the app repo:

```yaml
name: CD

on:
  push:
    branches: [main, develop]

concurrency:
  group: deploy-${{ github.ref_name }}
  cancel-in-progress: false

jobs:
  cd:
    uses: FINTLabs/fint-core-workflows/.github/workflows/cd-beta-api.yaml@main
    permissions:
      contents: read
      packages: write
    secrets: inherit
```

### More than one image (for example fint-core)
Give the `services` input: a JSON list with one item for each image. Each item has `image` (required),
`dockerfile`, `context` and `paths`. Each image gets its own tag from the last commit on its paths.
The workflow builds each image and sets all of them in the same overlays.

```yaml
jobs:
  cd:
    uses: FINTLabs/fint-core-workflows/.github/workflows/cd-beta-api.yaml@main
    permissions:
      contents: read
      packages: write
    secrets: inherit
    with:
      exclude-paths: ""
      services: |
        [
          {
            "image": "fint-core-client-api-db",
            "dockerfile": "fint-core-client-api/Dockerfile",
            "paths": "fint-core-client-api fint-core-shared buildSrc build.gradle.kts settings.gradle.kts gradle.properties gradle gradlew gradlew.bat"
          },
          {
            "image": "fint-core-adapter-gateway-db",
            "dockerfile": "fint-core-adapter-gateway/Dockerfile",
            "paths": "fint-core-adapter-gateway fint-core-shared buildSrc build.gradle.kts settings.gradle.kts gradle.properties gradle gradlew gradlew.bat"
          }
        ]
```

With these inputs, the tags are the same as the ones that fint-core's own `cd.yml` makes.

### Inputs
All inputs are optional.

| Input | Default | Description |
|---|---|---|
| `image-name` | repository name | image name in `ghcr.io/fintlabs` (one image) |
| `dockerfile` | `Dockerfile` | path to the Dockerfile (one image) |
| `context` | `.` | Docker build context (one image) |
| `app-paths` | `.` | paths whose last commit gives the image tag (one image) |
| `services` | none | JSON list for more than one image. When set, the four inputs above are not used. |
| `exclude-paths` | `kustomize .github *.md` | paths that do not change any image. Use `""` to turn this off. |
| `build-args` | none | Docker build arguments for all images, one `KEY=value` for each line |
| `kustomize-dir` | `kustomize/overlays` | folder with one overlay folder for each environment |
| `namespace` | overlay `namespace` field | namespace for a flat overlay |

### Before the first run
- The repo must have access to the org secrets `AKS_BETA_FINT_GITHUB` and `AKS_API_FINT_GITHUB`.
- The deploy job uses the GitHub environments `beta` and `api`. GitHub makes them on the first run.
  Add protection rules to `api` if a person must approve production deploys.
- Compare the overlays with the cluster before the first deploy, for example with `kubectl diff -k kustomize/overlays/beta`.
  Changes made by hand in the cluster are lost on the first automated deploy.
- Repos with more than one image use the `services` input (see above).

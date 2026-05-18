# fint-core-workflows
This repository contains reusable workflows for Fint.

The CI workflow will build and test the project.

The CD forkflow will build and deploy.

Use the examples in [CD](#cd---deployment) chapter, to make the workflow deploy to API when `main` branch is changed
and to BETA when `develop` branch is changed.

## CI - Building and Testing
For PRs, each time you push to the branch, it should try to build the project and run tests.

### To use the CI workflow
In your repository, in the folder `.github/workflows` create a `CI.yaml` file.
Copy this inside it:
```yaml
name: CI

on:
  pull_request:

jobs:
  ci:
    uses: FINTLabs/fint-core-workflows/.github/workflows/CI.yml@<latest-commit-hash>
    with:
      java-version: '25' # This can be changed to any Temurin java version
```
On creation of PR or when you push the branch that the PR uses, it will build and run tests.

### Custom gradlew command
Some projects uses custom gradlew commands. You can change this with the `gradle-command` parameter:

```yaml
...
      uses: FINTLabs/fint-core-workflows/.github/workflows/CI.yml@<latest-commit-hash>
      with:
        java-version: '25'
        gradle-command: 'check' # Will now run ./gradlew check, instead of ./gradlew build
```

## CD - Deployment
This workflow will deploy what is in the kustomize folder of your project. See [kustomize](#Kustomize) chapter for more info.

Due to the way we deploy things, namely that each repo will have to deploy to different counties in the different
environments, we have to write each namespace in the CD.yaml file.

In your repository, in the folder `.github/workflows` create a `CD.yaml` file.

Paste the following inside:
```yaml
name: Deploy

on:
  push:
    branches:
      - main
      - develop

jobs:
  deploy-api:
    if: github.ref == 'refs/heads/main'
    uses: FINTLabs/fint-core-workflows/.github/workflows/CI.yml@<latest-commit-hash>
    with:
      environment: api
      namespaces: '["afk-no","nfk-no","vlfk-no"]' # Edit to relevant namespaces
    secrets: inherit

  deploy-beta:
    if: github.ref == 'refs/heads/develop'
    uses: FINTLabs/fint-core-workflows/.github/workflows/CI.yml@<latest-commit-hash>
    with:
      environment: beta
      namespaces: '["afk-no","fintlabs-no"]' # Edit to relevant namespaces
    secrets: inherit
```

The resulting images from the build is tagget as follows:
```ghcr.io/fintlabs/<repository-name>:<commit-hash>```


### Kustomize
Kustomize is used to build the kubernetes files for each namespace.
Kustomize uses the following structure in your repo:

```
kustomize/
├── base
│   ├── flais.yaml
│   └── kustomization.yaml
└── overlays
    ├── alpha
    │   └── fintlabs-no
    │       └── kustomization.yaml
    ├── api
    │   ├── agderfk-no
    │   │   └── kustomization.yaml
    │   ├── mrfylke-no
    │   │   └── kustomization.yaml
    │   ├── nfk-no
    │   │   └── kustomization.yaml
    └── beta
        └── agderfk-no
            └── kustomization.yaml
```



# Deploy To Argo CD GitHub Action

Use this composite action to update Kubernetes manifests with a Docker image
digest and trigger an Argo CD application sync from within your CI/CD workflows.
The action pulls your target image, rewrites `image:` fields in the specified
deployment YAMLs, pushes the changes to your manifests repository, and performs
an Argo CD login + `app sync`.

> [!WARNING]
> Make sure you test on a different branch to make your life easier :)
>
> - Keep `target-branch` pointed at a disposable branch while iterating on the action so you can verify manifest changes without touching the production branch. You will have to delete the test branch every time you run the action.
> - Use a GitHub environment or repository secret for every sensitive input (`k8s-repo-token`, Argo CD credentials) to avoid leaking credentials in workflow logs.

## Usage

```yaml
name: Deploy to Prod

on:
  workflow_dispatch:
    inputs:
      image-tag:
        description: "Image tag to deploy"

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout this repo
        uses: actions/checkout@v4

      - name: Deploy to ArgoCD
        uses: oncokb/github-actions/deploy-to-argo-cd-action@main
        with:
          image-repo: "mskcc/oncokb-data-release"
          image-tag: ${{ github.event.inputs.image-tag }}

          k8s-repo-token: ${{ secrets.PRIVATE_REPO_TOKEN }}

          deployment-file-path: "argocd/aws/<aws-id>/clusters/path/to/release/file/dir"
          # you can pick more than one file in the same directory (space separated)
          deployment-file-names: "<release file>.yaml"

          argocd-server: "<argocd hostname>"
          argocd-app: "<argocd-app-name>" # this is the collection of pods
          argocd-resource-options: "--resource apps:Deployment:<k8-namespace>/<k8-name>"
          argocd-username: ${{ secrets.ARGOCD_USERNAME }}
          argocd-password: ${{ secrets.ARGOCD_PASSWORD }}
```

## Key Inputs

| Input                                                                       | Required | Description                                                                                                                                                                                                                         |
| --------------------------------------------------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `image-repo`, `image-tag`                                                   | ✅       | Image repo + tag to deploy. The action records the image digest and updates your manifests to match.                                                                                                                                |
| `k8s-repo`                                                                  | ❌       | Defaults to `knowledgesystems/knowledgesystems-k8s-deployment`. Override when your manifests live elsewhere.                                                                                                                        |
| `k8s-repo-token`                                                            | ✅       | Token with push access to the manifests repo.                                                                                                                                                                                       |
| `deployment-file-path`, `deployment-file-names`                             | ✅       | Folder and space-separated file names that should receive the new digest.                                                                                                                                                           |
| `git-user-name`, `git-user-email`                                           | ❌       | Identify the bot account that commits manifest changes. Defaults cover most uses.                                                                                                                                                   |
| `target-branch`                                                             | ❌       | Defaults to `master`. Override when testing changes in a throwaway branch or when your manifests repo uses a non-default branch. Setting this explicitly is useful when rehearsing the workflow before merging to your main branch. |
| `argocd-server`, `argocd-username`, `argocd-password`, `argocd-cli-version` | Mixed    | Control the CLI login performed by the action.                                                                                                                                                                                      |
| `argocd-app`, `argocd-resource-options`                                     | ✅       | Identify which Argo CD application to sync and optionally restrict the sync to specific resources.                                                                                                                                  |

## Finding the Argo CD values

1. Log into your Argo CD dashboard (or run `argocd app list`). The application name in the Applications view is the value for `argocd-app`.
2. Click the application to inspect managed resources. Each resource row lists the API group, kind, namespace, and name. Compose the `argocd-resource-options` string with this information, for example `--resource apps:Deployment:namespace/name` to target a single Deployment.

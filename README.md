<p align="center">
    <h1 align="center">Build & Deploy Action</h1>
    <p align="center">Build a Dockerfile and deploy to <a href="https://www.deploys.app">Deploys.app</a> — no secrets required.</p>
</p>

Pushing to your default branch builds and deploys to production. Opening a pull
request builds and deploys a **temporary preview** (`<name>-pr-<n>`) that gets a
sticky preview comment and a GitHub *View deployment* button, and is deleted
automatically when the PR closes (with a TTL as the backstop).

Authentication is keyless: the workflow's GitHub OIDC token is exchanged for a
short-lived deploys.app credential. There is nothing to copy, rotate, or leak.

## Setup (once)

1. Install the **deploys.app GitHub App** on your repository.
2. Link the repository to your project and a service account
   (`deploys github link`, or the console's project settings). The service
   account needs `deployment.deploy`, `deployment.get`, `deployment.delete`,
   `registry.push`.

## Usage

```yaml
name: Deploy
on:
  push:
    branches: [main]
  pull_request:

permissions:
  id-token: write   # required — this is the credential
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: deploys-app/build-deploy-action@v1
      with:
        project: my-project
        location: gke.cluster-rcf2
        name: web
        port: 3000
```

Fork-opened pull requests don't receive OIDC tokens (a GitHub restriction), so
previews run only for same-repo branches.

## Inputs

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `project` | yes | | Project ID |
| `location` | yes | | Deploy location ID |
| `name` | yes | | Base deployment name; PRs deploy to `<name>-pr-<n>` |
| `context` | | `.` | Docker build context |
| `dockerfile` | | `<context>/Dockerfile` | Dockerfile path |
| `buildArgs` | | | Docker build args, one `KEY=VALUE` per line |
| `port` | | `8080` | Container port (WebService/TCPService) |
| `type` | | `WebService` | `WebService`, `Worker`, `TCPService`, `InternalTCPService` |
| `env` | | | Deployment env vars, one `KEY=VALUE` per line |
| `previewTtl` | | `7d` | Preview TTL (`30m`, `12h`, `7d`, …), refreshed on every push |
| `apiEndpoint` | | `https://api.deploys.app` | API endpoint |
| `registry` | | `registry.deploys.app` | Registry host |

## Outputs

| Name | Description |
| --- | --- |
| `url` | The deployed URL |
| `deployment` | The deployed deployment name |
| `environment` | `production`, or `pr-<n>` for previews |
| `image` | The pushed image in digest form |

## How it works

1. Exchanges the workflow's OIDC token (`aud: https://deploys.app`) at
   `github.exchangeToken` for a 1-hour deploys token scoped to the linked
   service account.
2. Reports `started` via `github.notify` (drives the GitHub deployment status).
3. Builds with Buildx (GitHub Actions cache enabled) and pushes to
   `registry.deploys.app/<project>/<name>:<sha>`, logging in with the same
   token.
4. Deploys the image by digest — previews carry a rolling TTL.
5. Reports `success` (preview URL lands on the PR) or `failure`.

Deploying a pre-built image with service-account secrets instead? Use
[deploys-app/deploys-action](https://github.com/deploys-app/deploys-action).

## License

MIT

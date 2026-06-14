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

## Static sites (`mode: static`)

Set `mode: static` to build a static site (Hugo or Node), upload it to
deploys.app object storage as an immutable content-addressed release, and deploy
it as a **Static** deployment — no Dockerfile, no image, no port, no registry.
Static deploys are served by the shared static-gateway (scale-to-zero, atomic
publish, instant rollback). PR previews and the rolling `previewTtl` work exactly
as for container deploys.

```yaml
# .github/workflows/deploy.yml
name: deploy
on:
  push:
    branches: [main]   # production
  pull_request:        # previews

permissions:
  id-token: write      # required — keyless OIDC -> deploys token
  contents: read
  pull-requests: write # sticky preview comment

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: deploys-app/build-deploy-action@v1
        with:
          project: deploys
          location: gke.cluster-rcf2
          name: website
          mode: static
          framework: hugo
          outputDir: public
          spa: false
          notFound: 404.html
          # baseUrl omitted -> the action injects the planned (prod/preview) URL
          # so sitemap.xml and RSS index.xml carry the correct host.
```

The action installs Hugo extended pinned from `.tool-versions` (SCSS needs the
extended binary), builds, then uploads only the files that changed (blobs are
content-addressed and deduped). For a site that builds with `hugo --minify`
(e.g. a Makefile `build:` target that uses it), set `buildCommand: hugo --minify`.

For a Node/Vite SPA: `framework: node`, `outputDir: dist`, `spa: true`. The
action runs `npm ci`/`npm install` then your `buildCommand` (default
`npm run build`). The base URL is exported as `BASE_URL` for `node` frameworks —
your framework must read its own base-path env (or pass `--base`); set `baseUrl`
explicitly if you need a specific value.

## Inputs

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `project` | yes | | Project ID |
| `location` | yes | | Deploy location ID |
| `name` | yes | | Base deployment name; PRs deploy to `<name>-pr-<n>` |
| `mode` | | `dockerfile` | `dockerfile` (build+push an image) or `static` (build+publish a static site) |
| `workingDirectory` | | `.` | Root folder of the app to build (for monorepos, e.g. `apps/web`). dockerfile mode: the build context and default Dockerfile resolve under it. static mode: the build runs inside it and `outputDir` is read relative to it |
| `context` | | `.` | Docker build context, resolved relative to `workingDirectory` (mode=dockerfile) |
| `dockerfile` | | `<workingDirectory>/<context>/Dockerfile` | Dockerfile path, resolved relative to `workingDirectory` (mode=dockerfile) |
| `buildArgs` | | | Docker build args, one `KEY=VALUE` per line (mode=dockerfile) |
| `framework` | | `auto` | mode=static: `auto` (Hugo via `.tool-versions`/`hugo.toml`/`config.toml`, else Node via `package.json`), `hugo`, `node`, `none` |
| `buildCommand` | | preset | mode=static: defaults to `hugo` (or `hugo --minify` if the Makefile uses it) / `npm run build`; required for `framework: none` |
| `outputDir` | | `public` | mode=static: built static tree to upload (`public` for Hugo, `dist`/`build` for Node) |
| `nodeVersion` | | `.nvmrc`/`.tool-versions` else `20` | mode=static, `node` framework: Node toolchain version |
| `spa` | | `false` | mode=static: SPA fallback to `index.html` on unknown routes (Hugo sites are not SPAs) |
| `notFound` | | `404.html` | mode=static: custom 404 document served on clean-URL misses when `spa: false` |
| `baseUrl` | | computed | mode=static: build-time base URL; if empty the action injects the planned deploy URL |
| `port` | | `8080` | Container port (mode=dockerfile, WebService/TCPService) |
| `type` | | `WebService` | mode=dockerfile: `WebService`, `Worker`, `TCPService`, `InternalTCPService` |
| `env` | | | Deployment env vars, one `KEY=VALUE` per line (mode=dockerfile; ignored for static — no runtime container) |
| `envGroups` | | | Env groups to attach, one per line or comma-separated; each must already exist in the project (mode=dockerfile; ignored for static) |
| `pullSecret` | | | Pull secret name for a private image registry (mode=dockerfile); the secret must already exist in the deploy location |
| `previewTtl` | | `7d` | Preview TTL (`30m`, `12h`, `7d`, …), refreshed on every push |
| `apiEndpoint` | | `https://api.deploys.app` | API endpoint |
| `registry` | | `registry.deploys.app` | Registry host (mode=dockerfile) |

## Outputs

| Name | Description |
| --- | --- |
| `url` | The deployed URL |
| `deployment` | The deployed deployment name |
| `environment` | `production`, or `pr-<n>` for previews |
| `artifact` | The deployed artifact: the pushed image in digest form (`mode: dockerfile`) or the static release-sha (`mode: static`) |
| `image` | Deprecated alias of `artifact`; kept for one minor version |

## How it works

**`mode: dockerfile` (default):**

1. Exchanges the workflow's OIDC token (`aud: https://deploys.app`) at
   `github.exchangeToken` for a 1-hour deploys token scoped to the linked
   service account.
2. Reports `started` via `github.notify` (drives the GitHub deployment status).
3. Builds with Buildx (GitHub Actions cache enabled) and pushes to
   `registry.deploys.app/<project>/<name>:<sha>`, logging in with the same
   token.
4. Deploys the image by digest — previews carry a rolling TTL.
5. Reports `success` (preview URL lands on the PR) or `failure`.

**`mode: static`:**

1. Exchanges the OIDC token as above.
2. Plans the deployment and, unless `baseUrl` is set, resolves the planned
   deploy host (`location.get`/`project.get`) and injects it as `HUGO_BASEURL`
   so SEO artifacts carry the correct host.
3. Detects the framework, installs the toolchain (Hugo extended pinned from
   `.tool-versions`, or Node), and runs the build into `outputDir`.
4. Opens an upload session, uploads each file as a content-addressed blob
   (skipping ones already present), assembles a manifest sorted by path with
   `environment`/`spa`/`notFound`, and commits it as a release
   (`release-sha = sha256(manifest)`).
5. Deploys with `type: Static` and `site: site://…@<release-sha>` — no image,
   no port. Previews carry a rolling TTL.
6. Reports `success`/`failure` (the PR comment shows `Site: <release-sha>`).

Deploying a pre-built image with service-account secrets instead? Use
[deploys-app/deploys-action](https://github.com/deploys-app/deploys-action).

## License

MIT

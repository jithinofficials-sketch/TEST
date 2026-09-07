# Secure GitHub Actions Deployment Design

## Goal

Remove secret-bearing environment files from production and staging Docker builds. Deploy both environments through GitHub Actions, with DigitalOcean App Platform injecting private runtime configuration from its encrypted environment-variable store.

This resolves GitHub issues #182, #259, #260, and #261 through one deployment-path change.

## Scope

- Add a production GitHub Actions deployment workflow that mirrors staging.
- Remove local environment-file deployment tooling for both environments.
- Prevent local environment and credential files from entering Docker build contexts.
- Keep private secrets out of Docker image layers and final image filesystems.
- Pin every third-party GitHub Action to an immutable, reviewed commit SHA.
- Document configuration ownership and an image smoke check.
- Add an automated repository-level regression check for the security invariants.

The DigitalOcean apps already contain their respective encrypted runtime variables. This change does not create, update, or export those variables.

## Deployment Architecture

GitHub Actions is the only release path for staging and production.

- Pushes to `staging` trigger the staging workflow; the workflow also supports manual dispatch.
- Pushes to the production release branch trigger the production workflow; the workflow also supports manual dispatch.
- Each workflow checks out the repository, installs root and widget dependencies with Yarn, builds widget assets, builds and pushes a Linux AMD64 Docker image, then triggers the corresponding DigitalOcean App Platform deployment.
- The workflow never writes `.env`, copies `env.staging`, or copies `env.production`.
- DigitalOcean App Platform injects all server-side private configuration only when the deployed container runs.

Each environment has independent GitHub repository variable and secret names so staging values cannot be selected by a production workflow and vice versa. The workflow's DigitalOcean access token is limited to the permissions required to authenticate to the registry and request deployments.

## Configuration Boundaries

### Docker Build Context

`.dockerignore` excludes all secret-bearing local files:

- `.env` and `.env.*`
- `env.production` and `env.staging`
- private keys, certificates, SSH identity files, service-account JSON files, and `credentials.json`

Checked-in examples remain available to developers by explicitly allowing `.env.example`, `env.production.example`, and `env.staging.example` as applicable.

### Build-Time Values

Next.js compiles selected values into browser assets. Workflows pass only values intentionally public or otherwise required during the build, using explicit Docker build arguments. These include the Shopify API key, app URL, client Sentry DSN, PostHog key, and Pro coupon configuration.

The Dockerfile declares build arguments only for this allowlist. It supplies placeholder values only for private variables needed to satisfy code evaluated while building. Private values are not Docker build arguments, environment variables, copied files, or BuildKit build secrets in this design.

### Runtime Values

DigitalOcean App Platform stores and injects private environment variables separately for staging and production. Private runtime values include `SHOPIFY_API_SECRET`, `MONGO_URI`, `ENCRYPTION_STRING`, `BREVO_API_KEY`, cloud-provider credentials, and other provider credentials.

Runtime configuration is maintained in the relevant DigitalOcean app, not GitHub workflow files or Docker images. Secret rotation must happen in DigitalOcean and affected historical registry images must be restricted or removed according to the existing incident process.

## Docker Image Changes

The production image copies only application artifacts required to run Next.js: built output, package metadata, Prisma schema, public assets, dependencies, and static assets. It must not copy `.env` or any environment-specific file from the build stage.

The Dockerfile continues to use a multi-stage build. A secret-free build context guarantees no local environment files are present in either stage; omitting the final `.env` copy independently protects the final image.

## Workflow Changes

Both deployment workflows have the same secure structure, differing only by trigger, environment-specific configuration names, image target, and DigitalOcean App ID.

Every external GitHub Action is pinned to a full commit SHA. Actions to pin include `actions/checkout`, `actions/setup-node`, `digitalocean/action-doctl`, and `docker/setup-buildx-action`. Dependabot or Renovate should manage reviewed SHA update pull requests if it is available for this repository.

The workflows use separate variables and secrets such as:

- `STAGING_DEPLOY_DOCKER_IMAGE` and `PRODUCTION_DEPLOY_DOCKER_IMAGE`
- `STAGING_DEPLOY_DO_APP_ID` and `PRODUCTION_DEPLOY_DO_APP_ID`
- `STAGING_DIGITALOCEAN_ACCESS_TOKEN` and `PRODUCTION_DIGITALOCEAN_ACCESS_TOKEN`
- staging and production equivalents of public build values

No private DigitalOcean runtime secret is passed to `docker buildx build`.

## Removed Legacy Tooling

Delete `deploy.js`, its `deploy/` support files, platform-specific local deployment scripts, and their package scripts. These paths copy `env.staging` or `env.production` to `.env` and are no longer supported after GitHub Actions owns both releases.

Local development remains supported through developers' ignored `.env` files and checked-in example files. It is not a deployment path.

## Documentation And Verification

The deployment section in `README.md` documents:

- GitHub Actions as the staging and production deployment path.
- DigitalOcean App Platform as the location for private runtime configuration.
- The boundary between allowed public build-time variables and private runtime secrets.
- Required environment-specific GitHub variable and secret names.
- A smoke check that builds an image then searches `/app` for `.env`, `.env.*`, `env.production`, and `env.staging`; successful output has no matching paths.

An operations test validates the Dockerfile does not copy `.env`, `.dockerignore` blocks required env files while preserving examples, and deployment workflows do not write `.env` or use mutable third-party action tags.

## Error Handling And Rollback

If a deployment fails, GitHub Actions stops before or during DigitalOcean deployment and exposes the failing step in workflow logs. Runtime secret changes remain outside the image, so rotating a secret requires updating the applicable DigitalOcean runtime variable and triggering a new deployment.

Rollback deploys a previously approved secret-free image or re-runs the appropriate workflow from a known-good commit. It does not restore the retired local env-file deployment scripts.

## Acceptance Criteria

- Final staging and production images contain no `.env`, `.env.*`, `env.production`, or `env.staging` paths.
- Docker build contexts exclude local environment files and common private credential files while preserving checked-in examples.
- Staging and production deploy through SHA-pinned GitHub Actions workflows without creating a secret-bearing env file.
- Private values are injected by the corresponding DigitalOcean App Platform application at runtime only.
- Local env-file deployment scripts and package commands are removed.
- Documentation identifies configuration locations and provides an image smoke check.
- Automated checks prevent reintroduction of the insecure image or workflow patterns.

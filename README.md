# CreatorPlaza-Spree

A [Spree Commerce](https://spreecommerce.org) project.

## Getting Started

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed and running

### Start the Spree API

```bash
cd CreatorPlaza-Spree
pnpm spree dev
```

The first run completes setup automatically — it pulls the latest Spree image,
seeds the database, and configures API keys.

Wait for the services to be healthy, then open:

- **Admin Dashboard:** http://localhost:5173 — started automatically by `spree dev`
  - The first run opens a setup link where you create the admin account
- **Seller Panel (marketplace):** http://localhost:5174 — run it with `cd apps/seller-dashboard && pnpm run dev`
- **Store API:** http://localhost:3000/api/v3/store

### Start the storefront

Dependencies are already installed during setup — just start it:

```bash
cd apps/storefront
pnpm run dev
```

Open http://localhost:3001

### The Admin Dashboard

`apps/dashboard/` is your admin — a customizable React SPA (plugins, your
own pages, table tweaks) with live reload. `spree dev` starts it
automatically alongside the API; to run it on its own:

```bash
cd apps/dashboard
pnpm run dev
```

Open http://localhost:5173 and sign in with the admin email and
password you chose during the first run.

When you deploy, the production image builds your dashboard and serves it at
`/dashboard` on the same origin as the API (`pnpm spree build --production`).

To learn how to add pages, tweak tables, or build plugins, see the
[Admin Dashboard docs](https://spreecommerce.org/docs/developer/dashboard/overview).

## Customizing the Spree API

The `server/` directory is the Spree API — a full Rails application serving the Store and Admin APIs (plus background jobs and transactional emails) that your storefront and dashboard talk to. By default, the project runs it from a prebuilt Docker image. To switch to building from your local copy:

```bash
pnpm spree eject
```

This rebuilds the Docker image from `server/` and restarts services. You can then:

- **Customize the API** by editing the files in `server/`
- **Add gems** to `server/Gemfile`
- **Add new resources** with `spree generate model <name> <attributes>`

## Container images and Kubernetes

The production API image is built from `server/Dockerfile` with the repository
root as its build context. Separate production images are also provided for the
storefront, admin dashboard, and seller dashboard. The GitHub Actions workflow
publishes all four images to GHCR; the Forgejo workflow uses the registry
configured in the repository.

GitHub image names are `ghcr.io/<owner>/<repository>/{api,storefront,dashboard,seller-dashboard}`.
The workflows publish commit-SHA tags and update `latest` from the default
branch.

Before running either workflow, configure these repository variables:

- `SPREE_API_URL`: public API origin reachable by the storefront image build,
  for example `https://store.example.com`
- `SPREE_PUBLISHABLE_KEY`: the Spree store's publishable API key
- `NEXT_PUBLIC_SITE_URL`: public storefront origin

Forgejo additionally needs `CONTAINER_REGISTRY` (registry hostname) and
`CONTAINER_NAMESPACE` (registry owner/project path) variables, plus
`CONTAINER_USERNAME` and `CONTAINER_PASSWORD` repository secrets.

### Kubernetes

`k8s/creatorplaza.yaml` deploys all four services, an NGINX ingress, and
placeholder Kubernetes Secrets in one file. Before applying it:

1. Rotate the Neon password that was shared during setup, then replace the
   `DATABASE_URL` placeholder with the rotated Neon URL. Keep `sslmode=require`
   and `channel_binding=require`; URL-encode any reserved characters in the
   password.
2. Generate `SECRET_KEY_BASE` with `bin/rails secret` and the three Active
   Record encryption keys with `bin/rails db:encryption:init`; use stable
   production values and do not rotate encryption keys after storing encrypted
   data.
3. Set the Spree publishable key, registry credentials, TLS certificate/key,
   and replace the sample container image paths with the published image paths.
    For Forgejo, also change the registry host in `.dockerconfigjson`.
4. Replace `store.example.com` with your domain everywhere in the manifest and
   set the three storefront repository variables above to the corresponding
   public URLs. Ensure an NGINX Ingress Controller is installed.
5. Apply with `kubectl apply -f k8s/creatorplaza.yaml`.

The application Secret is intentionally a template: no live database password
or generated key is committed. Kubernetes Secrets are base64-encoded rather
than encrypted by default; enable etcd encryption and restrict RBAC, or use an
external secret manager for production.

Uploads use local Active Storage unless both AWS access-key fields are set.
To use S3, create a bucket and IAM credentials, set `AWS_ACCESS_KEY_ID` and
`AWS_SECRET_ACCESS_KEY` in the `creatorplaza-secrets` Secret, and configure
`AWS_BUCKET` and `AWS_REGION` on the API Deployment. The API then automatically
selects the existing S3 Active Storage service; no persistent volume is needed
for uploads. Until S3 is configured, local uploads are ephemeral and can be
lost when the API pod is replaced. Do not put AWS credentials in the image or
CI build arguments.

## Spree CLI

This project uses [`@spree/cli`](https://spreecommerce.org/docs/developer/cli/quickstart) to manage the Spree API.

### Services

| Command | Description |
|---------|-------------|
| `spree dev` | Run the app in the foreground — streams logs, Ctrl+C stops it. First run completes setup automatically |
| `spree stop` | Stop the API services |
| `spree upgrade` | Upgrade Spree — the server, database and `@spree/*` packages |
| `spree eject` | Switch from prebuilt image to building from `server/` |
| `spree build --production` | Build the production image — includes `apps/dashboard` when present |
| `spree logs` | View web server logs |
| `spree logs worker` | View background jobs logs |
| `spree console` | Open Rails console |

### Data

| Command | Description |
|---------|-------------|
| `spree migrate` | Install pending Spree migrations from gems, then run them or just run your own migrations |
| `spree seed` | Seed the database |
| `spree sample-data` | Load sample products, categories, images |

### Users & API Keys

| Command | Description |
|---------|-------------|
| `spree user create` | Create an admin user |
| `spree api-key create` | Create a publishable or secret API key |
| `spree api-key list` | List all API keys |
| `spree api-key revoke <id>` | Revoke an API key (ID from `api-key list`) |

### Generators

| Command | Description |
|---------|-------------|
| `spree generate model Brand name:string slug:string:uniq` | Generate a new database model |
| `spree generate api_resource Brand name:string slug:string:uniq` | Generate a new Spree API resource |
| `spree generate subscriber OmsOrderSync order.placed` | Generate a new event subscriber |
| `spree generate migration AddPositionToSpreeBrands position:integer` | Generate a new database migration |

### Admin API

Project setup mints a read-only secret key into `.spree/credentials.json` (gitignored), so the Admin API client works out of the box. If you skipped the setup step, `spree api` mints the key on first use instead:

```bash
pnpm spree api get products
pnpm spree api get "orders?q[status_eq]=complete"
pnpm spree api endpoints          # list endpoints + required scopes
pnpm spree api status             # show resolved credentials + server reachability
```

The pre-configured key is read-only. To write, create a scoped secret key and pass it via `SPREE_API_KEY`:

```bash
pnpm spree api-key create --scopes write_products
SPREE_API_KEY=sk_... pnpm spree api post products --data '{"name":"New product","prices":[{"currency":"USD","amount":"29.99"}]}'
```

| Command | Description |
|---------|-------------|
| `spree api get/post/patch/delete <path>` | Call the Admin API directly |
| `spree api endpoints` | List Admin API endpoints with required scopes |
| `spree auth login --profile <name>` | Save named credentials for a remote store |

> **Running `spree` directly.** The commands above use `pnpm` because `@spree/cli` is a local project dependency. You can also run any of the package scripts (e.g. `pnpm run api -- get products`), or install the CLI globally for a bare `spree` command:
>
> ```bash
> pnpm add -g @spree/cli
> spree api get products
> ```

## Learn More

- [Spree Documentation](https://spreecommerce.org/docs)
- [Spree Discord](https://discord.spreecommerce.org)
- [Store API Reference](https://spreecommerce.org/docs/api-reference/store-api/introduction)
- [Admin API Reference](https://spreecommerce.org/docs/api-reference/admin-api/introduction)
- [CLI Reference](https://spreecommerce.org/docs/developer/cli/quickstart)
- [Spree GitHub](https://github.com/spree/spree)

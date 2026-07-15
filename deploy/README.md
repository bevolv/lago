# Lago Deploy

This repository contains the necessary files to deploy the Lago project.

## Docker Compose Local

To deploy the project locally, you need to have Docker and Docker Compose installed on your machine.
This configuration can be used for small production usages but it's not recommended for large scale deployments.

### Get Started

1. Get the docker compose file

```bash
curl -o docker-compose.yml https://raw.githubusercontent.com/getlago/lago/main/deploy/docker-compose.local.yml
```

2. Run the following command to start the project:

```bash
docker compose up --profile all

# If you want to run it in the background
docker compose up -d --profile all
```

## Docker Compose Light

This configuration provide Traefik as a reverse proxy to ease your deployment.
It supports SSL with Let's Encrypt. :warning: You need a valid domain (with at least one A or AAA record)!

1. Get the docker compose file

```bash
curl -o docker-compose.yml https://raw.githubusercontent.com/getlago/lago/main/deploy/docker-compose.light.yml
curl -o .env https://raw.githubusercontent.com/getlago/lago/main/deploy/.env.light.example
```

2. Replace the .env values with yours

```bash
LAGO_DOMAIN=domain.tld
LAGO_ACME_EMAIL=email@domain.tld
```

3. Run the following command to start the project

```bash
docker compose up --profile all

# If you want to run it in the background
docker compose up -d --profile all
```

## Docker Compose Production (Recommended for EC2)

This configuration provide Traefik as a reverse proxy to ease your deployment.
It supports SSL wth Let's Encrypt. :warning: You need a valid domain (with at least one A or AAA record)!
It also adds multiple services that will help your to handle more load.
Portainer is also packed to help you scale services and manage your Lago stack.

```bash
curl -o docker-compose.yml https://raw.githubusercontent.com/getlago/lago/main/deploy/docker-compose.production.yml
curl -o .env https://raw.githubusercontent.com/getlago/lago/main/deploy/.env.production.example
curl -o postgresql.conf https://raw.githubusercontent.com/getlago/lago/main/deploy/postgresql.conf
```

The `db` service runs on `getlago/postgres-partman`, which ships the `pg_partman`
extension Lago's schema requires (partitioning for the `enriched_events` table) —
plain `postgres:15-alpine` is missing it and `db:migrate` fails with `extension
"pg_partman" is not available`. `postgresql.conf` configures `pg_partman_bgw` for
scheduled partition maintenance and must sit next to `docker-compose.yml`.

2. Replace the .env values with yours

```bash
LAGO_DOMAIN=domain.tld
LAGO_ACME_EMAIL=email@domain.tld
SECRET_KEY_BASE=<strong-random-secret>
LAGO_ENCRYPTION_PRIMARY_KEY=<strong-random-secret>
LAGO_ENCRYPTION_DETERMINISTIC_KEY=<strong-random-secret>
LAGO_ENCRYPTION_KEY_DERIVATION_SALT=<strong-random-secret>
POSTGRES_PASSWORD=<strong-random-secret>
REDIS_PASSWORD=<strong-random-secret>
```

3. Run the following command to start the project

```bash
docker compose up --profile all

# If you want to run it in the background
docker compose up -d --profile all

# If you use external PostgreSQL + external Redis
docker compose up -d --profile all-no-db
```

4. Optional: start Portainer only when needed

```bash
docker compose up -d --profile all --profile portainer
```

### Production notes for self-hosted EC2

- Use `deploy/docker-compose.production.yml` for production. Do not use the root `docker-compose.yml` for internet-facing production.
- `deploy/deploy.sh` is a convenience bootstrap script. For reproducible production deployments, run Docker Compose directly with pinned files and explicit profiles.
- Ensure your DNS `A/AAAA` record points to your EC2 public IP before starting the stack.
- Keep `.env` out of Git and store secrets in AWS SSM Parameter Store / Secrets Manager.
- Restrict EC2 security groups to required ports only (`443`; optionally `80` for redirects/challenges).
- Use external managed PostgreSQL/Redis (RDS/ElastiCache) for higher reliability.
- Configure backups, monitoring, and log aggregation for production operations.

### Automated Deployment via GitHub Actions

Pushing to the `deploy/ec2` branch triggers `.github/workflows/deploy-ec2.yml`, which
SSHes into a single EC2 instance, uploads `docker-compose.production.yml`, generates
`.env` from GitHub Actions secrets/variables, and runs `docker compose pull && up -d`.
It also deploys a Dozzle log viewer (`amir20/dozzle`) behind Traefik, protected by
basic auth.

Configure a `production` GitHub Environment (Settings → Environments) with the
following secrets and variables.

**Secrets**

| Secret | Required | Purpose |
|---|---|---|
| `EC2_SSH_PRIVATE_KEY` | Yes | SSH private key for the instance |
| `EC2_SSH_HOST` | Yes | Instance IP/hostname |
| `EC2_SSH_USER` | Yes | SSH login user |
| `POSTGRES_PASSWORD` | Yes | Bundled Postgres password (`openssl rand -hex 32`) |
| `SECRET_KEY_BASE` | Yes | Rails secret key base (`openssl rand -hex 64`) |
| `LAGO_ENCRYPTION_PRIMARY_KEY` | Yes | Encryption key (`openssl rand -hex 32`) |
| `LAGO_ENCRYPTION_DETERMINISTIC_KEY` | Yes | Encryption key |
| `LAGO_ENCRYPTION_KEY_DERIVATION_SALT` | Yes | Encryption key |
| `REDIS_PASSWORD` | Optional | When set, enables `--requirepass` on the bundled Redis container and its healthcheck (`openssl rand -hex 32`) |
| `LAGO_RSA_PRIVATE_KEY` | Optional | Only needed to override the auto-generated key |
| `LAGO_AWS_S3_ACCESS_KEY_ID` | Optional | S3 storage |
| `LAGO_AWS_S3_SECRET_ACCESS_KEY` | Optional | S3 storage |
| `LAGO_SMTP_USERNAME` | Optional | SMTP email |
| `LAGO_SMTP_PASSWORD` | Optional | SMTP email |
| `PORTAINER_PASSWORD` | Optional | Only used if `EC2_ENABLE_PORTAINER=true` |
| `DOZZLE_PASSWORD` | Optional | Falls back to `changeme` — set before exposing Dozzle publicly |

**Variables**

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `LAGO_DOMAIN` | Optional | `lago.bevolv.co` | Public domain, A-recorded to the instance |
| `LAGO_ACME_EMAIL` | Yes | — | Let's Encrypt registration email |
| `EC2_DEPLOY_PATH` | Optional | `/opt/lago` | Remote deploy directory |
| `EC2_DEPLOY_PROFILE` | Optional | `all` | Compose `--profile` |
| `EC2_ENABLE_PORTAINER` | Optional | `false` | Adds `--profile portainer` |
| `EC2_KNOWN_HOSTS` | Optional | falls back to runtime `ssh-keyscan` | Pinned host key, closes the TOFU gap on first connect |
| `LAGO_USE_AWS_S3` | Optional | `false` | S3 toggle |
| `LAGO_AWS_S3_REGION` / `LAGO_AWS_S3_BUCKET` / `LAGO_AWS_S3_ENDPOINT` | Optional | empty | S3 config |
| `LAGO_FROM_EMAIL` / `LAGO_SMTP_ADDRESS` / `LAGO_SMTP_PORT` | Optional | empty / `587` | SMTP config |
| `PORTAINER_USER` | Optional | empty | Only relevant if Portainer enabled |
| `DOZZLE_USER` | Optional | `admin` | Dozzle basic-auth username — also gates the Traefik dashboard, which reuses the same htpasswd file |
| `DOZZLE_HOSTNAME` | Optional | `dozzle.lago` | Prefix only, always suffixed with the hardcoded `.bevolv.co` in the compose file |
| `TRAEFIK_HOSTNAME` | Optional | `traefik.lago` | Prefix only, suffixed with `.bevolv.co`. Serves the Traefik dashboard/API, protected by the same basic auth as Dozzle |

Not wired: GCS storage, Google SSO. `POSTGRES_USER/DB/HOST/PORT` and
`REDIS_HOST/PORT` stay at compose defaults since the workflow always bundles
Postgres/Redis on the instance (`--profile all`).

`POSTGRES_PASSWORD` and `REDIS_PASSWORD` only take effect on first container init —
Postgres and Redis both persist to a named Docker volume (`lago_postgres_data`,
`lago_redis_data`), so once that volume exists, changing the GitHub secret alone
does not rotate the running credential. To rotate either one later, update the
secret and also change the password inside the running container (e.g. `ALTER USER
lago WITH PASSWORD '...'` for Postgres, `redis-cli CONFIG SET requirepass ...` for
Redis) so the two stay in sync — otherwise the app's connection string and the
actual stored credential diverge and the app fails to connect.

**Prerequisites, before the first push to `deploy/ec2`:**
- DNS: `LAGO_DOMAIN` (or the `lago.bevolv.co` default) A-records to the instance's
  public IP. `<DOZZLE_HOSTNAME>.bevolv.co` (default `dozzle.lago.bevolv.co`) and
  `<TRAEFIK_HOSTNAME>.bevolv.co` (default `traefik.lago.bevolv.co`) A-record to the
  same IP — Traefik/Let's Encrypt need to resolve and challenge all three hosts.
- Security Group: `22` (restricted to an admin CIDR), `80`, `443` open. `5432` and
  `6379` not open to `0.0.0.0/0`.
- Instance: Docker Engine + Compose v2 plugin installed, SSH user has passwordless
  sudo (default on standard EC2 AMIs) and is in the `docker` group.
- GitHub: `production` Environment created, all secrets/variables above populated —
  in particular `DOZZLE_PASSWORD`, which otherwise defaults to `changeme`.

After the first run: confirm secret values render as `***` in the Actions log,
confirm the health-check and smoke-test steps pass, browse to the domain and confirm
a trusted Let's Encrypt certificate (not a staging/self-signed warning), confirm
Portainer isn't running unless intentionally enabled, and confirm both the Dozzle
and Traefik dashboard hostnames prompt for basic-auth credentials (same
`DOZZLE_USER`/`DOZZLE_PASSWORD` for both) rather than serving content straight away.
A common `EC2_SSH_USER` value (e.g. `ubuntu`) gets masked wherever it appears in the
Actions log, including unrelated text — expected, harmless.

There is no automatic rollback: the deploy runs `docker compose pull && up -d
--wait` against a stateful instance with an in-place database, and the `migrate`
service always runs first. A failure fails the job loudly; recover by SSHing in,
checking `docker compose logs`, and fixing forward.


## Configuration

### Profiles

The docker compose file contains multiple profiles to enable or disable some services.
Here are the available profiles:
- `all`: Enable all services
- `all-no-pg`: Disable the PostgreSQL service
- `all-no-redis`: Disable the Redis service
- `all-no-keys`: Disable the RSA keys generation service

This allow you to start only the service you want to use, please see the following sections for more information.

```bash
# Start all services
docker compose up --profile all

# Start without PostgreSQL
docker compose up --profile all-no-pg

# Start without Redis
docker compose up --profile all-no-redis

# Start without PostgreSQL and Redis
docker compose up --profile all-no-db

# Start without RSA keys generation
docker compose up --profile all-no-keys

# Start without PostgreSQL, Redis and RSA keys generation
docker compose up
```

### PostgreSQL

It is possible to disable the usage of the PostgreSQL database to use an external database instance.

1. Set those environment variables:

- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `POSTGRES_DB`
- `POSTGRES_HOST`
- `POSTGRES_PORT`
- `POSTGRES_SCHEMA` optional

2. Run the following command to start the project without PostgreSQL:

```bash
docker compose up --profile all-no-pg
```

### Redis

It is possible to disable the usage of the Redis database to use an external Redis instance.

1. Set those environment variables:

- `REDIS_HOST`
- `REDIS_PORT`
- `REDIS_PASSWORD` optional

2. Run the following command to start the project without Redis:

```bash
docker compose up --profile all-no-redis
```

### RSA Keys

Those docker compose file generates an RSA Keys pair for the JWT tokens generation.
You can find the keys in the `lago_rsa_data` volume or in the `/app/config/keys` directory in the backends containers.
If you do not want to use those keys:
- Remove the `lago_rsa_data` volume
- Generate your own key using `openssl genrsa 2048 | openssl base64 -A`
- Export this generated key to the `LAGO_RSA_PRIVATE_KEY` env var.
- Run the following command to start the project without the RSA keys generation:

```bash
docker compose up --profile all-no-keys
```

*All BE Services use the same RSA key, they will exit immediately if no key is provided.*

## Monitoring

For production deployments, we recommend setting up monitoring for Sidekiq workers. See the [Monitoring documentation](../docs/monitoring.md) for:
- Prometheus metrics endpoints and available metrics
- Recommended alerting rules
- Grafana dashboard recommendations

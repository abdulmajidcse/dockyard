# dockyard

One shared Docker stack for local development: **Postgres, Redis, SeaweedFS
(S3), Mailpit and Caddy**. Every project gets its own isolated database, Redis
slots and bucket inside it, instead of running its own copy of each service,
and Caddy gives each app a `http://<name>.localhost` address.

Needs only Docker. The clients (`psql`, `redis-cli`, `weed shell`) run inside
the containers.

| Service | Host port | Hostname on the network | Each project gets |
|---|---|---|---|
| PostgreSQL 17 | 5432 | `postgres:5432` | a database + role |
| Redis 7 | 6379 | `redis:6379` | two database numbers |
| SeaweedFS (S3) | 8333, admin UI 23646 | `seaweedfs:8333` | a bucket + scoped key |
| Mailpit | 1025 smtp, 8025 web | `mailpit:1025` | shared inbox |
| Caddy | 80 | — | `http://<name>.localhost` routes |

Host ports and image versions are set in `.env`.

## Start

```sh
cp .env.example .env                          # optional; every value has a default
cp caddy/Caddyfile.example caddy/Caddyfile    # optional; see "Local domains"
docker compose up -d --wait                   # returns once every service is healthy
```

```sh
docker compose ps             # status
docker compose logs -f redis  # logs for one service
docker compose down           # stop (data is kept)
```

## Add a project

**Postgres** — run the script and answer the prompts:

```sh
bin/create-db                 # asks for database, user, password
bin/create-db my-app          # or pass the name
```

- Press Enter to accept a default: the user is named after the database, and
  a blank password is generated for you.
- You see a summary before anything is created, and can **edit** or **quit**.
- It prints ready-to-paste `.env` lines for each
  [connection style](#connecting-an-application). Copy the password: it is
  not stored.
- It refuses to touch a database or user that already exists.

**Redis** — nothing to create. Pick two unused database numbers (0–15), for
example `4` and `5`, and note which project uses them. Redis cannot tell you.

**SeaweedFS** — create a bucket and a key limited to it:

```sh
docker compose exec -T seaweedfs weed shell -master=127.0.0.1:9333 <<'SH'
s3.bucket.create -name my-app
s3.configure -user my-app -access_key my-app -secret_key generate-a-real-one -buckets my-app -actions Read,Write,List,Tagging -apply
SH
```

- The key can read, write and list `my-app` only. It can't see other buckets
  or create new ones.
- Running `s3.configure` again with the same `-user` adds to that user.
  `s3.configure -user my-app -delete -apply` removes the user; with
  `-buckets`/`-actions` as well, `-delete` removes only those permissions.
- To serve a bucket's files to anyone without signing (public images, for
  example), grant reads to the special `anonymous` user:
  `s3.configure -user anonymous -buckets my-app -actions Read -apply`.

**SeaweedFS web interface**

| | |
|---|---|
| URL | http://127.0.0.1:23646 (`SEAWEEDFS_ADMIN_PORT`) |
| Username | `dockyard` (`SEAWEEDFS_ROOT_USER`) |
| Password | `dockyard-root` (`SEAWEEDFS_ROOT_PASSWORD`) |

- Use it to browse buckets and files, and to manage S3 users and keys.
- Port `8333` is the S3 API that apps connect to, not a web page.
- The same pair is the S3 admin key. Never put it in a project's `.env`.

**Mailpit** — nothing to create. Give the project its own from-address, such
as `my-app@dockyard.test`, so you can find its mail.

## Connecting an application

Pick one of three styles:

| Where the app runs | Host | Port |
|---|---|---|
| Container on the `dockyard` network (default) | service name, e.g. `postgres` | standard, e.g. `5432` |
| Container **not** on the network | `host.docker.internal` | host port from `.env` |
| Directly on your machine | `127.0.0.1` | host port from `.env` |

The second style works on Docker Desktop (macOS/Windows) only; see
[docs/connecting-without-the-network.md](docs/connecting-without-the-network.md).

**Joining the network** — in your app's `compose.yml`:

```yaml
networks:
  dockyard:
    external: true

services:
  app:
    build: .
    networks: [dockyard, default]
    env_file: .env
```

Only attach containers that use these services. Anything on `dockyard` can
reach anything else on it, including other projects' containers.

Start dockyard first. Otherwise the app fails with
`network dockyard declared as external, but could not be found`.

**Example `.env`** (Laravel key names; use your framework's):

```ini
DB_CONNECTION=pgsql
DB_HOST=postgres
DB_PORT=5432
DB_DATABASE=my_app
DB_USERNAME=my_app
DB_PASSWORD=from-bin-create-db

REDIS_HOST=redis
REDIS_PORT=6379
REDIS_DB=4
REDIS_CACHE_DB=5

AWS_ENDPOINT=http://seaweedfs:8333        # used by the app
AWS_URL=http://127.0.0.1:8333/my-app      # used by the browser
AWS_ACCESS_KEY_ID=my-app
AWS_SECRET_ACCESS_KEY=generate-a-real-one
AWS_BUCKET=my-app
AWS_USE_PATH_STYLE_ENDPOINT=true

MAIL_MAILER=smtp
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_FROM_ADDRESS=my-app@dockyard.test
```

`AWS_ENDPOINT` and `AWS_URL` differ because the browser runs on your machine,
where the hostname `seaweedfs` does not exist.

### From the host instead

Tools running directly on your machine (a GUI client, test runner or dev
server) use `127.0.0.1` and the host ports. They reach the same data.

## Local domains (Caddy)

Caddy routes `http://<name>.localhost` to your apps on port 80. Browsers send
every `*.localhost` name to `127.0.0.1` by themselves, so there is nothing to
add to `/etc/hosts`.

Your routes live in `caddy/Caddyfile`, which is gitignored. Start from the
example, then reload after every edit (no restart needed):

```sh
cp caddy/Caddyfile.example caddy/Caddyfile
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

Until `caddy/Caddyfile` exists, Caddy serves `Caddyfile.example` as-is.

An upstream is one of:

| The app | Upstream |
|---|---|
| Publishes a port, or runs directly on your machine | `host.docker.internal:<host port>` |
| A container on the `dockyard` network | `<service name>:<container port>` |

```caddyfile
http://my-app.localhost {
	reverse_proxy host.docker.internal:8000
}

# Multi-tenant: the apex and every subdomain.
http://my-shop.localhost, http://*.my-shop.localhost {
	reverse_proxy host.docker.internal:3000
}
```

- An app container that calls another app by its `.localhost` name needs
  `extra_hosts: ["my-app.localhost:host-gateway"]` (Docker Desktop, Colima).
  Inside a container, `localhost` is the container itself.
- On Linux, `host.docker.internal` can only reach ports an app publishes on
  all interfaces, not ones bound to `127.0.0.1`. Join the `dockyard` network
  and use the service name instead.
- Keep Caddy's admin API enabled (don't write `admin off`): the healthcheck
  and `caddy reload` use it. It is never published outside the container.

## Isolation

- **Postgres:** each project's role can only open its own database.
- **Redis:** separate database numbers, so one project's `FLUSHDB` doesn't
  wipe another's. 16 databases ÷ 2 = room for 8 projects.
- **SeaweedFS:** each key can only see its own bucket. Projects never get the
  admin key.
- **Mailpit:** not isolated. One shared inbox.

## Everyday commands

```sh
docker compose exec postgres psql -U postgres     # Postgres shell
docker compose exec redis redis-cli -n 4          # Redis database 4
open http://127.0.0.1:8025                        # mail inbox
open http://127.0.0.1:23646                       # SeaweedFS admin UI
docker compose exec seaweedfs weed shell -master=127.0.0.1:9333   # S3 admin shell
```

SeaweedFS login: see [SeaweedFS web interface](#add-a-project).

**Back up and restore a database:**

```sh
docker compose exec -T postgres pg_dump -U postgres -Fc my_app > my_app.dump
docker compose exec -T postgres pg_restore -U postgres -d my_app --clean --if-exists < my_app.dump
```

**Find one project's mail:**

```sh
curl -s 'http://127.0.0.1:8025/api/v1/search?query=from:my-app@dockyard.test'
```

## Configuration

- Every setting lives in `.env`. `.env.example` explains each one.
- **The Postgres password is read once.** Changing `POSTGRES_PASSWORD` after
  first start has no effect until the volume is recreated. `SEAWEEDFS_ROOT_*`
  is read on every start.
- **Keep `127.0.0.1:` on every port.** Without it, the service is reachable
  from your whole network.
- For changes that only apply to your machine, use `compose.override.yml`. It
  is gitignored.

## Reset

Data lives in Docker volumes and survives `docker compose down`. To wipe
**every project's data** (this can't be undone):

```sh
docker compose down -v
docker compose up -d --wait
```

## Not for production

This is a local development tool with convenience passwords. Everything is
bound to `127.0.0.1`.

## Licence

MIT.

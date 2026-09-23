# dockyard

One shared Docker stack for local development: **Postgres, Redis, MinIO and
Mailpit**. Every project gets its own isolated database, Redis slots and
bucket inside it, instead of running its own copy of each service.

Needs only Docker. The clients (`psql`, `redis-cli`, `mc`) run inside the
containers.

| Service | Host port | Hostname on the network | Each project gets |
|---|---|---|---|
| PostgreSQL 17 | 5432 | `postgres:5432` | a database + role |
| Redis 7 | 6379 | `redis:6379` | two database numbers |
| MinIO | 9000, console 9001 | `minio:9000` | a bucket + scoped key |
| Mailpit | 1025 smtp, 8025 web | `mailpit:1025` | shared inbox |

Host ports and image versions are set in `.env`.

## Start

```sh
cp .env.example .env          # optional; every value has a default
docker compose up -d --wait   # returns once every service is healthy
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

**MinIO** — create a bucket and a key limited to it:

```sh
docker compose exec -T minio sh -s <<'SH'
mc alias set local http://127.0.0.1:9000 "$MINIO_ROOT_USER" "$MINIO_ROOT_PASSWORD"
mc mb --ignore-existing local/my-app
cat > /tmp/my-app.json <<'JSON'
{ "Version": "2012-10-17", "Statement": [
  { "Effect": "Allow", "Action": ["s3:*"],
    "Resource": ["arn:aws:s3:::my-app", "arn:aws:s3:::my-app/*"] } ] }
JSON
mc admin user add local my-app generate-a-real-one
mc admin policy create local my-app /tmp/my-app.json
mc admin policy attach local my-app --user my-app
SH
```

**MinIO web interface**

| | |
|---|---|
| URL | http://127.0.0.1:9001 (`MINIO_CONSOLE_PORT`) |
| Username | `dockyard` (`MINIO_ROOT_USER`) |
| Password | `dockyard-root` (`MINIO_ROOT_PASSWORD`) |

- Use it to browse buckets and upload, download or delete files.
- It can't create users or access keys in newer MinIO versions. Use the `mc`
  commands above for that.
- Port `9000` is the storage API that apps connect to, not a web page.
- The login is the root account. Never put it in a project's `.env`.

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

AWS_ENDPOINT=http://minio:9000            # used by the app
AWS_URL=http://127.0.0.1:9000/my-app      # used by the browser
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
where the hostname `minio` does not exist.

### From the host instead

Tools running directly on your machine (a GUI client, test runner or dev
server) use `127.0.0.1` and the host ports. They reach the same data.

## Isolation

- **Postgres:** each project's role can only open its own database.
- **Redis:** separate database numbers, so one project's `FLUSHDB` doesn't
  wipe another's. 16 databases ÷ 2 = room for 8 projects.
- **MinIO:** each key can only see its own bucket. Projects never get the
  root key.
- **Mailpit:** not isolated. One shared inbox.

## Everyday commands

```sh
docker compose exec postgres psql -U postgres     # Postgres shell
docker compose exec redis redis-cli -n 4          # Redis database 4
open http://127.0.0.1:8025                        # mail inbox
open http://127.0.0.1:9001                        # MinIO console
```

MinIO console login: see [MinIO web interface](#add-a-project).

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
- **Passwords are read once.** Changing `POSTGRES_PASSWORD` or `MINIO_ROOT_*`
  after first start has no effect until the volume is recreated.
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

# dockyard

One Docker stack for the backing services every local project needs, instead
of one stack per project.

Most local setups give each project its own `compose.yml` with its own
Postgres, Redis, object store and mail catcher. That is fine for one project.
By the fourth you are running four database servers to hold four mostly-idle
databases, only one container can have port 5432, and the projects have
quietly drifted onto different major versions.

This stack runs one of each service and gives every project an isolated
database, role, Redis slot and bucket inside it.

It is a `compose.yml` and an `.env`, driven with `docker compose` — there is
no wrapper command to install and nothing to build.

Your own applications stay in their own compose files. They join the network
this stack publishes and reach the services by hostname, so a project's
compose file shrinks to the containers that project actually owns.

## What it runs

| Service | Version | Host port | Per project |
|---|---|---|---|
| PostgreSQL | 17 | 5432 | database + role |
| Redis | 7 | 6379 | two database numbers |
| MinIO | latest | 9000, console 9001 | bucket + scoped key |
| Mailpit | latest | 1025, web 8025 | shared inbox, filtered by from-address |

Four services, no optional tier and nothing to enable: if it is in the stack
it runs, and if you do not need it you are paying for one idle container.

Those host ports are for things running directly on your machine. A container
joining the shared network uses hostnames instead — see
[Connecting an application](#connecting-an-application).

Every version in that table is pinned in `.env` rather than in `compose.yml`,
so an upgrade is a one-line change with a visible diff.

Anything that is not one of these four stays in the project that needs it.
A single project needing MySQL or MongoDB adds it to its own compose file;
a second project needing the same thing is the point at which it earns a
place here. That keeps this stack to services shared by everything, rather
than a menu.

## Requirements

Docker. That is all.

Every client you need — `psql`, `redis-cli`, `mc` — already ships inside the
service containers, so you reach them with `docker compose exec` and install
nothing on your machine. The client can never drift out of step with the
server version it is talking to, because they are the same image.

## Start it

```
cp .env.example .env              # optional; every value has a default
docker compose up -d --wait
```

`--wait` blocks until every healthcheck passes, so the command returning
means the services are actually accepting connections — not merely that the
containers started.

```
docker compose ps                 # what is up, on which port, how healthy
docker compose logs -f postgres   # follow one service
docker compose down               # stop; data stays in the volumes
```

## Adding a project

Four steps, one per service. The examples use the slug `my-app`; Postgres
identifiers use `my_app`, since a hyphen would need quoting everywhere.

**Postgres** — a database *and* a role that owns it, so the project cannot
reach any other project's data:

```
docker compose exec -T postgres psql -U postgres -v ON_ERROR_STOP=1 <<'SQL'
CREATE ROLE my_app LOGIN PASSWORD 'generate-a-real-one'
  NOSUPERUSER NOCREATEDB NOCREATEROLE;
CREATE DATABASE my_app OWNER my_app;
REVOKE ALL ON DATABASE my_app FROM PUBLIC;
SQL
```

That `REVOKE` is the line doing the work. Without it every role can connect
to the new database, because `PUBLIC` holds `CONNECT` by default.

`bin/create-db` runs those same statements and prints the `.env` lines to
paste into the project — one complete block for each connection style: an
app container on the shared network, an app container going through
`host.docker.internal` (see
[Connecting without joining the network](docs/connecting-without-the-network.md)),
and something running directly on the host. The latter two use
`POSTGRES_PORT` from `.env`. Run it bare and it asks for the database name, the
user and the password; Enter takes the default, which is a user named after
the database and a generated password:

```
bin/create-db                       # asks for everything
bin/create-db my-app                # asks for user and password
bin/create-db my-app my_app_user    # asks for the password
```

The password shows as you type it and is asked twice. Before anything is
created it shows a summary and asks `Create it? [Y]es, [e]dit, [q]uit`;
edit goes round the questions again with your answers as the defaults, so
Enter keeps whatever was already right. A name that is invalid or already
taken is asked again rather than ending the run, and Ctrl-D at any prompt
cancels without creating anything.

Without a terminal — in a script or CI — nothing is asked and the defaults
are used.

It refuses if the role or database already exists rather than resetting a
password a project is already using. The password is printed once and stored
nowhere, so copy it before you close the terminal.

**Redis** — nothing to create. Pick two unused database numbers, one for the
default connection and one for the cache, and write them into the project's
`.env`. Keep a note of which project holds which pair; the server cannot tell
you.

**MinIO** — a bucket plus a key whose policy names that bucket only:

```
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

The project gets `my-app` and its own secret. It never gets
`MINIO_ROOT_USER`.

**Mailpit** — nothing to create. Point the project at the SMTP port and give
it a distinct from-address so you can pick its mail out of the shared inbox.

That covers the server side. Wiring the application itself up to these
credentials is [Connecting an application](#connecting-an-application) below.

## Projects are isolated from each other

This is the part that makes sharing one instance safe rather than merely
convenient.

- **Postgres** — each project gets its own database *and* its own role,
  created `NOSUPERUSER NOCREATEDB`. Project `alpha` connecting to project
  `beta`'s database is refused by the server with `permission denied for
  database "beta"`.
- **Redis** — each project gets its own pair of database numbers, not a
  shared database with key prefixes. Framework cache drivers commonly
  implement "clear the cache" as `FLUSHDB`; with prefixes, one project
  clearing its cache would wipe everyone's. Redis exposes 16 databases and
  each project claims two, so the stack holds 8 projects — a bounded
  capacity, traded knowingly for a destructive command staying contained.
- **MinIO** — each project gets a bucket and a key whose policy names that
  bucket only. `mc ls` with a project's key lists that one bucket. Projects
  never get the root credentials.
- **Mailpit** — *not isolated*, and the odd one out. There is one inbox and
  every project can read all of it. A distinct `MAIL_FROM_ADDRESS` per
  project lets you filter the inbox down to one project's mail, but that is a
  convention for finding things, not a boundary.

The first three are enforced by the engine, not by whoever ran the setup
being careful. A mistake can fail to create a role; it cannot hand one
project access to another's database, because the server is the thing
refusing.

## Connecting an application

Your app runs in its own compose project. It joins the `dockyard` network and
reaches the services by **hostname on their standard ports** — the host port
mappings are irrelevant from inside, and remapping `POSTGRES_PORT` on the host
does not change anything here.

| Service | Hostname | Port |
|---|---|---|
| PostgreSQL | `postgres` | 5432 |
| Redis | `redis` | 6379 |
| MinIO | `minio` | 9000 |
| Mailpit | `mailpit` | 1025 smtp, 8025 web |

In the application's own `compose.yml`:

```yaml
name: my-app

networks:
  dockyard:
    external: true          # created by this stack; not by your project
  default:                  # your project's own network, created for you

services:
  app:
    build: .
    networks: [dockyard, default]   # needs Postgres, Redis, MinIO, Mailpit
    env_file: .env
    ports:
      - "127.0.0.1:8000:8000"

  queue:
    build: .
    command: ["php", "artisan", "queue:work"]
    networks: [dockyard, default]   # needs Redis and Postgres
    env_file: .env

  vite:
    build: .
    command: ["npm", "run", "dev"]
    networks: [default]             # talks to nothing shared; stays off it
```

Attach a container to `dockyard` only if it actually uses one of the four
services. Everything on that network can resolve and reach everything else on
it, across projects — your `app` container is addressable by another project's
containers exactly as `postgres` is addressable by yours. That is the honest
cost of one shared network, and the way to keep it small is to put only what
needs to be there on it.

Listing `default` alongside it keeps your own services reachable by service
name if you later restrict what sits on the shared network.

And the application's `.env`:

```
DB_CONNECTION=pgsql
DB_HOST=postgres
DB_PORT=5432
DB_DATABASE=my_app
DB_USERNAME=my_app
DB_PASSWORD=generate-a-real-one

REDIS_HOST=redis
REDIS_PORT=6379
REDIS_DB=0
REDIS_CACHE_DB=1

AWS_ENDPOINT=http://minio:9000
AWS_URL=http://127.0.0.1:9000/my-app
AWS_ACCESS_KEY_ID=my-app
AWS_SECRET_ACCESS_KEY=generate-a-real-one
AWS_BUCKET=my-app
AWS_USE_PATH_STYLE_ENDPOINT=true

MAIL_MAILER=smtp
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_FROM_ADDRESS=my-app@dockyard.test
```

`AWS_ENDPOINT` and `AWS_URL` differ on purpose, and it is the one setting
here that catches people out. Your app uploads over the network as
`http://minio:9000`, but any URL it hands to a browser has to work on the
host, where `minio` does not resolve. `AWS_ENDPOINT` is for the server side;
`AWS_URL` is what ends up in an `<img src>`.

These are Laravel's key names. The values are what matter — use whatever your
framework calls them.

### Start order

The network belongs to this stack, so it has to be up first. Start it before
your app:

```
cd /path/to/dockyard && docker compose up -d --wait
cd /path/to/my-app   && docker compose up -d
```

If you forget, the app fails immediately and says so:

```
network dockyard declared as external, but could not be found
```

Nothing starts half-configured — an app that cannot reach the network does not
come up at all.

### From the host instead

If something runs directly on your machine rather than in a container — a GUI
database client, a test runner, a framework dev server — use `127.0.0.1` and
the host ports from the table further up. Both routes reach the same servers
and the same data; they are two addresses for one thing, not two setups.

A container can use that host route too, without joining the network at all —
see [Connecting without joining the network](docs/connecting-without-the-network.md).


## Everyday use

```
docker compose exec postgres psql -U postgres          # superuser shell
docker compose exec redis redis-cli -n 0               # one project's db
docker compose exec minio mc ls local                  # after `mc alias set`
open http://127.0.0.1:8025                             # the mail inbox
open http://127.0.0.1:9001                             # the MinIO console
```

The MinIO console logs in with `MINIO_ROOT_USER` and `MINIO_ROOT_PASSWORD`
from `.env` — `dockyard` / `dockyard-root` unless you changed them. That is
the root account: use it in the console and for `mc alias set`, never in a
project's `.env`, which gets its own scoped key. Recent MinIO releases have
stripped user and policy management out of the console, so create project
keys with the `mc` commands under [Adding a project](#adding-a-project).

Dump and restore a project's database:

```
docker compose exec -T postgres pg_dump -U postgres -Fc my_app > my_app.dump
docker compose exec -T postgres pg_restore -U postgres -d my_app --clean --if-exists < my_app.dump
```

`--if-exists` matters: without it `--clean` fails on every object that is not
already there, which is all of them when you are restoring into a fresh
database.

Find one project's mail in the shared inbox:

```
curl -s 'http://127.0.0.1:8025/api/v1/search?query=from:my-app@dockyard.test'
```

## Configuration

`.env` sets the compose project name, the image version for each engine, the
host ports, the superuser credentials and the mail domain. Every key has a
working default, so an empty `.env` — or none at all — starts the same stack.
`.env.example` documents each one.

Two things are worth knowing before you change it:

- **Credentials are only read once.** Postgres and MinIO apply
  `POSTGRES_PASSWORD` and `MINIO_ROOT_*` when they initialise an empty data
  directory. Changing them later does nothing until you recreate the volume.
- **Keep the `127.0.0.1:` prefix** on any port mapping you edit. Docker
  programs packet filtering directly, so a bare `5432:5432` publishes the
  database to every machine on your network, host firewall or not.

For changes that belong to this machine rather than the project — extra
memory for Postgres, a second engine version on another port — put a
`compose.override.yml` next to `compose.yml`. Docker Compose merges it
automatically, and it is gitignored.

## Data and resetting

Everything lives in named Docker volumes — `postgres-data`, `redis-data`,
`minio-data`, `mailpit-data` — so `docker compose down` stops the stack
without touching data, and `up` brings it back as it was.

Starting over is meant to be routine:

```
docker compose down -v        # destroys every volume, every project's data
docker compose up -d --wait
```

`down -v` is irreversible and takes every project with it, not just the one
you were thinking about.

## Not for production

Everything binds `127.0.0.1`. The superuser passwords are conveniences. This
is a development tool, and the threat model is "do not expose a database to
the coffee shop wifi", not "survive an attacker".

## Licence

MIT.

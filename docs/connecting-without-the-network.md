# Connecting without joining the network

By default, app containers join the `dockyard` network (see the
[README](../README.md#connecting-an-application)). You can skip that and
connect **through your machine** instead, using `host.docker.internal`.

**Works on:** Docker Desktop (macOS, Windows).
**Linux:** use the network instead. See [below](#linux).

## Why use it

- Other projects' containers can't reach your app.
- Your app doesn't need dockyard running before it starts.
- No changes to your app's `compose.yml`.

## Setup

Use `host.docker.internal` and the **host ports** from dockyard's `.env`:

```ini
DB_HOST=host.docker.internal
DB_PORT=5432                # POSTGRES_PORT

REDIS_HOST=host.docker.internal
REDIS_PORT=6379             # REDIS_PORT

AWS_ENDPOINT=http://host.docker.internal:9000
AWS_URL=http://127.0.0.1:9000/my-app

MAIL_HOST=host.docker.internal
MAIL_PORT=1025              # MAILPIT_SMTP_PORT
```

Everything else (database name, user, password, bucket) stays the same.
`bin/create-db` prints this block for you with the right port.

## Things to watch

- **Ports must match `.env`.** If you change `POSTGRES_PORT` in dockyard,
  update your app too. On the network, the port is always `5432`.
- **Check MinIO presigned URLs in a browser.** They are signed with
  `host.docker.internal`, which your browser may not resolve.

## Linux

`host.docker.internal` needs this in your app's service:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

It still won't connect. dockyard's ports only listen on `127.0.0.1`, and on
Linux this hostname points at a different address. Opening the ports wider
would expose the databases to your network, so **join the network instead**.

## Which to use

| Situation | Use |
|---|---|
| Linux | The network |
| macOS / Windows, want isolation from other projects | This page |
| macOS / Windows, no preference | The network (default) |

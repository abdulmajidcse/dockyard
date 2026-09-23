# Connecting without joining the network

The README's [Connecting an application](../README.md#connecting-an-application)
has your app's containers join the `dockyard` network and reach the services
by hostname. That is the default, and it is the route to prefer on Linux.

There is a second route: leave the app's containers off the shared network
entirely and reach the services through the host, on the ports this stack
already publishes to `127.0.0.1`. Nothing in `compose.yml` changes for it.

## Why you might want it

- **Your app stays unreachable from other projects.** Everything on the
  `dockyard` network can reach everything else on it, across projects. An app
  that never joins is not addressable by another project's containers at all.
- **No start-order dependency.** An `external: true` network has to exist
  before the app starts. Going through the host, the app starts whether or
  not this stack is up, and fails at its first connection instead.
- **Nothing to add to the app's compose file** on macOS and Windows.

## Through the host: `host.docker.internal`

Point the app at `host.docker.internal` and the **host** ports — the ones in
the README's [What it runs](../README.md#what-it-runs) table, or whatever you
set `POSTGRES_PORT` and friends to in `.env`.

```
DB_HOST=host.docker.internal
DB_PORT=5432

REDIS_HOST=host.docker.internal
REDIS_PORT=6379

AWS_ENDPOINT=http://host.docker.internal:9000
AWS_URL=http://127.0.0.1:9000/my-app

MAIL_HOST=host.docker.internal
MAIL_PORT=1025
```

Everything else — database names, credentials, Redis database numbers, the
bucket — is unchanged from the README.

### Docker Desktop (macOS, Windows)

Works as written. Docker Desktop resolves `host.docker.internal` inside every
container and forwards the traffic to the host's loopback, where the
`127.0.0.1:` port mappings are listening.

### Linux (Docker Engine)

`host.docker.internal` does not exist by default. Add it to each service that
needs it:

```yaml
services:
  app:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

That alone is not enough, and this is the catch. `host-gateway` resolves to
the bridge IP (typically `172.17.0.1`), not to `127.0.0.1`. The services here
publish to loopback only, so the connection is refused.

Making it work means publishing on the bridge address or on all interfaces,
which breaks rule 1 at the top of `compose.yml` — a bare port mapping exposes
these databases to your whole network regardless of the host firewall. Do not
do that for convenience. **On Linux, join the network instead.**

## Things that change on this route

- **Ports are coupled to `.env`.** On the shared network the app always uses
  the standard ports and host remapping is irrelevant. Through the host, if
  you change `POSTGRES_PORT` here, every app using this route has to change
  with it.
- **MinIO URLs still need two settings.** `AWS_ENDPOINT` is what the app
  connects to; `AWS_URL` is what ends up in a browser. The browser runs on the
  host, where `127.0.0.1` is right and `host.docker.internal` may not resolve.
  Presigned URLs are signed against the endpoint host, so check them in a
  browser before relying on them.
- **Mailpit is the same shared inbox.** Only the address changes.

## Not recommended: sharing a network namespace

`network_mode: "container:<name>"` puts the app inside another container's
network stack, so the service is reachable at `localhost`. It replaces the
app's own networking, ties the app to one service container by name, and
breaks whenever that container is recreated. It solves nothing the two routes
above do not.

## Which to use

| Situation | Route |
|---|---|
| Linux | Join the `dockyard` network |
| macOS / Windows, app should be isolated from other projects | Through the host |
| macOS / Windows, no preference | Either; the network route is the documented default |

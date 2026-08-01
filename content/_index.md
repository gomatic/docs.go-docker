---
title: go-docker
---

**go-docker** is a stdlib-only Go client for the [Docker Engine REST API](https://docs.docker.com/engine/api/): containers, images, volumes, exec, and the Engine's two stream formats. It speaks HTTP directly over the daemon socket, so consumers inherit **no Docker SDK module graph** — the only non-test dependency is [go-error](https://github.com/gomatic/go-error).

```go
client, err := docker.New(ctx)                    // unix socket or DOCKER_HOST, version negotiated
id, err := client.CreateContainer(ctx, docker.ContainerSpec{
    Name:  "db-5498",
    Image: "postgres:17-alpine",
    Ports: []docker.PortBinding{{HostAddress: "127.0.0.1", Host: 5498, Container: 5432}},
})
err = client.StartContainer(ctx, id)
```

## Why it exists

- **No moby graph in your go.sum.** The upstream `docker/docker` module drags a large transitive dependency surface into every consumer. This client implements the small, stable subset of the Engine API a lifecycle tool needs, with the standard library.
- **Honest failure surfacing.** An image pull that fails mid-stream still returns HTTP 200 — the error arrives inside the JSON progress stream. `PullImage` scans that stream and returns a real error; naive wrappers report false success.
- **Testable without a daemon.** The transport (`Doer`), host, environment lookup, and API version are all injectable options; the entire test suite runs against `httptest` fakes at 100.0% statement coverage with asserted failure paths.

## Surface

| Area | Operations |
| --- | --- |
| Client | `New` (options: `HostAddress`, `APIVersion`, `Connection`, `Environment`), `Ping` |
| Containers | `CreateContainer`, `StartContainer`, `StopContainer`, `RemoveContainer`, `InspectContainer`, `Containers`, `WaitContainer`, `ContainerLogs` |
| Exec | `Exec` (create → detached start → exit-code polling; no connection hijacking) |
| Images | `PullImage` (progress-stream scanned, registry auth option), `ImageExists` |
| Volumes | `CreateVolume`, `InspectVolume`, `Volumes` (label filters), `RemoveVolume` |
| Streams | `DemuxStream` (stdcopy frames → stdout/stderr writers), `ScanStream` (JSON progress) |

## Errors

Every failure is a package sentinel (`errs.Const` from [go-error](https://github.com/gomatic/go-error)), matchable with `errors.Is`: `ErrNotFound`, `ErrConflict`, `ErrBadParameter`, `ErrServer`, `ErrUnexpectedStatus` (mapped from Engine statuses with the daemon's envelope message attached), plus `ErrConnect`, `ErrPing`, `ErrPull`, `ErrCanceled`, `ErrStream`, `ErrStreamError`, `ErrFrameTooLarge`, `ErrEncodeRequest`, `ErrDecodeResponse`, `ErrHost`. Error text is never string-matched — not by the library, and consumers should not either.

## Hardening notes

- Frame payloads are capped (32 MiB) so a corrupt or hostile stream cannot force unbounded allocation; progress-stream lines and response documents are size-bounded the same way.
- Identifiers are path-escaped into request URLs; a hostile name cannot reshape a request path.
- The daemon's in-band system frames (stream byte 3) surface as `ErrStreamError` carrying the daemon's message.

Consumed by [go-pgdocker](https://github.com/gomatic/go-pgdocker) and [pegged](https://github.com/sqlrest/pegged).

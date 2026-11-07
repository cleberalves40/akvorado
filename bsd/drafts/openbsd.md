# Install on OpenBSD (draft)

> [!WARNING]
> **Draft.** This guide is not part of the *Akvorado* documentation yet.
> `distinfo` and `pkg/PLIST` were not produced by the OpenBSD ports tools, and
> the steps below have not been run end to end. Do not use it in production.
> See also [`openbsd/PORTING_STATUS.md`](../../openbsd/PORTING_STATUS.md).

## Status

| Item                        | State                                                                 |
|-----------------------------|-----------------------------------------------------------------------|
| Port                        | `openbsd/ports/net/akvorado`, work files in this repository           |
| Source build and smoke test | ✅ CI on OpenBSD 8.0 amd64 (four services on one host, in-memory Kafka) |
| Architectures               | amd64 and aarch64: `modernc.org/libc` does not build on the others    |
| `make makesum`, `make update-plist`, `portcheck` | ⏳ to do                                         |
| User and group `_akvorado`  | ⏳ ID 906 to register in `infrastructure/db/user.list`                 |
| Logs                        | ⚠️ not captured yet: add `daemon_logger` to the rc scripts            |
| Service supervision         | ⚠️ `rc_bg`; a service that exits is not restarted                     |
| Kafka, ClickHouse           | ❌ not packaged: run them on other hosts                               |
| GeoIP updates               | ❌ no `geoipupdate` package: download the MMDB files by hand          |

## Install

From a ports tree in `/usr/ports`, as root:

```console
# pkg_add git go
# git clone --depth 1 https://github.com/cleberalves40/akvorado.git /tmp/akvorado
# mkdir -p /usr/ports/mystuff/net
# cp -R /tmp/akvorado/openbsd/ports/net/akvorado /usr/ports/mystuff/net/
# cd /usr/ports/mystuff/net/akvorado && make install
```

The package creates the `_akvorado` user and group, and installs the
configuration files in `/etc/akvorado` with `@sample`.

## Configure

The sample configuration is adapted like the FreeBSD one (see
[`bsd/README.md`](../README.md#runtime-adaptations)), with GeoIP databases in
`/var/db/GeoIP`. As Kafka and ClickHouse are not available on OpenBSD, set
their addresses under `kafka.brokers` and `clickhousedb.servers` in
`/etc/akvorado/akvorado.yaml`. Also set `clickhouse.orchestrator-url` to an
address of this host that the ClickHouse server can reach.

```console
# akvorado orchestrator /etc/akvorado/akvorado.yaml --check
```

## Start

Check that Kafka and ClickHouse are reachable first: the orchestrator exits
when they are not, and the other services exit when the orchestrator is not
reachable.

```console
# rcctl enable akvorado_orchestrator akvorado_inlet akvorado_outlet akvorado_console
# rcctl start akvorado_orchestrator
# rcctl start akvorado_inlet akvorado_outlet akvorado_console
# for port in 8080 8081 8082 8083; do akvorado healthcheck --http 127.0.0.1:$port; done
```

To run only a collector on this host, enable only `akvorado_inlet` and
`akvorado_outlet`, and point them to the orchestrator with
`rcctl set akvorado_inlet flags inlet http://<orchestrator>:8080`.

## Before publishing this guide

- Follow the steps in [`openbsd/PORTING_STATUS.md`](../../openbsd/PORTING_STATUS.md):
  `make makesum`, `make update-plist`, `make package`, `portcheck`.
- Add `daemon_logger` to the rc scripts and document where logs go.
- Register `_akvorado` in `user.list`.
- Move this guide to `console/data/docs/` once the steps above are done.

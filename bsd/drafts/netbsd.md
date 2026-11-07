# Install on NetBSD (draft)

> [!WARNING]
> **Draft.** This guide is not part of the *Akvorado* documentation yet. The
> pkgsrc package has not been through `pkglint` or `make package`, and the steps
> below have not been run end to end. Do not use it in production.

## Status

| Item                       | State                                                                 |
|----------------------------|-----------------------------------------------------------------------|
| Package                    | `pkgsrc/net/akvorado`, work files in this repository                  |
| Source build and smoke test | ✅ CI on NetBSD 11.1 amd64 (four services on one host, in-memory Kafka) |
| Architectures              | amd64 only: `modernc.org/libc` does not build on the others           |
| `pkglint`, `make package`  | ⏳ to do                                                              |
| Service supervision        | ⚠️ rc.d scripts with a pidfile; a service that exits is not restarted |
| Log rotation               | ⏳ to do (`newsyslog.conf` entry for `/var/log/akvorado/*.log`)        |
| Kafka                      | `devel/kafka` exists in pkgsrc, not tested with *Akvorado*           |
| ClickHouse                 | ❌ not in pkgsrc: run it on another host                              |
| GeoIP updates              | ❌ no `geoipupdate` package: download the MMDB files by hand          |

## Install

From a pkgsrc tree in `/usr/pkgsrc`, as root:

```console
# pkg_add go127
# git clone --depth 1 https://github.com/cleberalves40/akvorado.git /tmp/akvorado
# cp -R /tmp/akvorado/pkgsrc/net/akvorado /usr/pkgsrc/net/
# echo PKG_RCD_SCRIPTS=YES >> /etc/mk.conf
# cd /usr/pkgsrc/net/akvorado && make install clean
```

`PKG_RCD_SCRIPTS=YES` copies the rc.d scripts to `/etc/rc.d`. Without it,
copy them from `/usr/pkg/share/examples/rc.d`.

The package creates the `akvorado` user and group, installs the configuration
files in `/usr/pkg/etc/akvorado`, and creates `/var/log/akvorado` and
`/var/run/akvorado`.

## Configure

The sample configuration is adapted like the FreeBSD one (see
[`bsd/README.md`](../README.md#runtime-adaptations)), with GeoIP databases in
`/usr/pkg/share/GeoIP`. As ClickHouse is not available on NetBSD, set its
address under `clickhousedb.servers` in `/usr/pkg/etc/akvorado/akvorado.yaml`.
Also set `clickhouse.orchestrator-url` to an address of this host that the
ClickHouse server can reach.

```console
# akvorado orchestrator /usr/pkg/etc/akvorado/akvorado.yaml --check
```

## Start

Start Kafka and check that ClickHouse is reachable first: the orchestrator
exits when they are not, and the other services exit when the orchestrator is
not reachable.

```console
# cat >> /etc/rc.conf <<'EOF'
akvorado_orchestrator=YES
akvorado_inlet=YES
akvorado_outlet=YES
akvorado_console=YES
EOF
# /etc/rc.d/akvorado_orchestrator start
# /etc/rc.d/akvorado_inlet start
# /etc/rc.d/akvorado_outlet start
# /etc/rc.d/akvorado_console start
# for port in 8080 8081 8082 8083; do akvorado healthcheck --http 127.0.0.1:$port; done
```

Logs are in `/var/log/akvorado/<service>.log`.

## Before publishing this guide

- Run `pkglint` and `make package`, and install the binary package on a clean
  host.
- Add log rotation, and decide whether the services need a supervisor.
- Test `devel/kafka` with *Akvorado* and document its setup.
- Move this guide to `console/data/docs/` once the steps above are done.

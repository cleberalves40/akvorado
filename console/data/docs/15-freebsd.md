# Install on FreeBSD

This fork maintains a FreeBSD port of *Akvorado*, `net-mgmt/akvorado`. The port
is not in the official FreeBSD ports tree yet: you build it from this
repository. This guide installs *Akvorado*, Kafka and ClickHouse on a single
FreeBSD host, then explains how to split them across several hosts.

The port packages version `2026.10.1.20261104`, built from upstream commit
[`472d8382`](https://github.com/akvorado/akvorado/commit/472d8382d88e61f5181c13880b02e2808480075c).
It installs:

- `akvorado`, for the orchestrator, outlet and console services,
- `akvorado-inlet`, a smaller binary for the inlet service,
- an rc.d script for each service,
- sample configuration files in `/usr/local/etc/akvorado`, adapted to run every
  service on one host.

> [!NOTE]
> CI builds the port sources on FreeBSD 15.1 (amd64) for every change. It runs
> the tests of the network inputs, then starts the four services on one host
> with an in-memory Kafka broker and checks their healthchecks. The Kafka and
> ClickHouse steps below use the official FreeBSD packages and are not covered
> by CI.

## Requirements

- FreeBSD 15.1, the version tested in CI. The port builds on amd64, aarch64,
  i386 and armv7, but ClickHouse is only packaged for amd64, aarch64 and
  powerpc64le. Use amd64 or aarch64 for a single-host installation.
- The hardware listed in the [installation requirements](10-install.md#requirements).
  Most of it goes to Kafka and ClickHouse.
- Network access during the build, to fetch the distribution files.

## Services and ports

The sample configuration gives each service its own HTTP port. Upstream uses
`:8080` for every service because each one runs in its own container.

| Service      | HTTP    | Other listeners                                   |
|--------------|---------|---------------------------------------------------|
| orchestrator | `:8080` |                                                   |
| inlet        | `:8081` | UDP 2055 (NetFlow), 4739 (IPFIX), 6343 (sFlow)    |
| outlet       | `:8082` | TCP 10179 (BMP)                                   |
| console      | `:8083` |                                                   |
| Kafka        |         | TCP 9092 (clients), 9093 (KRaft controller)       |
| ClickHouse   | `:8123` | TCP 9000 (native protocol)                        |

The *Akvorado* HTTP ports listen on all interfaces. They expose metrics and the
configuration without authentication: only open the ones you need in your
firewall (see [Security](#security)).

## Install Kafka and ClickHouse

Install both from packages:

```console
# pkg install kafka clickhouse
```

Kafka 4 runs without ZooKeeper (KRaft mode). Create its configuration files
from the samples if they do not exist, format its storage directory once, then
start it:

```console
# cd /usr/local/etc/kafka
# for f in server.properties log4j2.yaml; do cp -n $f.sample $f; done
# su -m kafka -c '/usr/local/share/java/kafka/bin/kafka-storage.sh format --standalone \
    -t "$(/usr/local/share/java/kafka/bin/kafka-storage.sh random-uuid)" \
    -c /usr/local/etc/kafka/server.properties'
# sysrc kafka_enable=YES
# service kafka start
```

The default `server.properties` listens on `localhost:9092`, which matches the
*Akvorado* sample configuration. Kafka logs go to syslog with the `kafka` tag.

Start ClickHouse with its default configuration. It listens on the loopback
interface, and the `default` user has no password:

```console
# sysrc clickhouse_enable=YES
# service clickhouse start
# clickhouse client --query 'SELECT version()'
```

## Build and install the port

The port needs a FreeBSD ports tree. It also needs a user and a group,
`akvorado` (830), that are not registered in the official ports tree yet:
`freebsd/ports/UIDs-GIDs.diff` adds them.

```console
# pkg install git-lite go127
# git clone --depth 1 https://git.FreeBSD.org/ports.git /usr/ports
# git clone --depth 1 https://github.com/cleberalves40/akvorado.git /tmp/akvorado
# cp -R /tmp/akvorado/freebsd/ports/net-mgmt/akvorado /usr/ports/net-mgmt/
# cd /usr/ports && patch -p1 < /tmp/akvorado/freebsd/ports/UIDs-GIDs.diff
# make -C /usr/ports/net-mgmt/akvorado install clean
```

Skip the first `git clone` if you already have a ports tree. If the patch does
not apply, UID 830 may have been given to another port in the meantime: tell
the port maintainer.

The build downloads:

- the upstream source archive from GitHub,
- the Go modules (`go.mod` and `go.sum` pin them),
- `akvorado-2026.10.1.20261104-generated-assets.tar.gz`, from the
  [`bsd-distfiles-2026.10.1.20261104`](https://github.com/cleberalves40/akvorado/releases/tag/bsd-distfiles-2026.10.1.20261104)
  release of this repository.

The last file holds what upstream generates with Node.js and other tools before
compiling: the web console, generated Go code and ClickHouse data. The port uses
it so that the build only needs Go. `distinfo` pins the checksum of each file.

Check the installation:

```console
# akvorado version
# akvorado-inlet version
```

## Configure

On the first installation, the package creates the configuration files from
the samples in `/usr/local/etc/akvorado`:

- `akvorado.yaml`: Kafka and ClickHouse addresses, and the orchestrator
  settings. It includes the other three files.
- `inlet.yaml`: flow listeners.
- `outlet.yaml`: enrichment (GeoIP, networks, SNMP, BMP).
- `console.yaml`: web console.

The packaged samples differ from upstream ones in the following ways:

- Kafka, ClickHouse and the orchestrator are at `127.0.0.1`.
- Each service has its own HTTP port, as listed above.
- The console uses a memory cache instead of Redis.
- The inlet uses one worker per UDP port. On FreeBSD, `SO_REUSEPORT` lets
  several sockets bind the same port but does not spread datagrams among them,
  so extra workers would stay idle.
- GeoIP databases are read from `/usr/local/share/GeoIP`.

Then adapt the configuration to your network:

- in `akvorado.yaml`, your AS numbers under `clickhouse.asns`,
- in `outlet.yaml`, your networks under `networks`, and the SNMP credentials of
  your exporters,
- everything described in the [configuration reference](50-configuration.md).

Check the result before starting the services:

```console
# akvorado orchestrator /usr/local/etc/akvorado/akvorado.yaml --check
```

### GeoIP databases

GeoIP databases are optional. By default, the outlet looks for the IPinfo
databases `asn.mmdb` and `country.mmdb` in `/usr/local/share/GeoIP`. Download
them from [IPinfo](https://ipinfo.io) and copy them there.

To use MaxMind GeoLite2 databases instead, install `net/geoipupdate`, put your
account ID and license key in `/usr/local/etc/GeoIP.conf`, and run
`geoipupdate`. It writes to `/usr/local/share/GeoIP`. Then point
`asn-database` and `geo-database` in `outlet.yaml` to `GeoLite2-ASN.mmdb` and
`GeoLite2-Country.mmdb`. Run `geoipupdate` from cron to keep them up to date:
the outlet reloads the databases when they change.

## Start the services

Enable the services, then start the orchestrator first. It creates the Kafka
topic and the ClickHouse tables.

```console
# sysrc akvorado_orchestrator_enable=YES akvorado_inlet_enable=YES \
    akvorado_outlet_enable=YES akvorado_console_enable=YES
# service akvorado_orchestrator start
# service akvorado_inlet start
# service akvorado_outlet start
# service akvorado_console start
```

The orchestrator exits when Kafka or ClickHouse is not reachable, and the other
services exit when the orchestrator is not reachable. The rc.d scripts run each
service under `daemon(8)`, which restarts it after 10 seconds. The services
recover by themselves when they start before their dependencies, for example
at boot.

Check that each service is healthy:

```console
# for port in 8080 8081 8082 8083; do akvorado healthcheck --http 127.0.0.1:$port; done
```

Then open `http://<host>:8083` in a browser, and configure your routers to send
flows to the inlet, as explained in the [exporters guide](11-exporters.md). The
[tutorial](02-collect.md#check-that-the-flows-arrive) shows how to check that
flows arrive; on this setup, the inlet metrics are on port 8081.

### rc.conf variables

| Variable                       | Default                              | Description                                   |
|--------------------------------|--------------------------------------|-----------------------------------------------|
| `akvorado_<service>_enable`    | `NO`                                 | Start the service at boot.                    |
| `akvorado_orchestrator_config` | `/usr/local/etc/akvorado/akvorado.yaml` | Orchestrator configuration file.           |
| `akvorado_inlet_url`, `akvorado_outlet_url`, `akvorado_console_url` | `http://127.0.0.1:8080` | Orchestrator URL to fetch the configuration from. |
| `akvorado_user`                | `akvorado`                           | User running the services.                    |
| `akvorado_restart_delay`       | `10`                                 | Seconds before restarting a service that exited. |

## Logs

The services log to syslog, tagged with the service name (`akvorado_inlet`,
for example). With the default `syslog.conf`, they end up in
`/var/log/messages`. To keep them in a dedicated file:

```console
# cat > /usr/local/etc/syslog.d/akvorado.conf <<'EOF'
!akvorado_orchestrator,akvorado_inlet,akvorado_outlet,akvorado_console
*.*					/var/log/akvorado.log
!*
EOF
# echo '/var/log/akvorado.log	644  7  *  @T00  JC' > /usr/local/etc/newsyslog.conf.d/akvorado.conf
# touch /var/log/akvorado.log
# service syslogd restart
```

## Security

- The console has no authentication of its own. Put it behind a reverse proxy
  that authenticates users, as explained in the [console
  configuration](50-configuration.md#authentication), and only expose the
  proxy.
- Ports 8080, 8081 and 8082 expose metrics and the full configuration. Only
  allow them from the hosts running *Akvorado* services and from your
  monitoring system.
- Only allow the flow and BMP ports from your exporters.

## Several hosts

To spread the services across several hosts, install the port on each host,
then:

- in `akvorado.yaml` on the orchestrator host, set the Kafka brokers, the
  ClickHouse servers and `clickhouse.orchestrator-url`, which ClickHouse uses
  to reach the orchestrator, to addresses reachable from the other hosts;
- on the other hosts, set `akvorado_inlet_url`, `akvorado_outlet_url` or
  `akvorado_console_url` in `/etc/rc.conf` to the orchestrator URL, and only
  enable the services they run.

Kafka and ClickHouse must then listen on an interface the other hosts can
reach: see `listeners` and `advertised.listeners` in Kafka's
`server.properties`, and `listen_host` in ClickHouse's configuration.

One inlet handles each UDP port with a single socket. To receive more flows
than one CPU can handle, run inlets on several hosts. See the [scaling
guide](14-scaling.md#inlet) for other options.

## Upgrade

Read the [changelog](99-changelog.md) first. Then update the port from this
repository and rebuild it:

```console
# git -C /tmp/akvorado pull
# rm -rf /usr/ports/net-mgmt/akvorado
# cp -R /tmp/akvorado/freebsd/ports/net-mgmt/akvorado /usr/ports/net-mgmt/
# make -C /usr/ports/net-mgmt/akvorado deinstall reinstall clean
```

Your configuration files are kept. Restart the orchestrator first, as it
updates the ClickHouse schema if needed, then the inlets and outlets, then the
console:

```console
# service akvorado_orchestrator restart
# service akvorado_inlet restart
# service akvorado_outlet restart
# service akvorado_console restart
```

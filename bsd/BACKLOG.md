# BSD ports backlog

Two teams work on the BSD ports of this fork. Each change is marked with the
nickname of the team that made it, so that this backlog can tell who did what.

| Team        | Scope                                      | Marks                                        |
|-------------|--------------------------------------------|----------------------------------------------|
| **Craudin** | FreeBSD port (maintainers, frozen files)   | `Craudin` label, `[Craudin]` title prefix   |
| **Copi**    | NetBSD (pkgsrc) and OpenBSD ports          | `Copi` label, `[Copi]` title prefix         |

## Conventions

- Pull requests: add the team label and start the title with the team
  nickname, for example `[Copi] pkgsrc: add log rotation`.
- Commits: end the message with a `Team:` trailer, before the other trailers:

  ```
  Team: Copi
  ```

- After merging, add a line to the team's log below, in the same pull request
  or in the next one. Keep the open items up to date.
- Only Craudin can change the frozen FreeBSD files (see [FreeBSD
  freeze](README.md#freebsd-freeze)). Copi works on `pkgsrc/`, `openbsd/` and
  `bsd/drafts/`. Changes to the shared files in `bsd/` and to the CI should be
  announced to the other team.

## Craudin

### Log

| Date       | Reference | Summary |
|------------|-----------|---------|
| 2026-11-06 | release `bsd-distfiles-2026.10.1.20261104` | Generated-assets distfile for the three ports (SHA256 `0f3be371…09c3c`). |
| 2026-11-07 | #1 | Finalize the FreeBSD port; update the pkgsrc and OpenBSD work files; deterministic assets script; smoke test; `build-bsd` CI job on FreeBSD, NetBSD and OpenBSD VMs. |
| 2026-11-07 | #2 | FreeBSD installation guide (`15-freebsd.md`); NetBSD and OpenBSD guides as drafts in `bsd/drafts/`. |
| 2026-11-07 | tag `freebsd-port-2026.10.1.20261104` | FreeBSD port and guide frozen after homologation. |
| 2026-11-07 | #3, #4 | Freeze check; protection of `main` (pull requests only, up-to-date branches, required checks); freeze tested with a throwaway pull request. |
| 2026-11-07 | #5, tag `freebsd-port-2026.10.1.20261104-r2` | Sizing example for a regional ISP in the FreeBSD guide. |
| 2026-11-07 | #6, #7 | Only the FreeBSD maintainers can change the frozen files (`pull_request_target`, `.github/freebsd-maintainers`); team marks and this backlog. |

### Open items

- Run the FreeBSD guide end to end on a FreeBSD 15.1 virtual machine, Kafka and
  ClickHouse included, and check the sizing estimates against real traffic.
- Run `poudriere testport` on the FreeBSD port.
- Add the other Craudin members to `.github/freebsd-maintainers`.
- Confirm the maintainer email of the ports (`cleber@bsd.com.br` or
  `cleber@cloudbsd.com.br`).

## Copi

### Log

| Date | Reference | Summary |
|------|-----------|---------|
|      |           |         |

### Open items

NetBSD (pkgsrc), from [`drafts/netbsd.md`](drafts/netbsd.md):

- Run `pkglint` and `make package`, and install the binary package on a clean
  host.
- Add log rotation for `/var/log/akvorado/*.log`, and decide whether the
  services need a supervisor.
- Test `devel/kafka` with *Akvorado* and document its setup.

OpenBSD, from [`drafts/openbsd.md`](drafts/openbsd.md) and
[`openbsd/PORTING_STATUS.md`](../openbsd/PORTING_STATUS.md):

- Run `make makesum`, `make update-plist`, `make package` and `portcheck` on an
  OpenBSD host.
- Add `daemon_logger` to the rc scripts and document where logs go.
- Register `_akvorado` (906) in `infrastructure/db/user.list`.

Both: move the guide from `bsd/drafts/` to `console/data/docs/` once its
package is validated.

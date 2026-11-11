<!-- markdownlint-disable MD007 -- Unordered list indentation -->
<!-- markdownlint-disable MD010 -- No hard tabs -->
<!-- markdownlint-disable MD055 -- Table pipe style [Expected: leading_and_trailing; Actual: leading_only; Missing trailing pipe] -->
<!-- markdownlint-disable MD041 -- First line in a file should be a top-level heading -->

<!-- TOC ignore:true -->
# Design

Design, requirements, and direction. The active pre-v1.0.0 bug/feature task list lives in `backlog.md`.

- Status: draft, 2026-11-11.

- Author: t00mietum.

<!-- TOC ignore:true -->
## Table of contents

<!-- TOC -->

- [Summary](#summary)
- [Goals and non-goals](#goals-and-non-goals)
- [Assumptions](#assumptions)
- [Commands](#commands)
	- [start](#start)
	- [stop](#stop)
	- [status and watch](#status-and-watch)
	- [setup](#setup)
	- [update](#update)
	- [repair](#repair)
- [Project structure](#project-structure)
	- [Folder structure](#folder-structure)
	- [Logical code structure](#logical-code-structure)
	- [Data flow](#data-flow)
	- [Execution flow/loops](#execution-flowloops)
- [Direction decisions](#direction-decisions)
- [Plan](#plan)
- [Architecture](#architecture)
	- [Software stack](#software-stack)
	- [Shared code](#shared-code)
	- [Configuration model](#configuration-model)
	- [Saves and persistence](#saves-and-persistence)
	- [Privilege model](#privilege-model)
	- [Why a container](#why-a-container)
	- [Run modes](#run-modes)
	- [Tree ownership](#tree-ownership)
	- [Tor](#tor)
	- [Firewall](#firewall)
	- [Wallets](#wallets)
	- [Boot](#boot)
	- [NixOS](#nixos)
	- [ZFS snapshots](#zfs-snapshots)
	- [UI](#ui)
	- [Security](#security)
	- [Testing](#testing)
- [Needs more thought](#needs-more-thought)
	- [Need to sort out conflicts that emerged during design](#need-to-sort-out-conflicts-that-emerged-during-design)

<!-- /TOC -->

## Summary

- `tt-monerod-safeserver` runs a Monero node on Linux as a dedicated, unprivileged account. It starts and stops `monerod`, shows its status, sets it up from scratch, updates it, and repairs a broken install.

- `monerod` runs inside a locked-down container by default, under Podman or Docker. Without a container, such as in a dedicated VM, it runs in a detached tmux session, under the same account.

- The defaults favor privacy over sync speed. All p2p traffic goes over Tor, inbound connections only come in over a Tor onion service, and the RPC is restricted.

- Wallets such as Feather connect to the node's RPC. They can be on the same host, on the LAN, or anywhere through an onion address.

- It runs on Debian and NixOS, and can start `monerod` at boot through a systemd unit.

- Notes on each `monerod` option, and on what does and doesn't work, are in [monerod-options.md](monerod-options.md).

- Wallet setup is in [local-network-wallets.md](local-network-wallets.md).

## Goals and non-goals

- Goals:
	- Run `monerod` as an account that isn't used for anything else, and doesn't have sudo.
	- Keep root work to the few steps that need it, and do it in the foreground.
	- Hand the data tree to the service account before a start, and back to the operator after a stop.
	- Build one static binary that doesn't need anything at run time beyond Podman, Docker, or tmux, plus Tor and sudo.
	- Keep it generic, so the code doesn't have anything in it about one particular host.
	- Allow local network wallets (e.g. Feather) to securely connect, sync, and send/receive txs.
	- Supported platforms: Debian, NixOS.
	- Ability to start at boot

- Non-goals:
	- Wallet management.
	- Mining, or running P2Pool.
	- Hardening the rest of the host, such as SSH.

## Assumptions

- The host runs Debian or NixOS, on amd64 or arm64. Both use systemd.

- On NixOS, the system parts are declared in a NixOS module, rather than changed by `setup`. See [NixOS](#nixos).

- The operator has sudo. Container mode needs Podman or Docker, but not the `docker` group.

- Tor runs as a system service with a SOCKS port on localhost.

- The official release binaries from getmonero.org are used, rather than a distro package.

## Commands

~~~text
tt-monerod start   [--native] [--restart-tor] [--debug] [--dry-run] [--foreground]
tt-monerod stop    [--dry-run]
tt-monerod status
tt-monerod watch   [--interval N]
tt-monerod setup   [--yes] [--dry-run] [setup options]
tt-monerod update  [--yes] [--dry-run] [--self] [--monerod] [--tor] [--image]
tt-monerod repair  [--yes] [--dry-run] [--check]
tt-monerod --help | --about | --version
Common:  --config PATH
~~~

- Exit codes are 0 for success, 1 for an error, and 2 for bad usage.

- `--dry-run` prints every command it would run, with the sudo ones marked, and doesn't run any of them.

- When the boot unit is enabled, `start` and `stop` run `systemctl start` and `systemctl stop` on it instead. That way systemd and the tool don't disagree about what's running.

### start

1. Load and check the config. Refuse when any `monerod` process is already running, under any account.

2. Get sudo once, up front, with `sudo -v`.

3. With Tor on, check that the service is active and its SOCKS port listens. Restart it only when `--restart-tor` is given. Read the onion address and check its format.

4. With `firewall` on, check through sudo that the tool's nftables table is loaded. Refuse to start without it, so `monerod` never runs without its rules.

5. Fetch the ban list over Tor into a temp file, and check that every line is an IP address or range. Install it over the old one only when it passes. After a failed fetch, the old list stays.

6. Take a ZFS snapshot, when a dataset is configured, and prune old ones.

7. Hand the tree to the service account. See [Tree ownership](#tree-ownership).

8. Launch in the configured mode, then wait a few seconds and check that `monerod` is still alive. When it isn't, print the end of its log and exit 1.

- `--debug` runs `monerod` in the foreground with its console, and without `--non-interactive`.

- `--foreground` is for the boot unit. It runs as root, skips sudo, and stays attached to `monerod` until `monerod` exits, so systemd can track it.

### stop

1. When `monerod` isn't running, make sure the tree has gone back to the operator, and exit 0. That way the operator still gets the tree back after a crash.

2. Send SIGTERM, then wait up to `stopTimeout` seconds for a clean exit. Send SIGKILL after that.

3. Take a ZFS snapshot now that the database is closed.

4. Hand the tree back to the operator.

### status and watch

- `status` runs once, and doesn't need sudo. It shows:
	- Whether `monerod` runs, in which mode, as which account, and for how long.
	- Sync height, target height, and peer counts, read from the RPC `get_info` call.
	- Database size, free space, and the ZFS compression ratio.
	- What listens on each configured port, and how many connections each has.
	- Who owns the tree, and whether the boot unit is enabled.
	- A warning when the log shows a transaction that's waiting because there aren't any Tor zone peers connected.
	- The last lines of the log, cut to the terminal width.

- `watch` redraws `status` every few seconds until Ctrl-C.

### setup

- Needs root, so it runs itself again through sudo when needed. It states its plan and asks before it changes anything, unless `--yes` is given.

- On a new install it:
	1. Writes the config, from defaults and the setup options.
	2. Creates the service account and its group, and adds the operator to the group.
	3. Creates the data tree, or uses an existing one.
	4. Downloads and verifies `monerod`, the same way `update` does.
	5. Writes `bitmonero.conf` with a random RPC login, and prints the login once. The password is 32 letters and digits only, since Feather takes the login as part of the node address.
	6. Installs Tor through apt when it's missing, adds a marked block to `torrc` for the onion services, and restarts Tor.
	7. With `firewall` on, installs nftables through apt when `nft` is missing, writes the rules, and starts their unit. See [Firewall](#firewall).
	8. Pulls the pinned container image in container mode.
	9. Installs and enables the boot unit when `startAtBoot` is on.
	10. Copies itself to `/usr/local/bin` when it runs from somewhere else.
	11. Ends with `repair --check`.

- When it finds an install already, it runs `repair` and then `update`.

- It sets `runMode` to `podman` when Podman is installed, and to `docker` when only Docker is. With neither, it offers to install Podman and `catatonit` through apt, or to use native mode. It never installs Docker.

- On NixOS it skips steps 2, 6, 7, 9, and 10. It checks those parts instead, and prints the module settings for whatever is missing.

- It never edits `torrc` lines outside its own marked block. A conflicting onion service is reported, with the lines to remove.

### update

- Needs root, and asks first unless `--yes` is given. When a part isn't named, it updates all 4 parts.

- `--self` gets the latest `tt-monerod-safeserver` release from GitHub, checks its SHA-256 sum, and replaces the running binary in one rename.

- `--monerod` downloads the signed hash list from getmonero.org, and checks its signature with a release key built into the binary. It downloads the newest Linux build for this CPU, checks it against the list, extracts it beside the old versions, and points `exec/current` at it. The last 2 versions stay on disk.
	- A running `monerod` is left alone. It uses the new version on its next start.

- `--tor` upgrades the `tor` package through apt.

- On NixOS, `--self` and `--tor` only print what to change, since both come from the Nix store. `nixos-rebuild` does the rest.

- `--image` pulls the newest container base image through the configured runtime, checks that `monerod --version` works in it, and pins its digest in the config.

- With Tor on, all downloads go over Tor.

### repair

- Needs root unless `--check` is given. `--check` reports only.

- Each check prints OK, FIXED, or FAILED. The exit code is 1 when anything failed.

- Checks, in order:
	1. The config parses, and only root can write it.
	2. The service account and group exist, the account can't log in, and it isn't in any other group.
	3. The operator is in the group.
	4. The data dirs exist.
	5. `exec/current` points at a `monerod` that runs.
	6. Ownership and modes match whether `monerod` runs.
	7. `bitmonero.conf` has an RPC login, and only the account and group can read it.
	8. With Tor on, Tor runs, its SOCKS port listens, the marked `torrc` block matches the config, and the onion addresses exist.
	9. With `firewall` on, the rules file matches the config, its unit is enabled, and the table is loaded. With it off, the unit and the table get removed.
	10. When ufw or firewalld is active, it lets each `rpcAllowFrom` address reach the RPC port. A missing rule is reported with the command that adds it, and never added.
	11. In container mode, the image is there, and an old stopped container gets removed. A container with this tool's label in the other runtime is reported.
	12. With a dataset configured, it exists and is mounted at the base dir.
	13. The ban list exists.
	14. The boot unit is installed and enabled only when `startAtBoot` is on.
	15. Any `monerod` that this tool didn't start is reported, and never killed.

## Project structure

### Folder structure

~~~text
go.mod
source/cmd/tt-monerod/      main package, hands the service to tt-safeserver
source/internal/service/    the Service interface for monerod
source/internal/monerod/    argument list, process match, bitmonero.conf, ban list
source/internal/tor/        service checks, onion address, torrc block
source/internal/firewall/   the nftables rules for monerod
source/internal/rpc/        get_info over digest auth
source/internal/fetch/      downloads over Tor, signature and hash checks
source/internal/update/     update --monerod and --tor
source/internal/repair/     monerod checks and fixes
nix/                        NixOS module and flake
project/                    design, backlog, and notes
cicd/                       build, test, and release scripts
~~~

- The runner, config loading, the account, the ownership hand-off, ZFS, the launchers, the firewall unit, the boot unit, the `repair` runner, self-update, and the CLI come from `tt-safeserver`. See [Shared code](#shared-code).

### Logical code structure

- Every outside command goes through one runner in `tt-safeserver`. The runner logs each command, adds sudo where a step asks for it, and prints instead of running under `--dry-run`. Tests replace it with a fake that records what would run.

- `monerod` arguments are built once, as a list. The launcher only decides where the data dir is mounted.

- `repair` is a list of checks. Each one has a test function and an optional fix function. `setup` reuses the fix functions. The shared checks come from `tt-safeserver`, and the `monerod` ones are added through the Service interface.

### Data flow

- Config file -> checked settings -> argument list -> launcher.

- getmonero.org hash list -> signature check -> hash check of the download -> extract -> `exec/current`.

### Execution flow/loops

- Every command runs once and exits, except `watch`. It loops until interrupted.

- `stop` polls once a second while it waits for `monerod` to exit.

## Direction decisions

- Go was chosen over Bash and PowerShell.
	- `setup`, `update`, and `repair` need HTTPS over SOCKS, JSON, signature checks, and tar and bzip2 handling. Bash would need jq, gpg, and curl, and it gets hard to test past a few hundred lines.
	- PowerShell isn't installed on most Linux servers.
	- Go builds one static binary, and its standard library has nearly all of that. It's also easy to test.

- There are 3 run modes: Podman, Docker, and native tmux. A container is the default.
	- `monerod` parses traffic from strangers. In a container it can only see its binaries and its data dir. [Why a container](#why-a-container) has more.
	- Native mode is for a dedicated VM, where the VM already keeps `monerod` away from everything else, or a host without either runtime. It's a small amount of extra code.

- Podman is preferred over Docker, and both are supported.
	- Podman has no daemon running as root all the time.
	- It's in Debian main and in NixOS, with security updates from the distro, so `setup` can install it the way it installs Tor.
	- It pulls images in its own process, rather than through a daemon, so the pull should be able to go over Tor. That still needs a test.
	- Docker is there because many hosts already have it. The flags are the same for both.

- Both runtimes run as root through sudo, rather than rootless or through the `docker` group.
	- The `docker` group is as good as root on that host, with no password. Sudo asks for one and logs it.
	- `start` already needs sudo for the ownership hand-off, so it doesn't add a prompt.
	- The uids in the container are the host's own, so the ownership hand-off and the firewall's uid match work unchanged. Rootless Podman would map them to a range of other uids.

- Tor runs on the host as a system service, not in a container.
	- In the same container as `monerod`, a bug that lets a peer run code could read the onion service's secret key and take over the onion address.
	- Debian's `tor` package gets security fixes through apt. In an image, a fix waits for a rebuild and a new digest.
	- The tool itself downloads over Tor, sometimes before any container exists.
	- Restarting `monerod` would also restart Tor, and Tor would have to rebuild its circuits and republish the onion service.

- The container runs a stock Debian slim image that's pinned by its digest. The release binaries are mounted into it read-only.
	- A custom image would need its own build and update path. With the binaries mounted, a `monerod` update stays a file copy.

- The container uses host networking.
	- Tor, the onion service forward, and an RPC that's bound to a LAN address all work without changes. Neither runtime adds firewall rules for a host-network container.

- The operator gets sudo for single commands, rather than running the whole tool as root.
	- Downloads, parsing, and status code then never run as root during start and stop.

- The tree is owned by the service account only while `monerod` runs.
	- The operator can edit and back up the tree freely while it's stopped, without sudo.

- Inbound connections come in only through the Tor anonymity zone.
	- `monerod` advertises its onion address only for that zone, so hardly any peers would use a second onion port for the clearnet zone. It would also mix the 2 zones. [monerod-options.md](monerod-options.md) has the details.

- The RPC login is in `bitmonero.conf`, rather than on the command line.
	- Anything on the command line shows in `ps` and in the runtime's `inspect`.

- A LAN wallet uses the node's LAN address by default, rather than an onion address.
	- Feather sends traffic for local addresses directly, never over Tor, and wallet sync over Tor is slow.
	- An onion address for the RPC is there for wallets away from the LAN.

- Start at boot uses a systemd unit that runs `tt-monerod start --foreground` as root, rather than a unit that runs `monerod` directly.
	- The ownership hand-off, snapshots, and Tor checks then work the same way at boot as by hand.

- On NixOS, a NixOS module declares the account, the onion services, Podman or Docker, the config file, and the boot unit.
	- `/etc` and the Nix store are managed by Nix, so `setup` and `repair` can't change them there.
	- NixOS's own `services.monero` module was looked at. It runs `monerod` directly, without the container, the ownership hand-off, or the Tor setup.

- A bad line in the config gets a warning and also its default, rather than stopping the load.
	- Every default is the safe choice, such as Tor on and the RPC on localhost.

- The config format is shcl, and the tool reads it with shcl's Go module.

- Code that `tt-minecraft-safeserver` needs too comes from a shared Go module, `tt-safeserver`.
	- Each tool keeps its own repo, binary, and releases. This one is `tt-monerod`, and the other is `tt-minecraftd`.
	- Each tool supplies its own service behind one Service interface. The module runs the common commands.
	- The module gets its own repo when this project starts on code, so 2 tools test the interface. Until then its code is written in `tt-minecraft-safeserver`, and its design is there too. After that both tools keep it up.
	- It's GPL v2 or later, the same as both tools.

- The firewall rules go in the tool's own nftables table, rather than in ufw, firewalld, or `nftables.conf`.
	- A separate table works beside any of those, or none, and only the tool edits it.
	- A drop in any table wins, so the tool's table can't open a port that another firewall keeps closed.

- fail2ban was looked at for the RPC login, and left out.
	- On a LAN address, the firewall already limits the RPC to the addresses in `rpcAllowFrom`.
	- Over the RPC onion service, every wallet connects from Tor on localhost. fail2ban can't tell them apart, so a ban would block all of them.
	- The RPC password is 32 random letters and digits, which is too many to guess.
	- `monerod` bans misbehaving p2p peers on its own, and the ban list covers known bad ranges.

## Plan

- The first release is `v1.0.0-alpha.1`.

- The shared code moves to `tt-safeserver` before step 1.

1. `start`, `stop`, `status`, and `watch`, with `--dry-run` and tests.

2. `repair`, since `setup` reuses its fixes.

3. `setup`.

4. `update`.

5. Start at boot, and wallet access through an onion address.

6. The NixOS module.

7. Build and release scripts, and an install one-liner.

8. README.

## Architecture

### Software stack

- The tool is written in Go, and it uses the standard library wherever it can.

- Expected outside modules:
	- `tt-safeserver`, `github.com/t00mietum/tt-safeserver`. See [Shared code](#shared-code).
	- An OpenPGP package, for the getmonero.org signature.
	- The shcl Go module, `github.com/yottacore/shcl/source/go/v3`, at its latest published beta. It comes in through `tt-safeserver`.

- At run time it calls `podman`, `docker`, or `tmux`, plus `sudo`, `systemctl`, and `ss`. It also calls `zfs` when snapshots are on, `nft` when the firewall is on, and `apt-get` for `update --tor` on Debian. `repair` calls `ufw` or `firewall-cmd` to read their rules, when one is active.

### Shared code

- The parts that `tt-minecraft-safeserver` needs too come from `tt-safeserver`. This tool supplies a `monerod` service behind its Service interface, and doesn't keep its own copy of any shared part.

- From `tt-safeserver`:
	- The command runner, with sudo, logging, and `--dry-run`.
	- Config loading, through shcl. The keys and their defaults are set here.
	- The service account and its group.
	- The ownership hand-off. See [Tree ownership](#tree-ownership).
	- ZFS snapshots and pruning.
	- The Podman, Docker, and native launchers.
	- The nftables firewall unit, and writing and loading its rules file.
	- The boot unit.
	- The `repair` runner, and the checks both tools have, such as the account, ownership, the firewall unit, other firewalls, the image, the dataset, the boot unit, and a stray process.
	- `update --self`, and pulling and pinning the image for `update --image`.
	- The CLI, with the common commands and options, `--help`, `--about`, and `--version`.

- Stays here:
	- The `monerod` argument list, and how to spot a running `monerod`.
	- Tor, with the service checks, the onion addresses, and the marked `torrc` block.
	- The firewall rules themselves, from `useTor`, `rpcAllowFrom`, and the bind addresses.
	- `bitmonero.conf` and the RPC login.
	- The ban list.
	- The `status` lines from `get_info`, and the queued transaction warning.
	- `update --monerod` with its signature check, `update --tor`, and the `monerod --version` check in `update --image`.
	- The `repair` checks and `setup` steps for `exec/current`, `bitmonero.conf`, Tor, and the ban list.
	- `--restart-tor`.
	- The NixOS module.

- Where this design and the shared one don't agree yet is listed under [Need to sort out conflicts that emerged during design](#need-to-sort-out-conflicts-that-emerged-during-design).

### Configuration model

- One file, `/etc/tt-monerod/config.shcl`, owned by root and readable by all. `--config` points elsewhere.

- An unknown key or a bad value gets a warning each run, and the default is used.

- Settings:

| Key                | Default                       | Notes
| :---               | :---                          | :---
| `runMode`          | `podman`                      | `podman`, `docker`, or `native`. `setup` picks one
| `runUser`          | `monerod`
| `runGroup`         | same as `runUser`             | Stays the group before and after a run
| `ownerAfterStop`   | the operator                  | From `SUDO_USER`, else `USER`
| `baseDir`          | `/srv/monerod`
| `binDir`           | `<baseDir>/exec/current`
| `dataDir`          | `<baseDir>/data`
| `ownerTrees`       | `[<baseDir>]`                 | Every tree that changes hands
| `zfsDataset`       | none                          | No snapshots when unset
| `image`            | Debian trixie slim, by digest | `update --image` changes it
| `memCap`           | `8g`                          | Swap is capped the same, so it stays at 0
| `nice`             | `10`
| `maxConcurrency`   | `2`
| `useTor`           | `true`
| `allowInbound`     | `true`                        | Only works with `useTor`
| `torSocks`         | `127.0.0.1:9050`
| `torServiceDir`    | `/var/lib/tor/monerod/`
| `anonInboundPort`  | `28080`                       | Same number for the onion and local port
| `anonInboundMax`   | `8`
| `p2pBindIP`        | `127.0.0.1`
| `p2pPort`          | `18080`
| `rpcBindIP`        | `127.0.0.1`                   | The LAN address when the wallet is on another host
| `rpcPort`          | `18081`
| `rpcAllowFrom`     | none                          | Addresses or ranges that may reach an RPC on a LAN address
| `outPeers`         | 16 with Tor, 32 without
| `blockSyncSize`    | 20 with Tor, 100 without
| `zmq`              | `false`                       | Turn on only for P2Pool and the like
| `banListURL`       | a public ban list             | Empty skips it
| `stopTimeout`      | `240`                         | Seconds
| `rpcOnion`         | `false`                       | An onion address for the RPC, for wallets away from the LAN
| `rpcOnionPort`     | `18089`                       | Restricted RPC listener on localhost, and its onion port
| `torRpcServiceDir` | `/var/lib/tor/monerod-rpc/`
| `firewall`         | `true`                        | See [Firewall](#firewall)
| `startAtBoot`      | `false`
| `extraArgs`        | none                          | Added to the `monerod` command as given

### Saves and persistence

- The tool doesn't store any state of its own. Whether `monerod` runs comes from the process table and the container runtimes. Ownership comes from the files.

- Data tree layout:

~~~text
<baseDir>/exec/<release folder>/   unpacked releases, last 2 kept
<baseDir>/exec/current             link to one of them
<baseDir>/data/bitmonero.conf      log level and RPC login
<baseDir>/data/ban_list.txt
<baseDir>/data/lmdb/
<baseDir>/data/logs/monerod.log
~~~

### Privilege model

| Step                                    | Runs as
| :---                                    | :---
| Parsing, checks, `status`, `watch`      | operator
| Ban list and other downloads            | operator, or root in `update`
| Read the onion address                  | operator through sudo
| Read the firewall table                 | operator through sudo
| Restart Tor, with `--restart-tor` only  | operator through sudo
| ZFS snapshot and prune                  | operator through `zfs allow`, else sudo
| Change tree ownership                   | operator through sudo
| Container run, stop, and pull           | operator through sudo
| `sudo -u <runUser> tmux` in native mode | operator through sudo
| `setup`, `update`, and `repair`         | root
| `start --foreground` from the boot unit | root, from systemd
| `monerod`                               | the service account

- All sudo happens in the foreground, right after one `sudo -v`. Sudo never runs inside a detached session.

- The service account never runs `tt-monerod`, so that way it doesn't need to read the binary.

### Why a container

- `monerod` reads data from strangers all day. Every peer sends it blocks, transactions, and handshakes, and `monerod` has had bugs before that let a peer crash it from the network. A worse bug could let a peer run code as the service account.

- The service account limits what that code could do, but only so far. Outside a container it can still:
	- Read any file that's readable by everyone, which is most of `/etc` and often parts of home dirs.
	- See every process on the host, with its command line.
	- Run any program that's installed on the host.
	- Use as much memory and as many processes as the kernel allows.

- A container takes most of that away, under Podman or Docker:
	- It sees the image's files and the release binaries, both read-only, and the data dir. The rest of the host's files aren't mounted.
	- It has its own process list, so it can't see or signal host processes.
	- `--cap-drop ALL` and `no-new-privileges` mean a setuid program in the image can't give it root.
	- The pid limit and the memory cap stop a runaway `monerod` from using up the host's processes or memory. When memory runs out anyway, `--oom-score-adj 1000` has the kernel kill `monerod` before the desktop or other services.

- It helps day to day, too:
	- The image is pinned by its digest, so every start uses the same files whatever the host has installed. The release binaries only need libc, libm, and libpthread from it.
	- A stopped container keeps its output, so the runtime's `logs` shows why `monerod` died.
	- The runtime's `stop` sends SIGTERM and waits for `stopTimeout`, and `--init` passes the signal on to `monerod`.

- What it doesn't do:
	- With host networking, `monerod` can still connect to any port on localhost and on the LAN. Keeping p2p on Tor is up to the `monerod` options. With Tor on, the [Firewall](#firewall) rules block the LAN and the internet, but not localhost.
	- The container shares the host's kernel, so a kernel bug could still let code out. It's less isolation than a VM.
	- The runtime runs as root, through sudo. A bug in the runtime itself is a root bug.

- In native mode, `monerod` is limited by file modes only. That's enough in a dedicated VM, where nothing else is there to protect.

### Run modes

- Container mode runs `podman` or `docker` through sudo, and the container under the service account's uid and gid. Both get the same flags:
	- `--read-only`, a small tmpfs on `/tmp`, `--cap-drop ALL`, and `no-new-privileges`.
	- A pid limit, the memory cap, and `--oom-score-adj 1000`, so the desktop isn't what gets killed when memory runs out.
	- `--init`, so SIGTERM gets passed to `monerod` and child processes get reaped.
	- `nice` and `umask 0007` set inside, before `monerod` starts.
	- No `--rm`, so the runtime's `logs` still works after a crash. `start` removes the old container, and only one with this tool's label.

- `stop` and `repair` look for the tool's label in both runtimes. A container started before a `runMode` change still gets found.

- Podman's `--init` needs `catatonit`, so `setup` installs it with Podman and `repair` checks for it.

- `status` doesn't use sudo, so it tells the modes apart from the process table, by `monerod`'s parent process.

- Native mode runs `monerod` in the service account's own tmux server, on a named socket. It never uses the operator's tmux server.
	- The account's shell is `nologin`, so the tool sets `SHELL=/bin/sh` for tmux.
	- `start` prints the attach command.

### Tree ownership

- Each tree in `ownerTrees` changes hands with `chown -R`. A missing tree gets a warning.
	- Before a start, the trees go to `runUser`. After a stop, they go to `ownerAfterStop`.
	- The group is always `runGroup`, and the operator is in it. So logs and `status` work while `monerod` runs.

- At start, modes become `ug+rwX,o-rwx`. `monerod` runs with `umask 0007`, so new files follow the same rule.

- A tree that's `/` or less than 3 levels deep is refused.

### Tor

- `monerod` gets 3 Tor settings. `--proxy` is for all clearnet p2p, `--tx-proxy` is for its own transactions, and `--anonymous-inbound` is for inbound peers.

- `setup` adds one onion service to `torrc`, inside a marked block. It forwards `anonInboundPort` on the onion address to the same port on localhost.

- The tool never restarts Tor during `start` unless asked. A restart drops every other Tor user on the box.

- With `rpcOnion` on, `setup` adds a second onion service, in its own service dir, per `ANONYMITY_NETWORKS.md`. It forwards `rpcOnionPort` to a restricted RPC listener on localhost, which `monerod` gets through `--rpc-restricted-bind-ip` and `--rpc-restricted-bind-port`.

- A transaction that a wallet sends through the RPC goes out only to Tor zone peers. Without any connected, `monerod` queues it and logs an error, then sends it once a Tor zone peer connects. `status` warns when the log shows a queued transaction.

### Firewall

- With `firewall` on, the tool keeps its rules in its own nftables table, `inet tt_monerod`, so they cover IPv4 and IPv6.

- `setup` and `repair` write the rules to `/etc/tt-monerod/firewall.nft`, from the config. `tt-monerod-firewall.service` loads that file at boot.
	- It loads it again whenever `nftables.service` starts, restarts, or reloads. Debian's `nftables.conf` flushes every table, the tool's included.
	- The file deletes the old table and adds the new one in a single nft transaction, so there's no gap without rules.

- Outbound, with `useTor` on:
	- A new connection from the service account to anything but loopback gets dropped and counted. A rate-limited log line names the address.
	- Replies to connections that came in, such as a LAN wallet's RPC traffic, still go out.
	- Tor runs as its own account, so its traffic isn't touched.
	- It works the same in every mode, since a host-network container's sockets belong to the service account's uid.

- Inbound, new connections from anywhere but loopback get dropped:
	- On the RPC port, unless they come from an address in `rpcAllowFrom`.
	- On `anonInboundPort` and `rpcOnionPort`, always. Tor reaches them over loopback.
	- On `p2pPort`, unless `p2pBindIP` isn't a loopback address.

- The table only drops traffic. A drop in another table still applies, so ufw or firewalld also has to let the wallet's host in. `repair` checks for that rule, and prints the command when it's missing. The tool never changes another firewall's rules.

- A ufw reload or disable only rewrites ufw's own tables, so the tool's table stays loaded.

### Wallets

- Setup for Feather and other wallets, on the same host, on the LAN, or away from it, is in [local-network-wallets.md](local-network-wallets.md).

- The RPC binds to localhost unless a wallet on another host needs it. On a LAN address, `rpcAllowFrom` lists the hosts that may reach the RPC port. See [Firewall](#firewall).

### Boot

- With `startAtBoot` on, `setup` and `repair` install `tt-monerod.service` and enable it.
	- `ExecStart` runs `tt-monerod start --foreground`, and `ExecStop` runs `tt-monerod stop`.
	- `TimeoutStopSec` is a little longer than `stopTimeout`, so systemd doesn't kill `monerod` first.
	- It starts after `network-online.target`, and after `tor.service` when Tor is on. In Docker mode it also waits for `docker.service`. Podman has no service to wait for.
	- With `firewall` on, it requires `tt-monerod-firewall.service`, so `monerod` never starts at boot without the rules.

- With `startAtBoot` off, `repair` disables and removes the unit.

### NixOS

- The repo has a flake with a NixOS module. It declares:
	- The service account and group.
	- Tor, with the p2p onion service and the optional RPC onion service.
	- Podman or Docker, in container mode.
	- `/etc/tt-monerod/config.shcl`, written from module options.
	- The firewall unit and its rules.
	- The boot unit.

- `tt-monerod` detects NixOS by the `/etc/NIXOS` file. There, `setup` and `repair` still handle the data tree, the `monerod` download, and `bitmonero.conf`. For the rest they only check, and print the module options to set.

### ZFS snapshots

- With `zfsDataset` set, `start` takes `<YYYYmmDD-HHMMSS>_pre_monerod-start`, and `stop` takes `<YYYYmmDD-HHMMSS>_post_monerod-stop` after the database closes.

- Old ones get pruned on a grandfather-father-son schedule. The ones kept are the first, the newest 3, and the last of each of the last 3 days, 2 weeks, and 2 months.
	- Only snapshots that match the tool's name pattern are touched.

- `setup` offers `zfs allow` for snapshot, destroy, and mount on the dataset, so this works without sudo.

### UI

- Output is plain text, with a blank line at the start and the end, and one between sections. It uses color only on a terminal.

- `--help` lists the commands and options. `--about` describes the tool. `--version` prints the version and build.

### Security

- The service account doesn't have sudo, a login shell, a home, or any other groups.

- In container mode `monerod` can't see anything outside its binaries and data. In native mode it's limited by file modes only.

- The config doesn't have any secrets in it. The RPC login is in `bitmonero.conf`, with mode 0660.

- The RPC binds to localhost unless a wallet on another host needs it. On a LAN address, the firewall lets only the hosts in `rpcAllowFrom` reach it.

- With Tor and the firewall on, a leak from `monerod` gets dropped and logged, rather than reaching the LAN or the internet.

- Every download is checked before use. `monerod` releases need a good signature on the hash list and a matching hash. Tool releases need a matching SHA-256 sum.

- With Tor on, the tool and `monerod` don't make any clearnet connections. That includes DNS, since `--proxy` blocks DNS outside the proxy, and update checks are off.

### Testing

- Unit tests run with the fake command runner. They check the exact command lists for each mode, with Tor on and off, and the config errors. They also check the rendered `firewall.nft` for each mix of `useTor`, `rpcAllowFrom`, and bind addresses.

- A smoke test runs `monerod --offline` in a container against an empty data dir, once under each runtime. It checks the hardening flags, the log, and a clean exit on the runtime's `stop`.

- A live start and stop on a real node checks ownership, listening ports, and that every outbound connection goes to the Tor SOCKS port.

- With the firewall on, a connection from the service account to a LAN address fails and bumps the drop counter. A wallet in `rpcAllowFrom` still syncs.

## Needs more thought

- Stopping `nftables.service` flushes every table, the tool's included, and the table stays gone until the next start. `monerod` keeps running without its rules in the meantime.

- Tool releases could be signed, beyond a SHA-256 sum.

- `memCap` of 8 GB is a guess. LMDB's mapped pages count against the cap, so sync may slow down under it.

- Rootless Podman. Code that got out of the container would land as an unused uid, not the service account. It needs subuid ranges, linger, and cgroup delegation for a `nologin` account, and its uid mapping changes [Tree ownership](#tree-ownership) and the firewall's uid match.

- Tor in its own container, beside `monerod`'s. `monerod` could then sit on a network whose only way out is Tor, which would do the firewall's job in container mode. A LAN wallet would need a relay to reach the RPC, since an internal network can't publish ports.

### Need to sort out conflicts that emerged during design

Where this design differs from the shared one in `tt-safeserver`. Each one gets settled in one place or the other before the code moves.

- Native mode. Here it's a bare tmux session under the service account, limited by file modes only. The shared design runs tmux inside a transient systemd unit from `systemd-run`, with systemd's sandbox options, and its socket under `/run/`. That looks like a gain here too. Taking it changes [Run modes](#run-modes), [Why a container](#why-a-container), [Privilege model](#privilege-model), and [Security](#security).

- Stopping. Here it's SIGTERM, then SIGKILL after `stopTimeout`. The shared design sends a console `stop` command first, then SIGTERM, then SIGKILL 30 seconds later, and has `stop --warn N`. The service needs a way to supply or skip that first step. `monerod` closes its database cleanly on SIGTERM.

- Start check. Here `start` waits a few seconds and checks that `monerod` is still alive. The shared design waits for a log line that says the server is loaded, up to `startTimeout`. The service needs to supply its own test.

- Container mounts. Here the data dir is mounted, and the release binaries read-only, with a small `/tmp`. The shared design mounts each tree in `ownerTrees` at its own path, with read-only files on top, an exec `/tmp`, and `-i -t` for the console. Mounting `<baseDir>` whole would make the binaries writable here, so the service needs to supply its mounts.

- Downloads over Tor. With Tor on, every download here goes over Tor, `update --self` included. The shared self-update has no proxy, so it needs an HTTP client from the service. For `update --image`, Podman pulls in its own process and should take Tor's SOCKS address through `HTTPS_PROXY`, which still needs a test. Docker pulls through its daemon, which this tool doesn't give a proxy, so Docker mode isn't covered yet in either design.

- Firewall rules. Here the outbound drop is only on with Tor on, and loopback is allowed, since Tor's SOCKS port is there. The shared design drops loopback and the private ranges, and allows ports 80 and 443 with `egress` set to `web`. The rules come from the service either way, but the table layout, counters, and logging have to fit both. `status` there shows the drop counters, and could here too.

- Snapshot names and pruning. Here they're `_pre_monerod-start` and `_post_monerod-stop`, keeping the last of 3 days, 2 weeks, and 2 months. The shared design names them after the binary, and keeps 7 days, 4 weeks, and 6 months. Either the schedule becomes a setting, or one schedule fits both.

- `repair` order. Here the `monerod` checks sit between the shared ones, such as `exec/current` at 5 and the ban list at 13. The shared runner needs to say where service checks go. The dataset check here also wants it mounted at `<baseDir>`.

- The shcl version. Here it's the latest published beta. The shared design says the latest published release, beta or not. Both are `v3.0.0-beta1` today, but one module pins one version for both.

- The CLI. `start --restart-tor` is only here, and `stop --warn`, `console`, and `cmd` are only there. The shared CLI needs to take flags and commands from the service.

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
	- [Configuration model](#configuration-model)
	- [Saves and persistence](#saves-and-persistence)
	- [Privilege model](#privilege-model)
	- [Why Docker](#why-docker)
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

<!-- /TOC -->

## Summary

- `tt-monerod-safeserver` runs a Monero node on Linux as a dedicated, unprivileged account. It starts and stops `monerod`, shows its status, sets it up from scratch, updates it, and repairs a broken install.

- `monerod` runs inside a locked-down Docker container by default. Without Docker it runs in a detached tmux session, under the same account.

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
	- Build one static binary that doesn't need anything at run time beyond Docker or tmux, Tor, and sudo.
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

- The operator has sudo, and is in the `docker` group for Docker mode.

- Tor runs as a system service with a SOCKS port on localhost.

- The official release binaries from getmonero.org are used, rather than a distro package.

## Commands

~~~text
tt-monerodctl start   [--native] [--restart-tor] [--debug] [--dry-run] [--foreground]
tt-monerodctl stop    [--dry-run]
tt-monerodctl status
tt-monerodctl watch   [--interval N]
tt-monerodctl setup   [--yes] [--dry-run] [setup options]
tt-monerodctl update  [--yes] [--dry-run] [--self] [--monerod] [--tor] [--image]
tt-monerodctl repair  [--yes] [--dry-run] [--check]
tt-monerodctl --help | --about | --version
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
	8. Pulls the pinned container image in Docker mode.
	9. Installs and enables the boot unit when `startAtBoot` is on.
	10. Copies itself to `/usr/local/bin` when it runs from somewhere else.
	11. Ends with `repair --check`.

- When it finds an install already, it runs `repair` and then `update`.

- It never installs Docker. Without Docker, it offers native mode.

- On NixOS it skips steps 2, 6, 7, 9, and 10. It checks those parts instead, and prints the module settings for whatever is missing.

- It never edits `torrc` lines outside its own marked block. A conflicting onion service is reported, with the lines to remove.

### update

- Needs root, and asks first unless `--yes` is given. When a part isn't named, it updates all 4 parts.

- `--self` gets the latest `tt-monerod-safeserver` release from GitHub, checks its SHA-256 sum, and replaces the running binary in one rename.

- `--monerod` downloads the signed hash list from getmonero.org, and checks its signature with a release key built into the binary. It downloads the newest Linux build for this CPU, checks it against the list, extracts it beside the old versions, and points `exec/current` at it. The last 2 versions stay on disk.
	- A running `monerod` is left alone. It uses the new version on its next start.

- `--tor` upgrades the `tor` package through apt.

- On NixOS, `--self` and `--tor` only print what to change, since both come from the Nix store. `nixos-rebuild` does the rest.

- `--image` pulls the newest container base image, checks that `monerod --version` works in it, and pins its digest in the config.

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
	11. In Docker mode, the image is there, and an old stopped container gets removed.
	12. With a dataset configured, it exists and is mounted at the base dir.
	13. The ban list exists.
	14. The boot unit is installed and enabled only when `startAtBoot` is on.
	15. Any `monerod` that this tool didn't start is reported, and never killed.

## Project structure

### Folder structure

~~~text
go.mod
source/cmd/tt-monerodctl/   main package, argument parsing
source/internal/config/     config load and checks
source/internal/host/       command runner, sudo, dry run
source/internal/monerod/    monerod argument list
source/internal/launch/     Docker and tmux modes
source/internal/tree/       ownership hand-off
source/internal/zfs/        snapshots and pruning
source/internal/tor/        service checks, onion address, torrc block
source/internal/firewall/   nftables rules and their unit
source/internal/rpc/        get_info over digest auth
source/internal/fetch/      downloads over Tor, signature and hash checks
source/internal/update/
source/internal/repair/
source/internal/boot/       systemd unit
nix/                        NixOS module and flake
project/                    design, backlog, and notes
cicd/                       build, test, and release scripts
~~~

### Logical code structure

- Every outside command goes through one runner in `host`. The runner logs each command, adds sudo where a step asks for it, and prints instead of running under `--dry-run`. Tests replace it with a fake that records what would run.

- `monerod` arguments are built once, as a list. The launcher only decides where the data dir is mounted.

- `repair` is a list of checks. Each one has a test function and an optional fix function. `setup` reuses the fix functions.

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

- Docker mode is the default, and native tmux mode is the fallback.
	- `monerod` parses traffic from strangers. In a container it can only see its binaries and its data dir. [Why Docker](#why-docker) has more.
	- Native mode is a small amount of extra code, and it lets the tool work where Docker isn't installed.

- The container runs a stock Debian slim image that's pinned by its digest. The release binaries are mounted into it read-only.
	- A custom image would need its own build and update path. With the binaries mounted, a `monerod` update stays a file copy.

- The container uses host networking.
	- Tor, the onion service forward, and an RPC that's bound to a LAN address all work without changes. Docker doesn't add firewall rules for a host-network container.

- The operator gets sudo for single commands, rather than running the whole tool as root.
	- Downloads, parsing, and status code then never run as root during start and stop.

- The tree is owned by the service account only while `monerod` runs.
	- The operator can edit and back up the tree freely while it's stopped, without sudo.

- Inbound connections come in only through the Tor anonymity zone.
	- `monerod` advertises its onion address only for that zone, so hardly any peers would use a second onion port for the clearnet zone. It would also mix the 2 zones. [monerod-options.md](monerod-options.md) has the details.

- The RPC login is in `bitmonero.conf`, rather than on the command line.
	- Anything on the command line shows in `ps` and `docker inspect`.

- A LAN wallet uses the node's LAN address by default, rather than an onion address.
	- Feather sends traffic for local addresses directly, never over Tor, and wallet sync over Tor is slow.
	- An onion address for the RPC is there for wallets away from the LAN.

- Start at boot uses a systemd unit that runs `tt-monerodctl start --foreground` as root, rather than a unit that runs `monerod` directly.
	- The ownership hand-off, snapshots, and Tor checks then work the same way at boot as by hand.

- On NixOS, a NixOS module declares the account, the onion services, Docker, the config file, and the boot unit.
	- `/etc` and the Nix store are managed by Nix, so `setup` and `repair` can't change them there.
	- NixOS's own `services.monero` module was looked at. It runs `monerod` directly, without the container, the ownership hand-off, or the Tor setup.

- A bad line in the config gets a warning and also its default, rather than stopping the load.
	- Every default is the safe choice, such as Tor on and the RPC on localhost.

- The config format is shcl, and the tool reads it with shcl's Go module.

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
	- An OpenPGP package, for the getmonero.org signature.
	- The shcl Go module, `github.com/yottacore/shcl/source/go/v3`, at its latest published beta.

- At run time it calls `docker` or `tmux`, `sudo`, `systemctl`, and `ss`. It also calls `zfs` when snapshots are on, `nft` when the firewall is on, and `apt-get` for `update --tor` on Debian. `repair` calls `ufw` or `firewall-cmd` to read their rules, when one is active.

### Configuration model

- One file, `/etc/tt-monerodctl/config.shcl`, owned by root and readable by all. `--config` points elsewhere.

- An unknown key or a bad value gets a warning each run, and the default is used.

- Settings:

| Key                | Default                       | Notes
| :---               | :---                          | :---
| `runMode`          | `docker`                      | `docker` or `native`
| `runUser`          | `monerod`
| `runGroup`         | same as `runUser`             | Stays the group before and after a run
| `ownerAfterStop`   | the operator                  | From `SUDO_USER`, else `USER`
| `baseDir`          | `/srv/monerod`
| `binDir`           | `<baseDir>/exec/current`
| `dataDir`          | `<baseDir>/data`
| `ownerTrees`       | `[<baseDir>]`                 | Every tree that changes hands
| `zfsDataset`       | none                          | No snapshots when unset
| `dockerImage`      | Debian trixie slim, by digest | `update --image` changes it
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

- The tool doesn't store any state of its own. Whether `monerod` runs comes from the process table and Docker. Ownership comes from the files.

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
| `docker run` and `docker stop`          | operator, in the `docker` group
| `sudo -u <runUser> tmux` in native mode | operator through sudo
| `setup`, `update`, and `repair`         | root
| `start --foreground` from the boot unit | root, from systemd
| `monerod`                               | the service account

- All sudo happens in the foreground, right after one `sudo -v`. Sudo never runs inside a detached session.

- The service account never runs `tt-monerodctl`, so that way it doesn't need to read the binary.

### Why Docker

- `monerod` reads data from strangers all day. Every peer sends it blocks, transactions, and handshakes, and `monerod` has had bugs before that let a peer crash it from the network. A worse bug could let a peer run code as the service account.

- The service account limits what that code could do, but only so far. Outside a container it can still:
	- Read any file that's readable by everyone, which is most of `/etc` and often parts of home dirs.
	- See every process on the host, with its command line.
	- Run any program that's installed on the host.
	- Use as much memory and as many processes as the kernel allows.

- In Docker mode the container takes most of that away:
	- It sees the image's files and the release binaries, both read-only, and the data dir. The rest of the host's files aren't mounted.
	- It has its own process list, so it can't see or signal host processes.
	- `--cap-drop ALL` and `no-new-privileges` mean a setuid program in the image can't give it root.
	- The pid limit and the memory cap stop a runaway `monerod` from using up the host's processes or memory. When memory runs out anyway, `--oom-score-adj 1000` has the the kernel kill `monerod` before the desktop or other services.

- It helps day to day, too:
	- The image is pinned by its digest, so every start uses the same files whatever the host has installed. The release binaries only need libc, libm, and libpthread from it.
	- A stopped container keeps its output, so `docker logs` shows why `monerod` died.
	- `docker stop` sends SIGTERM and waits for `stopTimeout`, and `--init` passes the signal on to `monerod`.

- What it doesn't do:
	- With host networking, `monerod` can still connect to any port on localhost and on the LAN. Keeping p2p on Tor is up to the `monerod` options. With Tor on, the [Firewall](#firewall) rules block the LAN and the internet, but not localhost.
	- The container shares the host's kernel, so a kernel bug could still let code out. It's less isolation than a VM.
	- Being in the `docker` group is as good as root on that host. The operator has sudo already, so that doesn't add a new risk.

- Native mode is for hosts without Docker. There, `monerod` is limited by file modes only.

### Run modes

- Docker mode runs the container under the service account's uid and gid, with:
	- `--read-only`, a small tmpfs on `/tmp`, `--cap-drop ALL`, and `no-new-privileges`.
	- A pid limit, the memory cap, and `--oom-score-adj 1000`, so the desktop isn't what gets killed when memory runs out.
	- `--init`, so SIGTERM gets passed to `monerod` and child processes get reaped.
	- `nice` and `umask 0007` set inside, before `monerod` starts.
	- No `--rm`, so `docker logs` still works after a crash. `start` removes the old container, and only one with this tool's label.

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

- With `firewall` on, the tool keeps its rules in its own nftables table, `inet tt_monerodctl`, so they cover IPv4 and IPv6.

- `setup` and `repair` write the rules to `/etc/tt-monerodctl/firewall.nft`, from the config. `tt-monerodctl-firewall.service` loads that file at boot.
	- It loads it again whenever `nftables.service` starts, restarts, or reloads. Debian's `nftables.conf` flushes every table, the tool's included.
	- The file deletes the old table and adds the new one in a single nft transaction, so there's no gap without rules.

- Outbound, with `useTor` on:
	- A new connection from the service account to anything but loopback gets dropped and counted. A rate-limited log line names the address.
	- Replies to connections that came in, such as a LAN wallet's RPC traffic, still go out.
	- Tor runs as its own account, so its traffic isn't touched.
	- It works the same in both modes, since a host-network container's sockets belong to the service account's uid.

- Inbound, new connections from anywhere but loopback get dropped:
	- On the RPC port, unless they come from an address in `rpcAllowFrom`.
	- On `anonInboundPort` and `rpcOnionPort`, always. Tor reaches them over loopback.
	- On `p2pPort`, unless `p2pBindIP` isn't a loopback address.

- The table only drops traffic. A drop in another table still applies, so ufw or firewalld also has to let the wallet's host in. `repair` checks for taht rule, and prints the command when it's missing. The tool never changes another firewall's rules.

- A ufw reload or disable only rewrites ufw's own tables, so the tool's table stays loaded.

### Wallets

- Setup for Feather and other wallets, on the same host, on the LAN, or away from it, is in [local-network-wallets.md](local-network-wallets.md).

- The RPC binds to localhost unless a wallet on another host needs it. On a LAN address, `rpcAllowFrom` lists the hosts that may reach the RPC port. See [Firewall](#firewall).

### Boot

- With `startAtBoot` on, `setup` and `repair` install `tt-monerodctl.service` and enable it.
	- `ExecStart` runs `tt-monerodctl start --foreground`, and `ExecStop` runs `tt-monerodctl stop`.
	- `TimeoutStopSec` is a little longer than `stopTimeout`, so systemd doesn't kill `monerod` first.
	- It starts after `network-online.target`, `tor.service`, and `docker.service` when those are used.
	- With `firewall` on, it requires `tt-monerodctl-firewall.service`, so `monerod` never starts at boot without the rules.

- With `startAtBoot` off, `repair` disables and removes the unit.

### NixOS

- The repo has a flake with a NixOS module. It declares:
	- The service account and group.
	- Tor, with the p2p onion service and the optional RPC onion service.
	- Docker, in Docker mode.
	- `/etc/tt-monerodctl/config.shcl`, written from module options.
	- The firewall unit and its rules.
	- The boot unit.

- `tt-monerodctl` detects NixOS by the `/etc/NIXOS` file. There, `setup` and `repair` still handle the data tree, the `monerod` download, and `bitmonero.conf`. For the rest they only check, and print the module options to set.

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

- In Docker mode `monerod` can't see anything outside its binaries and data. In native mode it's limited by file modes only.

- The config doesn't have any secrets in it. The RPC login is in `bitmonero.conf`, with mode 0660.

- The RPC binds to localhost unless a wallet on another host needs it. On a LAN address, the firewall lets only the hosts in `rpcAllowFrom` reach it.

- With Tor and the firewall on, a leak from `monerod` gets dropped and logged, rather than reaching the LAN or the internet.

- Every download is checked before use. `monerod` releases need a good signature on the hash list and a matching hash. Tool releases need a matching SHA-256 sum.

- With Tor on, the tool and `monerod` don't make any clearnet connections. That includes DNS, since `--proxy` blocks DNS outside the proxy, and update checks are off.

### Testing

- Unit tests run with the fake command runner. They check the exact command lists for each mode, with Tor on and off, and the config errors. They also check the rendered `firewall.nft` for each mix of `useTor`, `rpcAllowFrom`, and bind addresses.

- A smoke test runs `monerod --offline` in the container against an empty data dir. It checks the hardening flags, the log, and a clean exit on `docker stop`.

- A live start and stop on a real node checks ownership, listening ports, and that every outbound connection goes to the Tor SOCKS port.

- With the firewall on, a connection from the service account to a LAN address fails and bumps the drop counter. A wallet in `rpcAllowFrom` still syncs.

## Needs more thought

- Stopping `nftables.service` flushes every table, the tool's included, and the table stays gone until the next start. `monerod` keeps running without its rules in the meantime.

- Tool releases could be signed, beyond a SHA-256 sum.

- `memCap` of 8 GB is a guess. LMDB's mapped pages count against the cap, so sync may slow down under it.

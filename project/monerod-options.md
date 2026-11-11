<!-- markdownlint-disable MD007 -- Unordered list indentation -->
<!-- markdownlint-disable MD010 -- No hard tabs -->
<!-- markdownlint-disable MD055 -- Table pipe style [Expected: leading_and_trailing; Actual: leading_only; Missing trailing pipe] -->
<!-- markdownlint-disable MD041 -- First line in a file should be a top-level heading -->

<!-- TOC ignore:true -->
# monerod options

This doc lists the `monerod` options that `tt-monerod` passes and why, plus the ones that were tried and dropped. They were checked against `monerod` v0.18.4.0, with its `--help` and the `ANONYMITY_NETWORKS.md` that comes in the release archive.

<!-- TOC ignore:true -->
## Table of contents

<!-- TOC -->

- [What the options are for](#what-the-options-are-for)
- [Tor wiring](#tor-wiring)
	- [Zones](#zones)
	- [The port has to match](#the-port-has-to-match)
	- [Why there's no onion port for the clearnet zone](#why-theres-no-onion-port-for-the-clearnet-zone)
- [Options passed](#options-passed)
- [Options not passed](#options-not-passed)
- [Wallets](#wallets)
- [ZMQ](#zmq)

<!-- /TOC -->

## What the options are for

- They set up a pruned node that wallets on the same host or LAN can use.

- It leaks as little as it can about the host. It syncs over Tor, never accepts clearnet inbound connections, and accepts inbound connections over Tor only.

## Tor wiring

### Zones

- `monerod` sorts peers into separate zones. The public zone is the normal p2p network, which `--proxy` can route over Tor. The Tor zone is a second network of onion peers, and it's set up with `--tx-proxy` and `--anonymous-inbound`.

- Per `ANONYMITY_NETWORKS.md`, the Tor zone only handles handshakes, timed syncs, and transaction broadcasts. It doesn't do block sync. So inbound Tor peers help relay transactions, but they don't sync blocks from this node.

- The node sends its own transactions out through the Tor zone. That hides which node a transaction started from.

### The port has to match

- `--anonymous-inbound <onion>:<port>,<bind ip>:<port>,<max>` tells `monerod` 2 things. It advertises `<onion>:<port>` to Tor zone peers, and it listens on `<bind ip>:<port>`.

- Tor has to forward that same onion port to that same local port, with a `HiddenServicePort` line.

- A common mistake is advertising one port while Tor forwards a different one. Peers that dial the advertised address then reach whatever listens behind the forwarded port. If that's the public p2p port, they get treated as public peers from localhost, and the Tor zone listener never gets a connection.

- `tt-monerod` uses one number, `anonInboundPort`, for the onion port, the forwarded port, and the listener.

### Why there's no onion port for the clearnet zone

- Forwarding an onion port to the public p2p port looks like a way to serve blocks over Tor. In practice, peers don't find it.
	- `monerod` advertises an onion address only to the Tor zone, and only with the port from `--anonymous-inbound`.
	- `--hide-my-port` stops it from advertising itself in the public zone at all.

- So the only peers that would use it are ones that got a wrong address, per [The port has to match](#the-port-has-to-match). That mixes the 2 zones, which `--tx-proxy` is meant to keep apart.

- Dropping it also leaves one less way in.

## Options passed

| Option                                                   | Value or condition       | Why
| :---                                                     | :---                     | :---
| `--non-interactive`                                      | not with `--debug`       | There's no console in a detached run.
| `--db-sync-mode`                                         | `safe`                   | Slower, but the database survives a kill. The container is the first thing killed when memory runs out.
| `--prune-blockchain`                                     |                          | Uses about a third of the disk space.
| `--sync-pruned-blocks`                                   |                          | Faster sync for a pruned node.
| `--pad-transactions`                                     |                          | Pads relayed transactions against traffic analysis.
| `--restricted-rpc`                                       |                          | Needed with the RPC on a LAN address. Wallets work with it.
| `--no-igd`                                               |                          | Turns off UPnP, so the router never opens a port. It has nothing to do with gray peer lists.
| `--hide-my-port`                                         |                          | Don't advertise this node in the public zone.
| `--block-sync-size`                                      | 20 with Tor, 100 without | Smaller batches are gentler on a Tor circuit.
| `--max-concurrency`                                      | `maxConcurrency`
| `--data-dir`                                             | `dataDir`
| `--config-file`                                          | `bitmonero.conf`         | Has the log level and the RPC login.
| `--log-file`                                             | `logs/monerod.log`
| `--max-log-files`                                        | `10`                     | Each one is up to 100 MB by default.
| `--ban-list`                                             | `ban_list.txt`           | Fetched over Tor before each start, and checked first.
| `--rpc-bind-ip`, `--rpc-bind-port`                       | `rpcBindIP`, `rpcPort`
| `--rpc-restricted-bind-ip`, `--rpc-restricted-bind-port` | `rpcOnion` on            | A second, restricted RPC listener on localhost, for the RPC onion service.
| `--confirm-external-bind`                                | RPC not on loopback      | `monerod` refuses a LAN address without it.
| `--p2p-bind-ip`                                          | `127.0.0.1`              | Nothing on the LAN or the internet can reach the public p2p port.
| `--p2p-bind-port`                                        | `p2pPort`
| `--out-peers`                                            | 16 with Tor, 32 without
| `--in-peers`                                             | `0`                      | Inbound peers come through the Tor zone, which has its own limit.
| `--check-updates`                                        | `disabled`               | The default does a DNS lookup.
| `--no-zmq`                                               | unless `zmq` is on       | See [ZMQ](#zmq).
| `--proxy`                                                | Tor only                 | All public zone p2p over Tor. It also blocks DNS outside the proxy, unless `--proxy-allow-dns-leaks` is given.
| `--tx-proxy`                                             | Tor only                 | This node's own transactions go out over Tor. Noise stays on.
| `--anonymous-inbound`                                    | Tor and inbound          | See [The port has to match](#the-port-has-to-match).
| `--disable-dns-checkpoints`                              | Tor only                 | No DNS outside Tor.
| `--enforce-dns-checkpointing`                            | without Tor              | Makes sure the node follows the right chain.

- `nice` and `umask` aren't `monerod` options. The launcher sets them before `monerod` starts. `ionice` was left out, since ZFS ignores it.

## Options not passed

| Option                               | Why not
| :---                                 | :---
| `--rpc-login`                        | It would show in `ps` and in the container runtime's `inspect`. It goes in `bitmonero.conf` instead.
| `--rpc-payment-allow-free-loopback`  | Does nothing unless `--rpc-payment-address` is set.
| `--public-node`                      | Advertises the RPC to everyone.
| `--enable-dns-blocklist`             | DNS lookups outside Tor. `--ban-list` does the same job.
| `--detach`                           | The container runtime or tmux handles that.
| `--p2p-use-ipv6`                     | Off by default, and it's left off.
| `--no-gd`                            | Only connects to whitelisted peers. It slows sync, and it can hide the node from Tor peers.
| `--add-exclusive-node` with an onion | With only onion peers, the node can't sync blocks, per `ANONYMITY_NETWORKS.md`.

- Old forms that no longer work:
	- `--proxy=127.0.0.1:9050,tor`. Drop the `,tor`.
	- `--anonymous-inbound=<onion>:18080` without a bind address, and `--anonymous-inbound=127.0.0.1:18080,<onion>:18080` with the 2 halves swapped.

## Wallets

- A wallet such as Feather connects to the RPC port with the login from `bitmonero.conf`. `--restricted-rpc` still allows what a wallet needs, which is syncing and sending.

- A transaction that a wallet sends through the RPC didn't come from a p2p peer. With `--tx-proxy` on, `monerod` sends a transaction like that only to Tor zone peers, per `ANONYMITY_NETWORKS.md`. So the wallet's sends go out over Tor even when the wallet talks to the node over the LAN.

- Without any Tor zone peers connected, `monerod` queues the transaction and logs an error. It sends it once a Tor zone peer connects, or after a restart with Tor off.

- Tor can't reach a LAN address. A wallet that sends everything over Tor needs an exception for local nodes, or an onion address for the RPC.

- The hop between the wallet and the node isn't encrypted unless the wallet connects with SSL. `monerod` accepts both, since `--rpc-ssl` defaults to `autodetect`.

## ZMQ

- ZMQ is ZeroMQ, a message queue library. `monerod` uses it to push notices about new blocks and transactions to other programs on the same host, on port 18082.

- P2Pool, some block explorers, and some payment processors use it. Wallets don't.

- It's off by default here, with `--no-zmq`, since that's one less open port. `zmq = true` in the config turns it back on.

<!-- markdownlint-disable MD007 -- Unordered list indentation -->
<!-- markdownlint-disable MD010 -- No hard tabs -->
<!-- markdownlint-disable MD055 -- Table pipe style [Expected: leading_and_trailing; Actual: leading_only; Missing trailing pipe] -->
<!-- markdownlint-disable MD041 -- First line in a file should be a top-level heading -->

<!-- TOC ignore:true -->
# Local network wallets

This doc explains how to point a wallet at the node, from the same host, from another host on the LAN, or from away. Feather is the example, but other wallets have the same settings under their own names.

<!-- TOC ignore:true -->
## Table of contents

<!-- TOC -->

- [How it works](#how-it-works)
- [Node settings](#node-settings)
- [Feather](#feather)
	- [Wallet on the same host](#wallet-on-the-same-host)
	- [Wallet on another LAN host](#wallet-on-another-lan-host)
	- [Wallet away from the LAN](#wallet-away-from-the-lan)
	- [Proxy mode](#proxy-mode)
- [Other wallets](#other-wallets)
- [When the node is down](#when-the-node-is-down)
- [Troubleshooting](#troubleshooting)

<!-- /TOC -->

## How it works

- The wallet talks to the node's RPC port. It syncs balances from it, and sends transactions through it.

- The RPC is restricted, and it needs the login from `bitmonero.conf`. A restricted RPC still allows everything a wallet needs.

- A transaction sent through the RPC goes out only to the node's Tor zone peers. So sends leave over Tor, even when the wallet talks to the node directly.

- Without any Tor zone peers connected, `monerod` queues the transaction and sends it once one connects. `tt-monerodctl status` warns when that happens.

## Node settings

| Wallet runs on   | `rpcBindIP`            | `rpcAllowFrom`          | `rpcOnion`
| :---             | :---                   | :---                    | :---
| The same host    | `127.0.0.1`            | none                    | `false`
| Another LAN host | the node's LAN address | the wallet's address    | `false`
| Anywhere         | either of the above    | as above                | `true`

- After a change, run `tt-monerodctl repair` and restart the node.

- On a LAN address, only the wallet's host should be able to reach the RPC port, since RPC traffic on the LAN isn't encrypted. With `firewall` on, the tool's own rules drop RPC connections from any address that isn't in `rpcAllowFrom`.

- When ufw or firewalld is active, it also has to let the wallet's host in. `tt-monerodctl repair` prints the command when that rule is missing. With ufw, for example:

	~~~bash
	sudo ufw allow from 192.168.1.50 to any port 18081 proto tcp
	~~~

- The RPC username and password are on the `rpc-login=` line of `bitmonero.conf`, in the data dir. `setup` prints them once, when it makes them.

## Feather

Add the node under Settings, Network, Node. Turn off "Let Feather manage this list", then add a custom node.

- Feather takes the login as part of the node address, in the form `<username>:<password>@<address>:<port>`. That's the `rpc-login` value, then `@`, then the address and port.

### Wallet on the same host

- Node address: `<username>:<password>@127.0.0.1:18081`.

### Wallet on another LAN host

- Node address: the login, then the node's LAN address and RPC port, such as `<username>:<password>@192.168.1.10:18081`.

- Feather treats only 127.0.0.0/8, 172.16.0.0/12, and 192.168.0.0/16 as local. Feather tries a node on a 10.x address over Tor, and that fails, since Tor can't reach a LAN address.

### Wallet away from the LAN

- Turn on `rpcOnion` on the node. `tt-monerodctl status` shows the RPC onion address.

- Node address: the login, then the onion address and `rpcOnionPort`, such as `<username>:<password>@<address>.onion:18089`.

- Sync is slower over Tor.

### Proxy mode

- Set it to "Always over Tor", under Settings, Network, Proxy.

- Feather never sends traffic for a local node over Tor, whatever the mode. So for a node on the same host or the LAN, this setting only moves Feather's other traffic to Tor.

## Other wallets

- `monero-wallet-cli` and `monero-wallet-rpc` take `--daemon-address <host>:<port>` and `--daemon-login <username>:<password>`. For an onion address they also need `--proxy 127.0.0.1:9050`.

- The Monero GUI wallet has the same settings, as daemon address, username, and password for a remote node.

## When the node is down

- A wallet with only this node in its list can't sync while the node is down.

- Feather's managed list of onion nodes works as a fallback. Turn "Let Feather manage this list" back on to use it. A remote node then sees which outputs the wallet asks about, so switch back once the node is up.

## Troubleshooting

- Feather won't connect:
	- `tt-monerodctl status` shows which address the RPC listens on. It has to match the wallet's node address.
	- Check the username and password in the node address against `rpc-login`.
	- For a LAN node, check the address range, `rpcAllowFrom`, and the ufw or firewalld rule above.

- A send shows in the wallet but never confirms:
	- `tt-monerodctl status` shows whether a transaction is waiting for a Tor zone peer. It goes out once one connects.

<!-- markdownlint-disable MD007 -- Unordered list indentation -->
<!-- markdownlint-disable MD010 -- No hard tabs -->
<!-- markdownlint-disable MD041 -- First line in a file should be a top-level heading -->

<!-- TOC ignore:true -->
# Project backlog

<!-- TOC ignore:true -->
## Table of contents

<!-- TOC -->

- [Introduction](#introduction)
- [Issues](#issues)
- [Template](#template)
	- [New format](#new-format)

<!-- /TOC -->

## Introduction

This is a product backlog just for pre-v1.0.0 release. After that, bugs, features, and enhancements will be managed in GitHub issues.

## Issues

- Config reader for shcl in Go
	- ID: 2026111109365398
	- Type: Task
	- Status: Queued
	- Priority: High
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Requirements:
		- Read `config.shcl` from Go.
	- Decisions:
		- Config is shcl. The tool reads it with the latest published beta of shcl's Go module.

- Core commands
	- ID: 2026111109365411
	- Type: Feature
	- Status: Queued
	- Priority: High
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Prereq IDs: 2026111109365398
	- Requirements:
		- `start`, `stop`, `status`, and `watch`, per `design.md`.
		- Docker mode is the default, and native tmux mode is the fallback.
		- `--dry-run` and `--debug`.
		- `--help`, `--about`, and `--version`.
		- Hand the tree to the service account on start, and back on stop.

- repair
	- ID: 2026111109365424
	- Type: Feature
	- Status: Queued
	- Priority: High
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Prereq IDs: 2026111109365411
	- Requirements:
		- Run the checks listed in `design.md`, each with an optional fix.
		- `--check` only reports.
		- Check the firewall table and its unit, and fix them. Check that ufw or firewalld lets the `rpcAllowFrom` hosts in, and print the command when it doesn't.

- setup
	- ID: 2026111109365438
	- Type: Feature
	- Status: Queued
	- Priority: High
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Prereq IDs: 2026111109365424
	- Requirements:
		- Install from scratch, per `design.md`.
		- Make the RPC password from 32 letters and digits only, since Feather takes the login as part of the node address.
		- Install nftables when it's missing, write the firewall rules, and start their unit.
		- On an existing install, run `repair` and then `update`.

- Firewall
	- ID: 2026111110391351
	- Type: Feature
	- Status: Queued
	- Priority: High
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Prereq IDs: 2026111109365411
	- Requirements:
		- Keep the tool's rules in its own nftables table, loaded by its own systemd unit, per `design.md`.
		- With Tor on, drop new outbound connections from the service account that aren't to loopback.
		- Drop inbound connections to the RPC port from anywhere but loopback and `rpcAllowFrom`.
		- `start` refuses to run when the table isn't loaded.
		- `setup` and `repair` manage it. They never change ufw or firewalld rules.
	- Decisions:
		- fail2ban was looked at and left out, per `design.md`. The firewall already limits the RPC on a LAN address, and over Tor every wallet comes from localhost.

- update
	- ID: 2026111109365451
	- Type: Feature
	- Status: Queued
	- Priority: High
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Prereq IDs: 2026111109365411
	- Requirements:
		- Update the tool from GitHub releases.
		- Update `monerod` from getmonero.org, with the signature and hash checked.
		- Update Tor through apt.
		- Refresh the pinned container image.

- Config defaults and overrides
	- ID: 2026111110002898
	- Type: Feature
	- Status: Queued
	- Priority: High
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Prereq IDs: 2026111109365398
	- Requirements:
		- Default configuration hard-coded.
		- Overridden by the config file.
		- Resistant to errors, so one bad line doesn't sink the whole thing.
		- Overridden by program options at run-time.
	- Decisions:
		- Config is shcl, in `/etc/tt-monerod/config.shcl`, and `setup` writes it, per `design.md`. That replaces the earlier plan of a per-user YAML or TOML file under `~/.config`, which would've been made on the first changed setting.

- Start at boot
	- ID: 2026111110002911
	- Type: Feature
	- Status: Queued
	- Priority: Avg
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Prereq IDs: 2026111109365424
	- Requirements:
		- Add `startAtBoot` to the config, and a systemd unit that runs `start --foreground`, per `design.md`.
		- `start` and `stop` go through `systemctl` when the unit is enabled.
		- `setup` and `repair` install or remove the unit to match the config.

- Wallet access
	- ID: 2026111110002925
	- Type: Feature
	- Status: Queued
	- Priority: Avg
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Prereq IDs: 2026111109365411
	- Requirements:
		- Bind the RPC to localhost by default, or to a LAN address.
		- Add `rpcOnion`, with its own onion service and a restricted RPC listener on localhost, per `design.md`.
		- `status` warns when a sent transaction is waiting for a Tor zone peer.

- NixOS module
	- ID: 2026111110002938
	- Type: Feature
	- Status: Queued
	- Priority: Avg
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Prereq IDs: 2026111109365438
	- Requirements:
		- Add a flake with a NixOS module for the account, Tor, Docker, the config file, and the boot unit.
		- On NixOS, `setup`, `repair`, and `update` check the system parts and print module settings, rather than change them.

- CI/CD pipeline
	- ID: 2026111110002951
	- Type: Feature
	- Status: Queued
	- Priority: Avg
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Requirements:
		- A CI/CD pipeline kicked off by a bash script (`cicd/cicd.bash`). It builds, tests, and can commit and push. Packaging and publishing are opt-in.

- Release-install script
	- ID: 2026111110030775
	- Type: Feature
	- Status: Queued
	- Priority: Avg
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Prereq IDs: 2026111110002951
	- Requirements:
		- Runnable via a single `curl` or `wget`, and documented under "how to install".
		- Downloads, installs, and runs the latest release, with an option to abort.
	- Target OS: Debian, NixOS
	- Decisions:
		- It's Bash only, since both targets are Linux. It installs the binary, and `setup` does the rest.

- First release
	- ID: 2026111110030788
	- Type: Task
	- Status: Queued
	- Priority: Avg
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Prereq IDs: 2026111110002951, 2026111110030775
	- Requirements:
		- Publish `v1.0.0-alpha.1` on GitHub, with Linux builds for amd64 and arm64 and a checksum file that `update --self` can read.

- Move the first node over
	- ID: 2026111109365464
	- Type: Task
	- Status: Queued
	- Priority: Avg
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Prereq IDs: 2026111109365438
	- Requirements:
		- Move the existing node to the new account and tool.
		- Point Feather at the node with the RPC login in the node address, per `local-network-wallets.md`.
		- Retire the old start, stop, and stats scripts after it runs clean.

- Dev-environment install script
	- ID: 2026111110030762
	- Type: Feature
	- Status: Queued
	- Priority: Low
	- Opened: 2026-11-11
	- Opened by: t00mietum
	- Prereq IDs: 2026111110002951
	- Requirements:
		- Runnable via a single `curl` or `wget`, and documented under "how to develop".
		- Clones main, installs dependencies, and states what it will do, with an option to abort.
	- Target OS: Debian, NixOS
	- Decisions:
		- It's Bash only, since both targets are Linux. The template also listed macOS sh and Windows PowerShell.

## Template

### New format

- Notes:

	- Only use rows that you actually need or expect will be filled in. Always fill in the title, ID, Type, Status, Opened and Created by.

	- The ID is the local time to the hundredth of a second. Opened is when it was written down, which may differ. (Use a keyboard macro and possibly something like project 'zuid' to generate.)

	- Status values meaning: Testing means the fix is in and checks are running or still to run. Waiting on signoff means automated testing passed. Moot means something else changed that made it irrelevant. Canceled means it still applies but was decided against. Waiting for testing means the fix is in and waits on a long CI run or an outside test host. Can't reproduce means a real attempt to reproduce it failed.

	- As issues are worked, and statuses change, place them in correct sorting order within the list:
		- First by status: Waiting for answers, Waiting on signoff, Testing, Waiting for testing, Can't reproduce, Stalled, Started, Queued, Done, Deferred, Canceled, Moot
		- Then by severity|priority: Critical, High, Avg, Low
		- Then by type: Bugs, [not bugs together]

	- Rows marked [Bug] are for bugs only, and rows marked [Feature] for features and enhancements. Priority and Severity share one row and one scale. Priority is for a Feature or Enhancement, and Severity for a Bug. Children are not nested. They sit at the top level and point back with Parent ID.

Template:

- Title
	- ID: YYYYmmDDHHMMSSNN
	- Type: [Bug|Feature|Enhancement|Task]
	- Status: [Queued|Waiting for answers|Waiting on signoff|Waiting for testing|Started|Testing|Stalled|Can't reproduce|Moot|Canceled|Deferred|Done]
	- Needs local test suite run?:
	- Needs external testing:
	- Priority [Feature|Enhancement] | Severity [Bug]: [Critical|High|Avg|Low]
	- Opened:
	- Opened by:
	- Assigned to:
	- Parent ID:
	- Prereq IDs:
	- Related IDs:
	- Target OS:
	- Test environment:
	- Version and build:
	- Requirements  [Feature]:
		- Hierarchical bulleted list.
	- Steps to reproduce [Bug]:
		- …
	- Incorrect behavior [Bug]:
	- Expected behavior [Bug]:
	- Reproduced [Bug]: [No, or when, where and how]
	- Possible cause [Bug]:
	- Actual cause [Bug]:
		- …
	- Estimated effort: [High|Avg|Low]
	- Actual effort: [High|Avg|Low]
	- Progress log:
		- …
	- Decisions:
		- …
	- Actual fix [Bug]:
	- Branch:
	- Commit:
	- Test case: [Reason not applicable, or CI test case #]
	- Acceptance signoff:
	- Superseded by ID:
	- Closed:

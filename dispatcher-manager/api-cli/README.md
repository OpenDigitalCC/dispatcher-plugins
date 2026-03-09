---
title: dispatcher-manager cli
subtitle: Perl CLI client for the dispatcher HTTP API
brand: odcc
---

# cli

A monolithic Perl script providing full CLI access to the dispatcher HTTP API.
Covers all endpoints: health, ping, run, and discovery. Intended as a reference
implementation and a portable tool for testing API functions before building
higher-level interfaces.

No CPAN dependencies. Uses only modules available as Debian trixie system
packages.

## Purpose

Exposes the complete dispatcher HTTP API as a command-line interface with
human-readable table output by default and raw JSON output on request. Useful
for scripted automation, ad-hoc operations, and as a reference for how each
endpoint behaves before implementing a UI or integration.

## Dependencies

| Module | Debian package | Notes |
| --- | --- | --- |
| `HTTP::Tiny` | `perl` (core) | HTTP client |
| `JSON::PP` | `perl` (core) | JSON encode/decode |
| `Getopt::Long` | `perl` (core) | Argument parsing |

All dependencies are included in Perl's core distribution and require no
additional installation on a standard Debian trixie system.

## Installation

Copy the script to any location on the dispatcher host or any host with
network access to the API port:

```bash
sudo cp dispatcher-manager /usr/local/bin/
sudo chmod 755 /usr/local/bin/dispatcher-manager
```

Verify:

```bash
dispatcher-manager --version
dispatcher-manager health
```

## Configuration

The base URL, token, and username are resolved in priority order:
flag > environment variable > config file > default.

Config file (`~/.dispatcher-manager.conf`):

```ini
url      = http://myhost:7445
token    = mytoken
username = alice
```

Environment variables:

```
DISPATCHER_API_URL    API base URL (default: http://localhost:7445)
DISPATCHER_TOKEN      Auth token
DISPATCHER_USERNAME   Username (default: $USER)
```

Global flags (override all other sources):

```
--url <url>        API base URL
--token <token>    Auth token
--username <name>  Username
```

Use `DISPATCHER_TOKEN` rather than `--token` to prevent the token value
appearing in `ps` output.

## Subcommands

### health

Check API server liveness and version. No authentication required.

```bash
dispatcher-manager health
dispatcher-manager health --json
```

### ping

Test mTLS connectivity to one or more agents. Reports cert expiry and agent
version per host.

```bash
dispatcher-manager ping <host> [host...] [--json]
```

`<host>` accepts `hostname` or `hostname:port` (port defaults to 7443).

### run

Run an allowlisted script on one or more agents in parallel.

```bash
dispatcher-manager run <host> [host...] --script <name> [--json] [-- arg...]
```

`--script` is required. Everything after `--` is passed to the script as
positional arguments. stdout and stderr from the script are printed to stdout
and stderr respectively. Exit code is 0 if all hosts succeeded, 1 if any
host failed.

### discovery

List registered agents and their allowlisted scripts.

```bash
dispatcher-manager discovery [host...] [--scripts] [--json]
```

Without `--scripts`, outputs a summary table (host, status, version, script
count). With `--scripts`, outputs one block per host listing each script name
and path. Optional host arguments filter the results to specific agents.

## Examples

```bash
# Check the API is reachable
dispatcher-manager health

# Ping all agents (uses discovery to find hosts, then pings)
dispatcher-manager ping web-01 web-02 db-01

# Run a script on multiple hosts
dispatcher-manager run web-01 web-02 --script deploy

# Run with script arguments
dispatcher-manager run db-01 --script backup-mysql -- --database myapp

# Discovery summary
dispatcher-manager discovery

# Discovery with script list
dispatcher-manager discovery --scripts

# Filter discovery to specific hosts
dispatcher-manager discovery --scripts web-01 db-01

# JSON output for scripting
dispatcher-manager discovery --json | python3 -m json.tool

# Override URL and token inline
dispatcher-manager --url http://myhost:7445 --token mytoken ping web-01

# Use environment variables
export DISPATCHER_API_URL=http://myhost:7445
export DISPATCHER_TOKEN=mytoken
dispatcher-manager ping web-01
dispatcher-manager discovery --scripts
dispatcher-manager run web-01 --script check-disk --json
```

## Limitations

- HTTPS is supported (HTTP::Tiny supports TLS via IO::Socket::SSL), but the
  CA cert is not verified by default if IO::Socket::SSL is not present.
  Install `libio-socket-ssl-perl` for full TLS verification.
- The script does not implement the async `/status/{reqid}` poll pattern.
  Long-running scripts will block until completion or until the 120-second
  client timeout is reached. Adjust `timeout` in `_make_ua` for longer jobs.
- There is no retry logic. Connection errors and timeouts are reported and
  exit 1.
- `discovery` without a host filter uses `GET /discovery`, which queries all
  registered agents. On large fleets this may be slow.

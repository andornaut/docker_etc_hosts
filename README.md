# docker_etc_hosts

[![CI](https://github.com/andornaut/docker_etc_hosts/actions/workflows/test.yml/badge.svg)](https://github.com/andornaut/docker_etc_hosts/actions/workflows/test.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A lightweight bash script for Linux to add/update IP->hostname mappings in `/etc/hosts` for running Docker containers.

The script queries Docker for running containers and updates `/etc/hosts` to map each container's private IP address to its container name as hostname, which makes it easy to access containers by name from the host system.

Managed entries are wrapped in `# docker_etc_hosts BEGIN` / `# docker_etc_hosts END` markers. Every run rewrites the whole block, so entries for containers that are no longer running are removed. Nothing outside the markers is ever touched.

The new contents are built in full before the hosts file is written, and a run that wouldn't change anything doesn't write at all. The file is overwritten in place rather than renamed over, so that it keeps its inode: `/etc/hosts` is often a symlink, hard linked, or bind mounted into a container, and each of those would go on reading the replaced inode and never see another update. The cost of keeping it is a brief window during the rewrite in which the file is shorter than it should be.

If the markers are unbalanced or duplicated — for example after a hand-edit — the script reports the offending line and exits without writing anything, rather than guessing at which entries it's allowed to delete.

A container that can't be mapped cleanly is reported and skipped, so that one unusual container doesn't stop every other one from updating. That covers containers with no IP (host networking), containers whose name yields no valid hostname, and names that collide once normalized. A container attached to several networks uses the first of its addresses, since a hosts file maps a name to a single address.

For example, if you have running containers named `web_server` and `database`, then this script will add the following to `/etc/hosts`:

```text
# docker_etc_hosts BEGIN
172.17.0.2 database.internal
172.17.0.3 web-server.internal
# docker_etc_hosts END
```

## Requirements

Linux, Docker, and bash 4+.

macOS is explicitly unsupported, and the script exits rather than run there. Under Docker Desktop's default networking, container IPs aren't routable from the host, so the entries would name addresses nothing can reach. Runtimes that do route them — OrbStack, or Docker Desktop plus a tunnel such as [docker-mac-net-connect](https://github.com/chipmk/docker-mac-net-connect) — would additionally need a launchd daemon in place of the systemd unit and BSD-compatible tooling throughout. That isn't a trade this project makes.

## Installation

Download the script, make it executable, and (optionally) install it system-wide:

```bash
curl -O https://raw.githubusercontent.com/andornaut/docker_etc_hosts/main/docker_etc_hosts
chmod +x docker_etc_hosts
sudo cp docker_etc_hosts /usr/local/bin/
```

### Installing the systemd service

To automatically update `/etc/hosts` whenever a Docker container starts or stops, install the [provided systemd unit file](./docker-etc-hosts.service), which runs `docker_etc_hosts --watch`:

```bash
# Download the service file
curl -O https://raw.githubusercontent.com/andornaut/docker_etc_hosts/main/docker-etc-hosts.service
sudo cp docker-etc-hosts.service /etc/systemd/system/

# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable docker-etc-hosts.service
sudo systemctl start docker-etc-hosts.service
```

## Usage

```bash
# Update `/etc/hosts` with running Docker containers IPs and names:
sudo docker_etc_hosts

# Keep running, updating on every container start or stop:
sudo docker_etc_hosts --watch

# Preview the resulting hosts file without modifying anything:
docker_etc_hosts --dry-run

# Use a custom domain:
docker_etc_hosts --dry-run --domain local

# Use a custom hosts file:
docker_etc_hosts --hosts-file /tmp/hosts

# Show help:
docker_etc_hosts --help
```

```text
$ ./docker_etc_hosts --help
Update /etc/hosts with running Docker containers IPs and names

Usage: ./docker_etc_hosts [OPTIONS]

OPTIONS:
  --domain DOMAIN            Domain for containers (default: internal)
  --dry-run                  Write the resulting hosts file to stdout instead
  --hosts-file FILE          Target hosts file (default: /etc/hosts)
  --watch                    Keep running, updating on every container event
  -h, --help                 Show this help message

Container names are formatted as: container-name.DOMAIN

The managed entries are the ones between the '# docker_etc_hosts BEGIN' and
'# docker_etc_hosts END' markers, and every run rewrites all of them, so entries
for containers that are no longer running are removed.

--clean is accepted and ignored: it named the behaviour that is now the only one.
```

## Upgrading

Rewriting the managed block is now the only behaviour, so `--clean` no longer does anything. It's still accepted, and service files installed by earlier versions keep working — though the unit above is now a plain `--watch` invocation and is worth reinstalling.

Versions that predate the marker comments wrote entries directly into `/etc/hosts`, outside any block. Those are left alone, and because the first match in a hosts file wins, one sitting ahead of the managed block will mask the entry this script maintains. If a container resolves to a stale address, look for a line above the `# docker_etc_hosts BEGIN` marker and delete it:

```bash
grep -n '\.internal' /etc/hosts
```

## Development

[ShellCheck](https://www.shellcheck.net/) lints the scripts, and `test_docker_etc_hosts` exercises `docker_etc_hosts` against synthetic hosts files with a stubbed `docker` command. Both run in CI:

```bash
shellcheck docker_etc_hosts test_docker_etc_hosts
./test_docker_etc_hosts
```

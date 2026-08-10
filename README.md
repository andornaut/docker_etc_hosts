# docker_etc_hosts

A lightweight bash script for Linux and macOS to add/update IP->hostname mappings in `/etc/hosts` for running Docker containers.

The script queries Docker for running containers and updates `/etc/hosts` to map each container's private IP address to its container name as hostname, which makes it easy to access containers by name from the host system. Managed entries are wrapped in `# docker_etc_hosts BEGIN` / `# docker_etc_hosts END` markers. Existing entries are updated in-place, and new entries are appended within the managed block.

Only the managed block is ever rewritten. Where the hosts file is a standalone plain file, it's replaced by a rename, so it stays complete and resolvable even if the script is interrupted; where a rename would destroy something — a symlink, a hard-linked file, or a bind-mounted `/etc/hosts` such as the one inside every container — the file is overwritten in place instead, preserving its identity at the cost of that guarantee.

If the markers are unbalanced or duplicated — for example after a hand-edit — the script reports the offending line and exits without writing anything, rather than guessing at which entries it's allowed to delete. It also warns about entries that map a managed hostname ahead of the block, such as those written by versions of this script that predate the markers, because the first match in a hosts file wins and those entries would mask the managed one.

For example, if you have running containers named `web_server` and `database`, then this script will add the following to `/etc/hosts`:

```text
# docker_etc_hosts BEGIN
172.17.0.2 database.internal
172.17.0.3 web-server.internal
# docker_etc_hosts END
```

## Installation

Download the script, make it executable, and (optionally) install it system-wide:

```bash
curl -O https://raw.githubusercontent.com/andornaut/docker_etc_hosts/main/docker_etc_hosts
chmod +x docker_etc_hosts
sudo cp docker_etc_hosts /usr/local/bin/
```

### Installing the systemd service

To automatically update `/etc/hosts` whenever a Docker container starts or stops, install the [provided systemd unit file](./docker-etc-hosts.service):

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

# Replace all managed entries (removes stale hosts from stopped containers):
sudo docker_etc_hosts --clean

# Preview changes without modifying the hosts file:
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
  --clean                    Replace all managed entries instead of updating in-place
  --domain DOMAIN            Domain for containers (default: internal)
  --dry-run                  Write to stdout instead of updating a hosts file
  --hosts-file FILE          Target hosts file (default: /etc/hosts)
  -h, --help                 Show this help message

Container names are formatted as: container-name.DOMAIN
```

## Development

[ShellCheck](https://www.shellcheck.net/) lints the scripts, and `test_docker_etc_hosts` exercises `docker_etc_hosts` against synthetic hosts files with a stubbed `docker` command. Both run in CI:

```bash
shellcheck docker_etc_hosts test_docker_etc_hosts
./test_docker_etc_hosts
```

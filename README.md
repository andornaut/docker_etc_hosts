# docker_etc_hosts

A lightweight bash script that manages `/etc/hosts` entries for running Docker containers.

The script queries Docker for running containers and adds host entries with a `.internal` domain suffix, making it easy to access containers by name from the host system.

For example, if you have running containers named `web_server` and `database`,
then this script will add the following to `/etc/hosts`:

```text
# DOCKER_ETC_HOSTS_START
172.17.0.2 web-server.internal
172.17.0.3 database.internal
# DOCKER_ETC_HOSTS_END
```

## Installation

Download the script, make it executable, and (optionally) install it system-wide:

```bash
curl -O https://raw.githubusercontent.com/andornaut/docker_etc_hosts/main/docker_etc_hosts
chmod +x docker_etc_hosts
sudo cp docker_etc_hosts /usr/local/bin/
```

### Installing the systemd service

To automatically update `/etc/hosts` when Docker starts, install the provided systemd service:

```bash
# Copy the service file
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

# Preview changes without modifying the hosts file:
docker_etc_hosts --dry-run

# Show help:
docker_etc_hosts --help
```

## Configuration

The script can be configured via environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `HOSTS_FILE` | `/etc/hosts` | Path to the hosts file |
| `DOMAIN_SUFFIX` | `.internal` | Suffix for container domain names |

Example with custom configuration:

```bash
HOSTS_FILE=/tmp/hosts DOMAIN_SUFFIX=.local sudo -E docker_etc_hosts
```

# SunCore Server Hosting (k3s)

Modular game server hosting on a single machine, running on [k3s](https://k3s.io).
Every game server is described by Kubernetes manifests, so you can start it, stop it,
or add another instance without touching the others.

> **Status:** a Valheim server is working. Minecraft, Factorio and control through a
> Telegram bot are planned (see [Roadmap](#roadmap)).

## What is implemented

| Component | Status |
|---|---|
| Valheim: StatefulSet, volumes, startup probe, `status.json` endpoint | working |
| Service and a readiness probe based on the contents of `status.json` | planned |
| Minecraft and Factorio | planned |
| Telegram bot: start/stop, auto-shutdown on idle, backups | planned |
| Access from the internet | depends on your network, see [Network and access](#network-and-access) |

## Requirements

- A Linux machine (tested on Debian 13).
- RAM: an idle Valheim server (no players) uses about 1.5 GiB (measured with `kubectl top`)
  and more under load. The manifest requests 3 GiB and limits the container to 5 GiB.
  k3s itself uses about 1 GiB.
- Disk: 5 GiB for each of the two Valheim volumes (config with worlds, and server files).
- Outbound access to Steam and `ghcr.io` (the game server and the container image are downloaded).
- `kubectl` (installed together with k3s).

## Quick start

### 1. Install k3s

Create the k3s config before installing (Traefik is not needed, game traffic is not HTTP).
The `fail-swap-on=false` line is only required if swap is enabled on the machine.

```bash
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<'EOF'
disable:
  - traefik
kubelet-arg:
  - "fail-swap-on=false"
EOF

curl -sfL https://get.k3s.io | sh -
```

Use the cluster without `sudo`:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG=$HOME/.kube/config   # add this line to ~/.bashrc
kubectl get nodes                       # the node should be Ready
```

If NetworkManager runs on the machine (for example on a desktop Debian install),
stop it from managing the cluster's network interfaces:

```bash
sudo tee /etc/NetworkManager/conf.d/k3s.conf > /dev/null <<'EOF'
[keyfile]
unmanaged-devices=interface-name:cni*;interface-name:flannel*
EOF
sudo systemctl reload NetworkManager
```

### 2. Clone the repository

```bash
git clone https://github.com/Pakqot/suncore-k3s-game-server.git
cd suncore-k3s-game-server
```

### 3. Create the namespace and the secret

The server password is not stored in the repository, it is created directly in the cluster.
Valheim requirements: at least 5 characters, and it must not contain the server or world name.

```bash
kubectl apply -f valheim/00-namespace.yaml

read -rsp "Valheim password: " VPASS; echo
kubectl -n valheim create secret generic valheim-main --from-literal=SERVER_PASS="$VPASS"
unset VPASS
```

### 4. Configure the server

Open `valheim/10-statefulset.yaml` and adjust the environment variables if needed
(`SERVER_NAME`, `WORLD_NAME`, `TZ`, `SERVER_PUBLIC`), see [Configuration](#configuration).

### 5. Start it

```bash
kubectl apply -f valheim/10-statefulset.yaml
kubectl -n valheim get pods -w
kubectl -n valheim logs -f valheim-main-0
```

The first start takes 5-10 minutes: the container downloads the Valheim server from Steam
(about 1 GB) and then generates the world. The pod becomes `1/1 Running` once the startup
probe passes.

### 6. Check the status

```bash
kubectl -n valheim exec valheim-main-0 -- curl -s localhost/status.json
kubectl top pod -n valheim
```

The response should contain `"error": null`, the server name and `"player_count"`.

### 7. Connect

In Valheim: **Join Game → Join IP**, enter `MACHINE_IP:2456` (from the local network),
then the password.

## Operations

```bash
# stop the server (volumes and the world are kept)
kubectl -n valheim scale statefulset valheim-main --replicas=0

# start the server
kubectl -n valheim scale statefulset valheim-main --replicas=1

# follow the logs
kubectl -n valheim logs -f valheim-main-0

# restart the pod (volumes are kept)
kubectl -n valheim delete pod valheim-main-0
```

Running `kubectl apply -f valheim/10-statefulset.yaml` again resets everything to the values
in the file, including `replicas`.

## Configuration

Settings are environment variables in `valheim/10-statefulset.yaml`.
For the full list of image variables see the README of the
[`community-valheim-tools/valheim-server`](https://github.com/community-valheim-tools/valheim-server-docker) image.

| Variable | Purpose |
|---|---|
| `SERVER_NAME` | Server name |
| `WORLD_NAME` | World name (must not contain the password) |
| `SERVER_PASS` | Password, read from the Kubernetes Secret `valheim-main` |
| `SERVER_PUBLIC` | `"true"` lists the server in the in-game browser, `"false"` hides it |
| `STATUS_HTTP` | Enables `status.json` (used by probes and player counting) |
| `UPDATE_CRON` | An empty string disables self-updates while running, so the server never restarts in the middle of a game |
| `RESTART_CRON` | An empty string disables the scheduled nightly restart |
| `BACKUPS` | The image's built-in backups, disabled (own backups are planned) |
| `TZ` | Time zone used in logs |

The image version is pinned with a tag so that updates are deliberate: change the tag in the
manifest and run `kubectl apply`. The game server itself is updated from Steam every time the
container starts.

## Network and access

Valheim UDP ports 2456-2458 are published on the node with `hostPort` while the server is running.
Ports in use are tracked in [`docs/ports.md`](docs/ports.md).

Connecting from the internet requires a public address:

- Check your router's WAN address. If it is in `100.64.0.0/10` (or `10.x`, `172.16-31.x`,
  `192.168.x`), your ISP uses CGNAT and port forwarding on your router will not work.
- Options behind CGNAT: a public IP from your ISP, a VPN network (Tailscale, ZeroTier),
  or a VPS with a WireGuard tunnel that forwards the game ports to your machine.
- Never expose SSH or the cluster API port (6443) to the internet. Forward only the game ports.

## Repository layout

```
.
├── README.md
├── docs/
│   └── ports.md             # table of ports in use
└── valheim/
    ├── 00-namespace.yaml    # namespace valheim
    └── 10-statefulset.yaml  # server, volumes, ports, probes
```

The `game`, `instance` and `gameserver` labels on the resources are there for the future bot:
it will discover servers by these labels instead of keeping a list in code.

## Troubleshooting

| Symptom | What to do |
|---|---|
| `Failed to install app '896660' (Missing configuration)` in the logs on first start | A steamcmd download failure. Restart the pod (`kubectl -n valheim delete pod valheim-main-0`), the volumes are kept |
| Pod is in `CreateContainerConfigError` | The Secret `valheim-main` has not been created (step 3) |
| Pod is `Pending`, or a port is already in use | Another server uses the port on the node, check `docs/ports.md` and `sudo ss -ulpn` |
| Server refuses to start because of the password | The password is shorter than 5 characters or contains the server or world name |

## Security

- Secrets (server password, bot token) are never stored in the repository. `.gitignore`
  contains `*.secret.yaml` and `.env`.
- Use a long password, especially if the server is reachable from the internet.

## Roadmap

- Service and a readiness probe based on the contents of `status.json`
- Minecraft and Factorio servers using the same pattern
- Telegram bot: start/stop, auto-shutdown on idle with a configurable timeout, backup management
- World backups to external storage
- External access (public IP or a tunnel through a VPS)

# Better MC Server

A self-hosted **Better MC [NEOFORGE] BMC5** Minecraft server running on an Oracle Cloud Always Free ARM instance.

This repository tracks the whole server.

---

## Overview

| | |
|---|---|
| **Modpack** | Better MC [NEOFORGE] BMC5 (Server Pack v50) |
| **Minecraft** | 1.21.1 |
| **Loader** | NeoForge 21.1.228 |
| **Java** | 21 (bundled/auto-installed by the server pack) |
| **Host** | Oracle Cloud — Ampere A1.Flex (ARM / aarch64), Ubuntu |
| **Resources** | 2 OCPU, ~12 GB RAM |
| **Heap** | `-Xmx8G -Xms8G` |
| **Port** | 25565 (default) |

---

## How players connect

Connect in the Minecraft client (with the matching Better MC BMC5 pack installed) to:

```
YOUR_SUBDOMAIN.duckdns.org
```

No port needed — the server runs on the default 25565. Everyone must be on the **same pack version** to join.

---

## Hosting notes

### Networking
Two firewalls both need port 25565 open:
- **Oracle Cloud** — an ingress rule for TCP 25565 in the subnet's security list (alongside the default SSH/ICMP rules).
- **The instance itself** — an `iptables` rule, saved with `netfilter-persistent`:
  ```bash
  sudo iptables -I INPUT -p tcp --dport 25565 -j ACCEPT
  sudo netfilter-persistent save
  ```

### Dynamic DNS
The public IP can change if the instance is stopped/started, so DuckDNS keeps the domain pointed at the current IP. A cron job re-asserts it every 5 minutes:
```
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
```
The update script leaves `ip=` blank so DuckDNS auto-detects the instance's current public IP.

### Running the server
Launched inside a named `screen` session so it survives SSH disconnects:
```bash
screen -S bettermc
cd ~/bettermc
./start.sh
# detach with: Ctrl+A then D
# reattach with: screen -r bettermc
```

---

## Configuration

RAM is set in `variables.txt`:
```
JAVA_ARGS="-Xmx8G -Xms8G"
```
On this 12 GB instance, 8 GB is allocated to the server, leaving headroom for the OS. Don't set this higher than the box can spare or the JVM will fail to start or get killed.

---

## Mod notes

### Removed
- **packanalytics** — disabled (`.jar.disabled`). It crashed the server on boot via a compatibility break with the bundled Better Compatibility Checker (`NoClassDefFoundError: BetterStatusServerHolder`). It's pack telemetry only, so removing it has no gameplay impact.

### Added
- **Chunky 1.4.23** (NeoForge) — chunk pre-generator. Note: use the **1.4.x** line for MC 1.21.1; the 1.5.x builds require Minecraft 26.1+ and will not load here.

### Pre-generating the world
Run before inviting players (it's CPU-heavy on 2 ARM cores, so do it overnight):
```
chunky radius 3000
chunky start
```
Resume an interrupted run with `chunky continue`. Set a matching world border afterward:
```
worldborder set 6000
```

---



## Rebuilding from scratch

1. Provision an Oracle A1.Flex (ARM) instance with Ubuntu.
2. Open port 25565 in both the Oracle security list and `iptables`.
3. Download the **BMC5 Server Pack v50** (Linux) into `~/bettermc` and unzip it.
4. Restore the tracked config files from this repo over the defaults.
5. Set the EULA: `echo "eula=true" > eula.txt`
6. Confirm `variables.txt` heap matches the instance's RAM.
7. Re-add Chunky 1.4.23 (NeoForge) to `mods/` if pre-generating.
8. Start in `screen` with `./start.sh`.

---

## Pending / TODO

- [ ] **Automated backups to Oracle Object Storage** (highest priority — the world is not yet backed up off-instance)
- [ ] **`systemd` service** for auto-start on reboot and crash recovery
- [ ] **Reserved (static) public IP** so the address survives stop/start
- [ ] Whitelist for the friends group

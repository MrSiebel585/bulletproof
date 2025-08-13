# Bulletproof Omniscient — Immutable, Self-Healing Framework (v1)

> A production-ready architecture and implementation plan to make Omniscient tamper-resistant, self-validating, and fast to recover — without killing developer velocity.

---

## 1) Objectives

* **Immutable Core:** Core binaries, configs, and critical scripts are read-only and cryptographically verifiable.
* **Controlled Mutability:** Feature modules, logs, caches, and data live in dedicated, writable layers.
* **Self-Verification:** Background integrity checks with automatic quarantine/rollback on mismatch.
* **Zero-Trust Updates:** Only signed, attested artifacts are allowed to modify the system.
* **Rapid Recovery:** One-command restore from last-known-good snapshot/bundle.
* **Operator Clarity:** Human-readable logs and simple CLI to update, verify, and recover.

---

## 2) Filesystem & Layout (aligned with your preferences)

```
/opt/omniscient
├─ core/                      # Immutable base (read-only mount + chattr +i where supported)
│  ├─ bin/                    # Core launchers, guardians
│  ├─ conf/                   # Locked configs (templated -> rendered into /state/conf.d)
│  ├─ services/               # systemd units + hardening drops-ins
│  ├─ py/                     # Core Python (thin wrappers; non-mutable)
│  └─ VERSION                 # Monotonic version string (semver+build)
├─ modules/                   # Add-on tools (writable, signed updates only)
│  ├─ osint/
│  ├─ forensics/
│  ├─ hud/
│  └─ ...
├─ venv/                      # /opt/omniscient/venv (as you prefer)
├─ state/                     # Writable operational state
│  ├─ conf.d/                 # Rendered configs from core templates
│  ├─ data/
│  ├─ cache/
│  └─ run/
├─ logs/                      # Human-readable + JSONL, rotation policy
│  ├─ system/
│  ├─ security/
│  └─ ops/
├─ backups/                   # Short-term verified bundles (rolling)
├─ archives/                  # Long-term signed snapshots
├─ install/                   # Installer + update payload staging
└─ tools/                     # Admin CLIs: bulletproofctl, verify, recover, update
```

**Mount strategy**

* `/opt/omniscient/core` mounted **read-only** (`bind,ro`) + `chattr +i` for select files.
* `/opt/omniscient/modules` writable but **guarded** by signature gate.
* `/opt/omniscient/state`, `/opt/omniscient/logs` writable (noexec,nodev,nosuid).
* Optional: **OverlayFS** to layer modules over core for zero-downtime swaps.

**Optional FS hardening (pick based on host):**

* `fs-verity`/`dm-verity` for core artifacts on supported kernels.
* `btrfs` snapshots (subvols for `core`, `modules`) or ZFS clones.

---

## 3) Trust & Integrity Model

* **Git + Signed Tags** for source of truth.
* **Build attestation** (provenance metadata); **SBOM** generated at build time.
* **Release bundles** (`.tar.zst`):

  * `core-<ver>.tar.zst` (immutable layer)
  * `modules-<ver>.tar.zst` (writable layer)
  * `sbom-<ver>.json`, `checksums.txt`, `sig.minisig` (or cosign)
* **On-host verification** before staging → mount → activation.
* **AIDE** (or `integritywatch`) baseline over immutable + critical mutable paths.

---

## 4) Security Baselines

* Linux hardening flags on mounts: `noexec,nodev,nosuid` for writable dirs.
* `systemd` service hardening (PrivateTmp, ProtectHome, ProtectSystem=strict, RestrictAddressFamilies, CapabilityBoundingSet=, NoNewPrivileges=yes, etc.).
* **Auditd** rules for changes under `/opt/omniscient` and escalations.
* **AppArmor/SELinux** profiles (optional but recommended).

---

## 5) Update Protocol (Zero-Trust)

1. **Obtain bundle(s):** `core-<ver>.tar.zst`, `modules-<ver>.tar.zst`, `checksums.txt`, `sig.minisig`.
2. **Verify:**

   * Signature (minisign/cosign) → **reject on fail**
   * `sha256sum --check checksums.txt` → **reject on mismatch**
   * Optional: verify SBOM matches on-host hashes
3. **Stage:** extract into `/opt/omniscient/install/stage/<ver>`
4. **Preflight tests:** dry-run integrity, dependency check, unit tests.
5. **Activate:**

   * Flip bind mounts/OverlayFS upperdir → `/opt/omniscient/core` (ro), `/opt/omniscient/modules`
   * `systemctl reload-or-restart` impacted units
6. **Post-verify:** run `aide --check`; log attestation.
7. **Rollback plan:** atomically revert mounts to `prev` on any failure.

---

## 6) Recovery Plan

* `bulletproofctl recover --to <ver>`

  * Unmount active overlay/binds
  * Mount previous version subvol/snapshot
  * Re-run integrity baseline; relaunch services
* Bare metal: restore last good `archives/core-*.tar.zst` + `modules-*.tar.zst` (signed)

---

## 7) Installer (idempotent)

**File:** `/opt/omniscient/install/setup_bulletproof.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

OMNI_ROOT=/opt/omniscient
VENVDIR="$OMNI_ROOT/venv"
DIRS=(core core/bin core/conf core/services core/py modules state state/conf.d state/data state/cache state/run logs logs/system logs/security logs/ops backups archives install tools)

ensure_dir() { sudo mkdir -p "$1" && sudo chown -R "$USER":"$USER" "$1"; }

for d in "${DIRS[@]}"; do ensure_dir "$OMNI_ROOT/$d"; done

# Python venv (your preference)
if [[ ! -d "$VENVDIR" ]]; then
  python3 -m venv "$VENVDIR"
fi
source "$VENVDIR/bin/activate"
pip install --upgrade pip wheel
# Core packages (adjust to project)
pip install fastapi uvicorn[standard] rich pydantic

# Mount hardening via fstab drop-in (bind ro for core)
if ! grep -q "/opt/omniscient/core" /etc/fstab; then
  echo "/opt/omniscient/core /opt/omniscient/core none bind,ro 0 0" | sudo tee -a /etc/fstab >/dev/null
fi

# Writable dirs hardening
for d in modules state logs backups archives install; do
  sudo mount -o remount,noexec,nodev,nosuid "/opt/omniscient/$d" || true
  echo "/opt/omniscient/$d /opt/omniscient/$d none bind,noexec,nodev,nosuid 0 0" | sudo tee -a /etc/fstab >/dev/null
done

# AIDE baseline (if available)
if command -v aide >/dev/null; then
  sudo aideinit || true
fi

# Install service files
sudo cp "$OMNI_ROOT/core/services/omniscient-guard.service" /etc/systemd/system/ || true
sudo cp "$OMNI_ROOT/core/services/omniscient-core.service" /etc/systemd/system/ || true
sudo systemctl daemon-reload
sudo systemctl enable omniscient-guard.service omniscient-core.service
sudo systemctl start omniscient-guard.service omniscient-core.service

printf "\nBulletproof base installed. Reboot recommended to enforce ro binds.\n"
```

---

## 8) Systemd Units (hardened)

**`core/services/omniscient-core.service`**

```ini
[Unit]
Description=Omniscient Core API
After=network.target

[Service]
Type=simple
User=%i
WorkingDirectory=/opt/omniscient
Environment=PYTHONUNBUFFERED=1
ExecStart=/opt/omniscient/venv/bin/uvicorn omniscient_core:app --host 0.0.0.0 --port 8080
Restart=on-failure

# Hardening
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/opt/omniscient/state /opt/omniscient/logs
RuntimeDirectory=omniscient
CapabilityBoundingSet=
LockPersonality=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

[Install]
WantedBy=multi-user.target
```

**`core/services/omniscient-guard.service`**

```ini
[Unit]
Description=Bulletproof Integrity Guard
After=network.target

[Service]
Type=simple
User=%i
WorkingDirectory=/opt/omniscient
ExecStart=/opt/omniscient/tools/guard.sh
Restart=always
RestartSec=10

# Hardening
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/opt/omniscient/state /opt/omniscient/logs

[Install]
WantedBy=multi-user.target
```

---

## 9) Guard Script (integrity + auto-rollback)

**`/opt/omniscient/tools/guard.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail
ROOT=/opt/omniscient
LOG=$ROOT/logs/security/guard.log

log() { printf "[%s] %s\n" "$(date -Is)" "$*" | tee -a "$LOG"; }

verify_checksums() {
  local sumfile="$ROOT/core/checksums.txt"
  [[ -f "$sumfile" ]] || { log "No checksums.txt"; return 1; }
  (cd "$ROOT" && sha256sum --quiet --check core/checksums.txt) || return 1
}

baseline_aide() {
  command -v aide >/dev/null || return 0
  sudo aide --check >/dev/null || return 1
}

rollback() {
  log "Attempting rollback to previous snapshot..."
  if [[ -L "$ROOT/core_prev" ]]; then
    sudo umount "$ROOT/core" || true
    sudo mount --bind -o ro "$ROOT/core_prev" "$ROOT/core"
    systemctl reload-or-restart omniscient-core.service || true
    log "Rollback complete."
  else
    log "No previous snapshot link found. Manual recovery required."
  fi
}

main() {
  while true; do
    if ! verify_checksums; then
      log "Checksum mismatch detected"; rollback
    fi
    if ! baseline_aide; then
      log "AIDE integrity failure"; rollback
    fi
    sleep 60
  done
}

main
```

Make executable: `chmod +x /opt/omniscient/tools/guard.sh`

---

## 10) Update Tool (signature gate)

**`/opt/omniscient/tools/bulletproof-update`**

```bash
#!/usr/bin/env bash
set -euo pipefail
ROOT=/opt/omniscient
STAGE=$ROOT/install/stage
BUNDLE_DIR=${1:?"Usage: bulletproof-update <bundle_dir>"}

log() { printf "[%s] %s\n" "$(date -Is)" "$*"; }

verify_signatures() {
  if command -v minisign >/dev/null; then
    minisign -V -P "$ROOT/core/pubkey.minisign" -m "$BUNDLE_DIR/checksums.txt" -x "$BUNDLE_DIR/sig.minisig"
  else
    echo "minisign not found; abort"; exit 1
  fi
}

verify_hashes() { (cd "$BUNDLE_DIR" && sha256sum --check checksums.txt); }

stage() {
  mkdir -p "$STAGE"
  rsync -a --delete "$BUNDLE_DIR/" "$STAGE/new/"
}

activate() {
  # Keep previous for fast rollback
  if [[ -d "$ROOT/core" ]]; then
    rsync -a --delete "$ROOT/core/" "$ROOT/core_prev/"
  fi
  rsync -a --delete "$STAGE/new/core/" "$ROOT/core/"
  rsync -a "$STAGE/new/modules/" "$ROOT/modules/"
}

postverify() {
  (cd "$ROOT" && sha256sum --check core/checksums.txt)
  systemctl reload-or-restart omniscient-core.service || true
}

verify_signatures
verify_hashes
stage
activate
postverify
log "Update successful."
```

---

## 11) Developer Velocity (without breaking immutability)

* **Template-first configs:** Core holds Jinja2 templates; runtime renders into `/opt/omniscient/state/conf.d` (writable), leaving core read-only.
* **Module sandboxes:** Dev on branches → create signed module bundle → deploy via `bulletproof-update`.
* **Feature flags:** Stored in `/opt/omniscient/state/conf.d/flags.json` (audited), not in core.

---

## 12) Logging & Evidence

* **Dual-format logging:** Human-readable `.log` + machine `.jsonl` per component.
* **Tamper-evident chains:** Append-only `.jsonl` with per-line HMAC chain (key in root-only protected file).
* **Rotation:** `logrotate` with copytruncate for long-running services.

---

## 13) Snapshots & Backups

* Preferred: `btrfs subvolume snapshot -r` after each successful update; label with version.
* Portable fallback: signed `tar.zst` bundles in `/opt/omniscient/archives` with checksum and sig.

---

## 14) Monetization & Ops Impact

* **SLA-ready**: Publish MTTR ≤ 5 min via automated rollback, MTBF improved with guard.
* **Sellable SKU**: Ship `.deb` with `postinst` that wires mounts, services, guard, and creates a customer keypair.
* **Compliance leverage**: Attestations + SBOM help with enterprise sales and due diligence.

---

## 15) Quickstart (TL;DR)

```bash
sudo bash /opt/omniscient/install/setup_bulletproof.sh
# Place your service units and checksums in core/services and core/checksums.txt
# Start guard + core
sudo systemctl enable --now omniscient-guard.service omniscient-core.service

# Update cycle
bulletproof-update /opt/omniscient/install/bundles/release-2025-08-12
```

---

## 16) Next Steps (recommended)

1. Generate initial **checksums** and sign them with **minisign** (or cosign).
2. Decide on **btrfs snapshots** vs. pure tarball rollback.
3. Wire **AIDE baseline** to a `systemd.timer` for hourly checks.
4. Add per-module **allowlist** to constrain what each module can touch.
5. Convert this doc into a `.deb` *packaging spec* (I can scaffold `debian/` for you next).

---

### Appendix A — Example `checksums.txt`

```
sha256  7f1f...  core/bin/omniscientctl
sha256  8a12...  core/services/omniscient-core.service
sha256  c311...  core/services/omniscient-guard.service
...
```

### Appendix B — AIDE minimal config snippet

```
/opt/omniscient/core    NORMAL
/opt/omniscient/modules RMD160
/opt/omniscient/tools   NORMAL
```

### Appendix C — Logrotate snippet `/etc/logrotate.d/omniscient`

```
/opt/omniscient/logs/**/*.log {
  daily
  rotate 14
  compress
  missingok
  copytruncate
}
```

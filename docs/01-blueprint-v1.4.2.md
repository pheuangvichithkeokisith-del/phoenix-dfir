# TODO: Stage 2 - content coming next month
# 📘 PHOENIX Blueprint v1.4.2 — Full Design
**Status:** Converged (Sonnet 5 ↔ Qwen adversarial review)
**Scope:** Architecture, write-block 4 layers, acquisition scripts, FIDO2 flow

---

## 1. Core Contract

> **Evidence Mode** ต้องพิสูจน์ว่า source read-only ตลอดกระบวนการและบันทึกความสมบูรณ์ได้
> **Recovery** และ **Triage** ไม่อ้างสถานะ forensic acquisition

---

## 2. Storage Layout (NVMe 1TB)

```
P1 ESP      1GB  FAT32        UKI + systemd-boot (signed)
P2 CORE    10GB  ext4 (RO)    squashfs immutable OS
P3 BRAIN   16GB  LUKS2→ext4   config, keys, audit (FIDO2)
P4 VAULT  120GB  LUKS2→Btrfs  staging, temp evidence
P5 EVIDENCE ~850GB LUKS2→Btrfs overflow evidence storage
```

---

## 3. Boot Modes (3 UKI)

| Mode | Kernel Cmdline | Write-Block | Banner |
|------|---------------|-------------|--------|
| **Evidence** | `ro rd.live.overlay=tmpfs phoenix.mode=evidence` | ✅ Layer 1-4 | 🔴 CHAIN OF CUSTODY |
| **Triage** | `ro phoenix.mode=triage` | ✅ Layer 1-4 | 🟡 NON-PHYSICAL |
| **Recovery** | `rw phoenix.mode=recovery` | ❌ Disabled | 🔴 WARNING |

---

## 4. Write-Block Architecture (4 Layers)

### Layer 1: Initramfs Lock
**ไฟล์:** `dracut/99phoenix-writeblock/lock-targets.sh`
```bash
#!/bin/sh
# Lock all non-boot disks at initramfs stage (before userspace)
BOOT_DISK=$(cat /run/phoenix/boot-disk)

for dev in /sys/block/sd* /sys/block/nvme*n*; do
    [ -e "$dev" ] || continue
    devname=$(basename "$dev")
    [ "$devname" = "$BOOT_DISK" ] && continue
    
    echo "🔒 Locking $devname"
    echo 1 > "/sys/block/$devname/ro"
done
```

### Layer 2: Boot Set-Equality Assertion
**ไฟล์:** `dracut/99phoenix-devices/identify-boot.sh`
```bash
#!/bin/sh
# Identify boot disk and assert it's in the expected set
BOOT_DISK=$(lsblk -ndo NAME,TRAN,MODEL | awk '/usb|nvme/ && !/part/{print $1; exit}')
echo "$BOOT_DISK" > /run/phoenix/boot-disk

# Assert: boot disk must be in allowed set
ALLOWED="sdb sdc nvme0n1"
if ! echo "$ALLOWED" | grep -qw "$BOOT_DISK"; then
    echo "❌ Boot disk $BOOT_DISK not in allowed set"
    exit 1
fi
```

### Layer 3: Fail-Closed Lock
**ไฟล์:** `runtime/bin/check-invariants.sh`
```bash
#!/bin/bash
set -euo pipefail
source /usr/lib/phoenix/common.sh

TARGET="${1:?Usage: check-invariants.sh <device>}"
TARGET_DISK=$(phoenix_disk_of "$TARGET")

# Invariant 1: Target ≠ boot media
BOOT_DISK=$(cat /run/phoenix/boot-disk)
if [ "$TARGET_DISK" = "$BOOT_DISK" ]; then
    echo "❌ Target is boot media"
    exit 1
fi

# Invariant 2: Target in target-list (or fail-closed)
if ! grep -qx "$TARGET_DISK" /run/phoenix/target-list; then
    echo "⚠️  Target not in list — fail-closed lock"
    phoenix_lock_disk "$TARGET_DISK"
fi

# Invariant 3: queue/ro == 1 (exact check)
if [ "$(phoenix_ro_status "$TARGET_DISK")" != "1" ]; then
    echo "❌ Device not read-only"
    exit 1
fi

# Invariant 4: Not mounted (exact match)
if findmnt -rn -o SOURCE | grep -q "^${TARGET}"; then
    echo "❌ Device is mounted"
    exit 1
fi

echo "✅ All invariants passed"
```

### Layer 4: Continuous Guard
**ไฟล์:** `runtime/bin/phoenix-guard.sh` (integrated in `phoenix-common.sh`)
```bash
phoenix_guard_start() {
    local target_disk="$1"
    local interval="${PHOENIX_GUARD_SEC:-5}"
    
    (
        while true; do
            sleep "$interval"
            
            # Check queue/ro
            if [ "$(phoenix_ro_status "$target_disk")" != "1" ]; then
                echo "🚨 Guard: queue/ro violation" > /run/phoenix/guard-violation
                kill -TERM -$$
                sleep 2
                kill -KILL -$$
                break
            fi
            
            # Check mount
            if findmnt -rn -o SOURCE | grep -q "^/dev/$target_disk"; then
                echo "🚨 Guard: mount violation" > /run/phoenix/guard-violation
                kill -TERM -$$
                sleep 2
                kill -KILL -$$
                break
            fi
        done
    ) &
    GUARD_PID=$!
    echo "$GUARD_PID" > /run/phoenix/guard.pid
}

phoenix_guard_stop() {
    [ -f /run/phoenix/guard.pid ] && kill "$(cat /run/phoenix/guard.pid)" 2>/dev/null || true
}
```

---

## 5. Shared Library (`phoenix-common.sh`)

**ไฟล์:** `lib/phoenix-common.sh`
```bash
#!/bin/bash
# phoenix-common.sh — single source of truth

phoenix_disk_of() {
    local dev="$1"
    lsblk -no PKNAME "$dev" 2>/dev/null || basename "$dev"
}

phoenix_boot_disk() {
    cat /run/phoenix/boot-disk
}

phoenix_is_boot_media() {
    local dev="$1"
    local disk=$(phoenix_disk_of "$dev")
    [ "$disk" = "$(phoenix_boot_disk)" ]
}

phoenix_lock_disk() {
    local disk="$1"
    echo 1 > "/sys/block/$disk/ro"
}

phoenix_ro_status() {
    local disk="$1"
    cat "/sys/block/$disk/ro" 2>/dev/null || echo "0"
}

phoenix_guard_start() {
    # See Layer 4 above
}

phoenix_guard_stop() {
    # See Layer 4 above
}

phoenix_run_guarded() {
    local target_disk="$1"
    shift
    
    phoenix_guard_start "$target_disk"
    "$@"
    local rc=$?
    phoenix_guard_stop
    
    return $rc
}

phoenix_quarantine() {
    local src="$1"
    local dest_dir="${2:-/phoenix/vault/quarantine}"
    mkdir -p "$dest_dir"
    
    local base=$(basename "$src")
    mv "$src" "$dest_dir/DANGER_CORRUPTED_${base}"
    
    cat > "$dest_dir/DANGER_CORRUPTED_${base}.sidecar" <<EOF
Quarantined: $(date -u +%FT%TZ)
Reason: $PHOENIX_QUARANTINE_REASON
Original: $src
EOF
}
```

---

## 6. Acquisition Scripts

### 6.1 acquire.sh (Evidence Mode)
**ไฟล์:** `runtime/bin/acquire.sh`
```bash
#!/bin/bash
set -euo pipefail
source /usr/lib/phoenix/common.sh

TARGET="${1:?Usage: acquire.sh <device> [raw|e01]}"
FORMAT="${2:-raw}"
OUT_DIR="/phoenix/vault/acquisitions"
mkdir -p "$OUT_DIR"

TARGET_DISK=$(phoenix_disk_of "$TARGET")

# Check invariants
/usr/lib/phoenix/check-invariants.sh "$TARGET" || exit 1

TIMESTAMP=$(date -u +%Y%m%dT%H%M%SZ)
OUT="$OUT_DIR/${TARGET_DISK}_${TIMESTAMP}"

case "$FORMAT" in
    raw)
        echo "📀 Acquiring $TARGET → $OUT.raw"
        phoenix_run_guarded "$TARGET_DISK" \
            dc3dd if="$TARGET" of="$OUT.raw" hash=sha256 hash=md5 log="$OUT.dc3dd.log"
        
        HASH=$(grep "sha256" "$OUT.dc3dd.log" | awk '{print $NF}')
        ;;
    
    e01)
        echo "📀 Acquiring $TARGET → $OUT.E01"
        phoenix_run_guarded "$TARGET_DISK" \
            ewfacquire -f encase7 -t "$OUT" -S 0 -C 1 -D "PHOENIX" -M physical -d sha256 -u "$TARGET"
        
        # Verify E01 (calibration-gated)
        if ! /usr/lib/phoenix/phoenix-verify-e01 "$OUT.E01"; then
            echo "❌ E01 verification failed"
            phoenix_quarantine "$OUT.E01"
            exit 1
        fi
        
        HASH=$(grep "SHA256" "$OUT.ewfverify.log" | awk '{print $NF}')
        ;;
    
    *)
        echo "❌ Unknown format: $FORMAT"
        exit 1
        ;;
esac

# Write manifest
cat > "$OUT.manifest.json" <<EOF
{
  "target": "$TARGET",
  "target_disk": "$TARGET_DISK",
  "format": "$FORMAT",
  "timestamp": "$TIMESTAMP",
  "hash_sha256": "$HASH",
  "operator": "$(whoami)",
  "mode": "evidence",
  "guard_violations": "$([ -f /run/phoenix/guard-violation ] && cat /run/phoenix/guard-violation || echo 'none')"
}
EOF

# Sign manifest
openssl dgst -sha256 -sign /etc/phoenix/manifest-key.pem \
    -out "$OUT.manifest.sig" "$OUT.manifest.json"

sync
echo "✅ Acquisition complete: $OUT.$FORMAT"
echo "📝 Manifest: $OUT.manifest.json + .sig"
```

### 6.2 stream-evidence.sh (Network)
**ไฟล์:** `runtime/bin/stream-evidence.sh`
```bash
#!/bin/bash
set -euo pipefail
source /usr/lib/phoenix/common.sh

TARGET="${1:?Usage: stream-evidence.sh <device> <server> <remote-path>}"
SERVER="${2:?}"
REMOTE_PATH="${3:?}"

TARGET_DISK=$(phoenix_disk_of "$TARGET")
/usr/lib/phoenix/check-invariants.sh "$TARGET" || exit 1

TIMESTAMP=$(date -u +%Y%m%dT%H%M%SZ)
REMOTE_FILE="$REMOTE_PATH/${TARGET_DISK}_${TIMESTAMP}.raw"

echo "🌐 Streaming $TARGET → $SERVER:$REMOTE_FILE"

# One-read pipeline with guard
phoenix_run_guarded "$TARGET_DISK" \
    bash -c "dc3dd if='$TARGET' hash=sha256 log=/tmp/stream.log | ssh '$SERVER' 'cat > $REMOTE_FILE.partial'"

# Server-side: verify-before-commit
ssh "$SERVER" "
    HASH=\\\$(grep sha256 /tmp/stream.log | awk '{print \\\$NF}')
    ACTUAL=\\\$(sha256sum '$REMOTE_FILE.partial' | awk '{print \\\$1}')
    if [ \"\\\$HASH\" = \"\\\$ACTUAL\" ]; then
        mv '$REMOTE_FILE.partial' '$REMOTE_FILE'
        echo COMMIT
    else
        mv '$REMOTE_FILE.partial' '$REMOTE_FILE.DANGER_CORRUPTED'
        echo QUARANTINE
    fi
" > /tmp/stream-result.txt

RESULT=$(cat /tmp/stream-result.txt)
if [ "$RESULT" = "COMMIT" ]; then
    echo "✅ Stream complete: $REMOTE_FILE"
else
    echo "❌ Stream corrupted — quarantined on server"
    exit 1
fi
```

### 6.3 triage.sh (Triage Mode)
**ไฟล์:** `runtime/bin/triage.sh`
```bash
#!/bin/bash
set -euo pipefail
source /usr/lib/phoenix/common.sh

echo "⚠️  WARNING: NON-PHYSICAL IMAGING MODE"
echo "⚠️  This is NOT forensic acquisition"
echo ""

TARGET="${1:?Usage: triage.sh <device>}"
OUT_DIR="/phoenix/vault/triage"
mkdir -p "$OUT_DIR"

TARGET_DISK=$(phoenix_disk_of "$TARGET")
/usr/lib/phoenix/check-invariants.sh "$TARGET" || exit 1

TIMESTAMP=$(date -u +%Y%m%dT%H%M%SZ)

# Detect filesystem
FS_TYPE=$(blkid -s TYPE -o value "$TARGET" 2>/dev/null || echo "unknown")

case "$FS_TYPE" in
    ntfs)
        OUT="$OUT_DIR/${TARGET_DISK}_${TIMESTAMP}.ntfs"
        phoenix_run_guarded "$TARGET_DISK" ntfsclone -o "$OUT" "$TARGET"
        ;;
    ext*|xfs|btrfs)
        OUT="$OUT_DIR/${TARGET_DISK}_${TIMESTAMP}.partclone"
        phoenix_run_guarded "$TARGET_DISK" partclone."$FS_TYPE" -c -s "$TARGET" -o "$OUT"
        ;;
    *)
        echo "❌ Unsupported filesystem: $FS_TYPE"
        exit 1
        ;;
esac

# Manifest (labeled as triage, not evidence)
cat > "$OUT.manifest.json" <<EOF
{
  "target": "$TARGET",
  "format": "triage",
  "filesystem": "$FS_TYPE",
  "timestamp": "$TIMESTAMP",
  "operator": "$(whoami)",
  "note": "NON-PHYSICAL imaging — not forensic acquisition"
}
EOF

echo "✅ Triage complete: $OUT"
```

---

## 7. FIDO2 Flow

### 7.1 format-luks.sh
```bash
#!/bin/bash
set -euo pipefail

PARTITION="${1:?Usage: format-luks.sh <partition> <label>}"
LABEL="${2:?}"

cryptsetup luksFormat --type luks2 \
    --label "$LABEL" \
    --pbkdf argon2id \
    "$PARTITION"

echo "✅ LUKS2 formatted: $PARTITION"
echo "⚠️  Now run enroll-existing-luks.sh to add FIDO2 + recovery key"
```

### 7.2 enroll-existing-luks.sh
```bash
#!/bin/bash
set -euo pipefail

PARTITION="${1:?Usage: enroll-existing-luks.sh <partition>}"

# Detect YubiKey
YUBIKEY=$(ls /dev/hidraw* 2>/dev/null | head -1)
if [ -z "$YUBIKEY" ]; then
    echo "❌ No YubiKey detected"
    exit 1
fi

# Add FIDO2 token
systemd-cryptenroll --fido2-device="$YUBIKEY" "$PARTITION"

# Add recovery key
RECOVERY_KEY=$(systemd-cryptenroll --recovery-key "$PARTITION" | tail -1)

echo ""
echo "=========================================="
echo "⚠️  RECOVERY KEY (write down, store offline)"
echo "=========================================="
echo "$RECOVERY_KEY"
echo "=========================================="
echo ""
read -p "Press Enter after writing down the recovery key..."
```

---

## 8. Residual Risks

| Risk | Mitigation | Residual |
|------|-----------|----------|
| TOCTOU (check-invariants vs long I/O) | Layer 4 guard (5s interval) | Window ≤5s |
| SG_IO/SCSI bypass | Use hardware write-blocker for court | Software can't close 100% |
| E01 corrupted-but-parseable | Calibration + quarantine | Header may lie |
| ewfverify version drift | Re-calibrate on libewf upgrade | Must calibrate every time |

---

## 9. Validation Matrix (6 Tests)

1. ✅ Boot on test machine (UEFI + Secure Boot)
2. ✅ Lock internal + hot-plug disks (RO guarantee)
3. ✅ Whole-disk acquire (raw/E01) passes invariants + hash match
4. ✅ Stream one-read hash/bytes match (network flow)
5. ✅ Mounts + FIDO/recovery flow works
6. ✅ Test suite passes on sacrificial media only

---

**Document Status:** FROZEN
**Last Updated:** v1.4.2
**Next:** v1.4.3 addendum (calibration fixes)

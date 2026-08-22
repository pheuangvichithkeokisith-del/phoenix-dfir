# PHOENIX Appliance
# 🐦 PHOENIX — Portable Forensic & Recovery Toolkit

> Design-phase repository • **Evidence ≠ Triage ≠ Recovery**
> Status: 🧊 Design FROZEN (v1.4.3) — awaiting hardware calibration

---

## 📜 Core Contract

**Evidence Mode** ต้องพิสูจน์ว่า source read-only ตลอดกระบวนการและบันทึกความสมบูรณ์ได้

**Recovery** และ **Triage** ไม่อ้างสถานะ forensic acquisition — เด็ดขาด

---

## 🏗️ Architecture Overview

### Storage Layout (NVMe 1TB)
```
┌──────────┬──────────────┬──────────────┬───────────────┬────────────────┐
│ P1 ESP   │ P2 CORE      │ P3 BRAIN     │ P4 VAULT      │ P5 EVIDENCE    │
│ FAT32    │ ext4 (RO)    │ LUKS2→ext4   │ LUKS2→Btrfs   │ LUKS2→Btrfs    │
│ 1GB      │ 10GB         │ 16GB         │ 120GB         │ ~850GB         │
└──────────┴──────────────┴──────────────┴───────────────┴────────────────┘
```

### Boot Modes (3 UKI, signed)
| Mode | Write Block | Banner |
|------|-------------|--------|
| **Evidence** | ✅ 4-layer lock | 🔴 CHAIN OF CUSTODY |
| **Triage** | ✅ 4-layer lock | 🟡 NON-PHYSICAL |
| **Recovery** | ❌ Disabled | 🔴 WARNING |

### Write-Block 4 Layers
1. **Initramfs lock** — lock before userspace
2. **Boot set-equality assertion** — fail loud if mismatch
3. **Fail-closed lock** — at acquisition time
4. **Continuous guard** — re-assert every 5s (TOCTOU mitigation)

---

## 📁 Repository Structure

```
phoenix-dfir/
├── lib/          shared library (single source of truth)
├── dracut/       initramfs hooks (devices + writeblock)
├── runtime/      on-device scripts (acquisition, udev, guard)
├── build/        build-host-only scripts (partition, LUKS, UKI)
├── tests/        validation matrix + fixtures
├── docs/         design documents (v1.4.2 → v1.4.3)
└── context/      AI context package (for chat handoff)
```

**Key Design Documents:**
- [`docs/01-blueprint-v1.4.2.md`](docs/01-blueprint-v1.4.2.md) — Full design + code artifacts
- [`docs/00-project-structure-v1.4.3.md`](docs/00-project-structure-v1.4.3.md) — High-level structure
- [`docs/review-log.md`](docs/review-log.md) — Attribution-based review history

---

## 🔐 Security Principles

- **FIDO2 without TPM2** — portable media must not bind to hardware
- **Minimize, Measure, Document, Justify** — zero-touch is impossible under UEFI
- **Software write-blocker ≠ hardware write-blocker** — use Tableau/WiebeTech for court cases
- **Calibration-gated E01 verify** — human-in-the-loop mandatory on first run

---

## 📋 Pre-Deployment Checklist

### Boot & Lock
- [ ] Boot on test machine (UEFI + Secure Boot)
- [ ] Lock internal + hot-plug disks (RO guarantee)
- [ ] Runtime lock works (udev)
- [ ] Target-list dynamic update

### Acquisition
- [ ] Whole-disk acquire passes invariants
- [ ] Raw/E01 hash assert
- [ ] Stream one-read verify
- [ ] Continuous guard abort + quarantine

### Mount & Security
- [ ] BRAIN/VAULT/EVIDENCE mounts work
- [ ] FIDO2 enroll + recovery flow
- [ ] Secure Boot signed + verified
- [ ] tmpfs caps (50%/25%)

### Calibration
- [ ] Run `calibrate-ewfverify.sh` on actual libewf version
- [ ] Known-good exit=0, known-corrupted (×3) exit≠0 or distinguishable
- [ ] **Operator reads all 4 logs manually**
- [ ] **Operator chooses mode manually** (exitcode/text/export-hash)
- [ ] Confirm `/etc/phoenix/ewf-verify-mode` ≠ UNCALIBRATED
- [ ] Record `calibration-record.txt`

---

## ⚠️ Disclaimer

- This is **design documentation** — no production code yet
- Software write-blocker is a **mitigation**, not a replacement for hardware write-blocker
- **Never commit** keys, recovery keys, audit logs, or evidence (see `.gitignore`)

---

## 🗓️ Roadmap

- [x] Design phase (v1.4.3 FROZEN)
- [ ] Hardware calibration (`calibrate-ewfverify.sh`) — next month
- [ ] VM sandbox test (QEMU/OVMF)
- [ ] Hardware burn-in (NVMe + YubiKey + sacrificial disk)
- [ ] Validation matrix 6 tests → v1.0 Gold Master

---

## 👥 Review History

Designed and reviewed by 4 AI assistants (design phase):

- **Qwen** — primary architect
- **Gemini Pro 3** — storage architecture (USB→SSD pivot), handed off after scope complete
- **ChatGPT Terra** — prototype scaffold, handed off after scope complete
- **Claude Sonnet 5** — TOCTOU/calibration logic + final structure review (ongoing)
See [`docs/review-log.md`](docs/review-log.md) for attribution details.

---

## 📚 Context Package

When starting a new chat to work on this project, paste the contents of:
- `context/dfir-context.md` — project context
- `context/dfir-rules.md` — chat rules
- `context/dfir-workflow.md` — workflow sequence

---

**Document Status:** FROZEN — Ready for External Review  
**Last Updated:** v1.4.3  
**Next Milestone:** Hardware Calibration (next month)

# TODO: Stage 2 - content coming next month
# 📘 PROJECT PHOENIX — Structure & Design Document
**Version:** v1.4.3 (Final Design Artifact)
**Status:** Design FROZEN — Ready for Review

---

## 0. Executive Summary

PHOENIX คือ Portable Forensic & Recovery Toolkit สำหรับงาน DFIR ที่ออกแบบมาให้:
- **Evidence Mode:** เก็บหลักฐานแบบ read-only พร้อม cryptographic chain of custody
- **Triage Mode:** ประเมินสถานการณ์แบบ non-physical (used-block imaging)
- **Recovery Mode:** กู้ระบบแบบ write-enabled (มี warning ชัดเจน)

**Core Contract:**
> Evidence Mode ต้องพิสูจน์ว่า source read-only ตลอดกระบวนการและบันทึกความสมบูรณ์ได้
> Recovery และ Triage ไม่อ้างสถานะ forensic acquisition

---

## 1. Architecture Overview

### 1.1 Storage Layout (NVMe 1TB)
```
┌──────────┬──────────────┬──────────────┬───────────────┬────────────────┐
│ P1 ESP   │ P2 CORE      │ P3 BRAIN     │ P4 VAULT      │ P5 EVIDENCE    │
│ FAT32    │ ext4 (RO)    │ LUKS2→ext4   │ LUKS2→Btrfs   │ LUKS2→Btrfs    │
│ 1GB      │ 10GB         │ 16GB         │ 120GB         │ ~850GB         │
└──────────┴──────────────┴──────────────┴───────────────┴────────────────┘
```

**บทบาทของแต่ละ partition:**
- **ESP:** UKI + systemd-boot (signed)
- **CORE:** Immutable OS (squashfs) — อ่านอย่างเดียว
- **BRAIN:** Config, keys, audit log (FIDO2 protected)
- **VAULT:** Staging area, temporary evidence
- **EVIDENCE:** Overflow storage สำหรับ evidence ขนาดใหญ่

### 1.2 Boot Modes (3 UKI, signed)
| Mode | Kernel Cmdline | Write Block | Banner |
|------|---------------|-------------|--------|
| **Evidence** | `ro, rd.live.overlay=tmpfs, phoenix.mode=evidence` | ✅ เปิด | แดง: CHAIN OF CUSTODY |
| **Triage** | `ro, phoenix.mode=triage` | ✅ เปิด | เหลือง: NON-PHYSICAL |
| **Recovery** | `rw, phoenix.mode=recovery` | ❌ ปิด | แดงเข้ม: WARNING |

### 1.3 Boot Chain
```
UEFI → systemd-boot → UKI (ukify) → dracut initramfs
     → phoenix hooks → dmsquash-live → systemd → rootfs
```

---

## 2. Write-Block Architecture (4 Layers)

| Layer | Mechanism | พิสูจน์ |
|-------|-----------|----------|
| **1** | initramfs lock ก่อน userspace | ล็อกก่อนใครแตะ |
| **2** | Boot set-equality assertion | ลิสต์ครบ — fail loud |
| **3** | Fail-closed lock ณ acquisition | device แปลกก็ถูกล็อก |
| **4** | Continuous guard + TERM→grace→KILL | ล็อกค้างตลอดกระบวนการ |

### 2.1 Key Invariants (check-invariants.sh)
```
✓ Target ≠ boot media
✓ Target อยู่ใน target-list (หรือ fail-closed lock)
✓ queue/ro == 1 (exact check)
✓ ไม่ถูก mount (findmnt + lsblk exact match)
```

### 2.2 Continuous Guard
- Re-assert ทุก `PHOENIX_GUARD_SEC` (default 5s)
- ตรวจทั้ง `queue/ro` และ `mount` status
- ถ้า violation → `kill -TERM -PGID`, grace 2s, `kill -KILL -PGID`
- เขียน `/run/phoenix/guard-violation` marker
- Acquisition abort + quarantine + signed ABORTED manifest

---

## 3. Shared Library (`phoenix-common.sh`)

**Single source of truth สำหรับ:**
- `phoenix_disk_of()` — resolve parent disk
- `phoenix_boot_disk()` — cached boot identity
- `phoenix_is_boot_media()` — safe check
- `phoenix_lock_disk()` / `phoenix_ro_status()` — RO control
- `phoenix_guard_start/stop()` — continuous guard
- `phoenix_run_guarded()` — argv-only acquisition runner
- `phoenix_quarantine()` — DANGER_CORRUPTED_ filename + sidecar

**Install:** `inst /usr/lib/phoenix/common.sh` ใน module-setup.sh ทั้งสอง module

---

## 4. Acquisition Scripts

### 4.1 acquire.sh (Evidence Mode)
- **Raw:** `dc3dd if=DEV of=OUT hash=sha256`
- **E01:** `ewfacquire -f encase7 -d sha256 -M physical`
- **Verify:** `phoenix-verify-e01` (calibration-gated) → เช็ค E01 integrity ตาม mode ที่ calibrate ไว้
- **Parse & Gate:** ดึง `HASH` จาก `ewfverify.log` → `[ -n "$HASH" ]` → **ปฏิเสธเซ็น manifest ถ้า parse ไม่ได้หรือ verify fail**
- **Manifest:** jq → ECDSA sign → sync → audit

### 4.2 stream-evidence.sh (Network)
- One-read: `dc3dd | ssh "cat > /tmp/FILE.partial"`
- Server: verify-before-commit → `COMMIT` หรือ `QUARANTINE`
- Field: parse result, write manifest

### 4.3 triage.sh (Triage Mode)
- `partclone` / `ntfsclone` (used-block only)
- **Banner:** NON-PHYSICAL WARNING
- Manifest labeled `triage`

### 4.4 phoenix-stream-once.sh
- Pipeline แยกออกมาสคริปต์เดี่ยว (quoting ปลอดภัย)
- Guard คุ้มครองตลอด streaming

---

## 5. E01 Verification (Calibration-Gated)

### 5.1 Design
- `/etc/phoenix/ewf-verify-mode` = `exitcode` | `text` | `export-hash` | `UNCALIBRATED`
- ถ้า `UNCALIBRATED` → **ปฏิเสธการเซ็น manifest**
- **Mode ถูกเลือกโดย operator ผ่าน `calibrate-ewfverify.sh`** (human-in-the-loop บังคับรอบแรก)
  สคริปต์รวบรวมข้อมูล known-good/known-corrupted แล้ว **operator อ่าน log และตัดสินใจเอง** ว่า exit code หรือ text pattern แยกแยะได้จริง

### 5.2 Calibration Script
- Known-good 1 + known-corrupted 3 ตำแหน่ง (75%, 85%, ท้ายไฟล์)
- Human-in-the-loop รอบแรก (ห้าม automate)
- บันทึก `calibration-record.txt` (audit trail ของ calibration เอง)

---

## 6. FIDO2 & Security

### 6.1 LUKS2 Enrollment
- **Format:** `format-luks.sh` (แยกจาก enroll)
- **Enroll:** `enroll-existing-luks.sh` (FIDO2 + recovery key พิมพ์ออกจอ)
- **ห้าม:** `--tpm2-device` (สื่อพกพาต้องไม่ผูกกับเครื่อง)

### 6.2 Mount Units
- `systemctl --root=$ROOTFS enable brain.mount phoenix-vault.mount phoenix-evidence.mount`
- Btrfs: `compress=zstd:3,noatime`

### 6.3 fstab
```
tmpfs /tmp       tmpfs defaults,noatime,mode=1777,size=50% 0 0
tmpfs /var/tmp   tmpfs defaults,noatime,mode=1777,size=25% 0 0
```
**ไม่มี swap**

---

## 7. Build Pipeline

### 7.1 mkosi Profiles
- `mkosi.conf.field` (Debian bookworm)
- `mkosi.conf.lab` (Fedora)

### 7.2 UKI Generation
```
ukify build --linux=VMLINUZ --initrd=INITRD --cmdline=CMDLINE \
    --os-release=... --output=phoenix-MODE.efi
```
**Verify:** `lsinitrd INITRD | grep phoenix`

### 7.3 Secure Boot
```
sbctl sign systemd-bootx64.efi BOOTX64.EFI \
    phoenix-evidence.efi phoenix-triage.efi phoenix-recovery.efi
```

### 7.4 Build Steps
```
1. mkosi build → rootfs
2. Bundle manifest-pub.pem → rootfs
3. mksquashfs → core.squashfs
4. Build UKI (3 modes)
5. Copy UKI → ESP
6. bootctl install
7. sbctl sign
8. Populate BRAIN
```

---

## 8. Residual Risks (บันทึกอย่างซื่อสัตย์)

| Risk | Class | Mitigation | Residual |
|------|-------|-----------|----------|
| TOCTOU ระหว่าง check-invariants กับ long-running I/O | **Timing attack class** | Continuous Guard (Layer 4) re-assert ทุก 5s | window ≤5s ยังเหลืออยู่ — จับเป็น abort+document ไม่ใช่ elimination |
| Guard polling interval ≤5s | Magnitude ของ guard เดียวกัน | ลด `PHOENIX_GUARD_SEC` ได้ถ้าต้องการ | แลกกับ CPU overhead |
| Software RO bypass ด้วย SG_IO/SCSI passthrough | Kernel-level bypass | คดีศาลใช้ hardware write-blocker | software ไม่มีวันปิด 100% |
| E01 corrupted-but-parseable | File format ambiguity | DANGER marker + quarantine + ห้ามวิเคราะห์โดยไม่ re-verify | header อาจ claim completeness เท็จ |
| ewfverify output ไม่แน่นอน | Tool version dependency | Calibration-gated + human-in-the-loop | ต้อง calibrate ทุกครั้งที่เปลี่ยน libewf version |

**Note:** TOCTOU = **reason Layer 4 exists** ไม่ใช่แค่ side effect

---

## 9. Pre-Deployment Checklist

### 9.1 Boot & Lock
- [ ] บูตได้บนเครื่องทดสอบ
- [ ] Evidence mode ล็อก internal + hot-plug disk
- [ ] Runtime lock ทำงาน (udev)
- [ ] Target-list dynamic update

### 9.2 Acquisition
- [ ] Whole-disk acquire ผ่าน invariants
- [ ] Raw/E01 hash assert
- [ ] Stream one-read verify
- [ ] Continuous guard abort + quarantine

### 9.3 Mount & Security
- [ ] BRAIN/VAULT/EVIDENCE mounts ทำงาน
- [ ] FIDO2 enroll + recovery flow
- [ ] Secure Boot signed + verified
- [ ] tmpfs caps (50%/25%)

### 9.4 Calibration
- [ ] รัน `calibrate-ewfverify.sh` บน libewf เวอร์ชันจริงที่ build
- [ ] ตรวจสอบว่า known-good exit=0 และ known-corrupted ทั้ง 3 ตำแหน่ง exit≠0 (หรือแยกแยะได้ด้วย text pattern)
- [ ] **Operator อ่าน log ทั้ง 4 ไฟล์ด้วยตา** (good.log + bad-*.logs × 3)
- [ ] **Operator เลือก mode ด้วยตนเอง** (exitcode/text/export-hash) แล้วพิมพ์คำตอบ
- [ ] ยืนยัน `/etc/phoenix/ewf-verify-mode` ≠ UNCALIBRATED
- [ ] บันทึก `calibration-record.txt` (ใครเลือก mode ไหน เมื่อไหร่)

---

## 10. Compliance Stance

**"Aligned with / designed to support":**
- NIST SP 800-86
- ISO/IEC 27037
- SWGDE

**ไม่เคลม "achieved"** โดยไม่มีผล validation

---

## 11. Review Notes

### 11.1 Reviewers
- **Qwen** (primary architect)
- **ChatGPT Terra** (architecture/boot-chain)
- **Gemini Pro 3** (operational reality)
- **Claude Sonnet 5** (TOCTOU/calibration logic)

### 11.2 Key Decisions
- **Severity = consequence × unresolved proof burden** (ไม่ใช่ probability)
- **Minimize, Measure, Document, Justify** (zero-touch เป็นไปไม่ได้ใต้ UEFI)
- **Evidence ≠ Triage ≠ Recovery** (แยก boot entry/banner/audit)
- **External-first storage** (NVMe = staging)
- **FIDO2 without TPM2** (สื่อพกพา)

### 11.3 Version History
| Version | Key Changes |
|---------|-------------|
| v1.0 | Initial design |
| v1.4.1 | P0 fixes (13 items) |
| v1.4.2 | Continuous guard, argv-only runner, DANGER marker |
| v1.4.3 | Exact-mount check, fail-fast, calibration-gated verify |

---

## 12. Files to Review

### 12.1 Design Documents
- `PHOENIX-v1.4.2-blueprint.md` — design หลัก
- `PHOENIX-v1.4.3-final-addendum.md` — patch สุดท้าย
- `PHOENIX-v1.4.3-calibration-revision.md` — calibration script

### 12.2 Context Package
- `dfir-context.md` — project context สำหรับเปิดแชทใหม่
- `dfir-rules.md` — กฎการทำงาน
- `dfir-workflow.md` — ลำดับงาน

### 12.3 Code Artifacts (ใน blueprint)
- `phoenix-common.sh` — shared library
- `identify-boot.sh` — boot-time identification
- `check-invariants.sh` — safety gate
- `acquire.sh` / `stream-evidence.sh` / `triage.sh` — acquisition scripts
- `calibrate-ewfverify.sh` — calibration script
- Mount units, fstab, udev rules, dracut modules

---

## 13. Reviewer Action Items

**สำหรับผู้ที่ต้องการรีวิว:**

1. **อ่าน Section 0-3** ก่อน (architecture + write-block)
2. **ตรวจสอบ Section 5** (calibration logic) — จุดที่ซับซ้อนที่สุด
3. **รีวิว Section 9** (checklist) — ครบถ้วนหรือไม่
4. **Comment ใน Section 11.2** — เห็นด้วยกับ key decisions ไหม
5. **ส่ง feedback กลับมาก่อน hardware test**

---

## 14. Contact & Timeline

- **Design Phase:** ✅ COMPLETE
- **Calibration:** ⏳ รอ hardware (เดือนหน้า)
- **Build Phase:** ⏳ รอ calibration
- **Validation:** ⏳ 6 ข้อ (boot, lock, acquire, stream, mount, test)

---

**Document Status:** FROZEN — Ready for External Review
**Last Updated:** v1.4.3
**Next Milestone:** Hardware Calibration (เดือนหน้า)

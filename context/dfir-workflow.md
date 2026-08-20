# 📁 dfir-workflow.md — ลำดับงาน PHOENIX

## Phase 0 — Prep (ตอนนี้ → เดือนหน้า)
- เซฟ context files; จัดหา NVMe enclosure (RTL9210B), YubiKey, sacrificial disk
- ผ่า HDD 1TB lab drive

## Phase 1 — Project skeleton
- setup-project.sh สร้าง tree
- สร้าง keys: ECDSA manifest keypair, ed25519 field-ssh-key

## Phase 2 — VM build & boot test
- mkosi → mksquashfs → ukify → sbctl sign
- QEMU/OVMF: ตรวจ 3 boot entries

## Phase 3 — Invariant & write-block tests
- run-all.sh ด้วย TEST_DISK

## Phase 4 — Acquisition & streaming tests
- acquire.sh raw/e01 → assert hash

## Phase 5 — Hardware burn-in
- NVMe sustained write 100GB

## Field SOP (operator)
1. บูต EVIDENCE → YubiKey+PIN → ตรวจ banner
2. RAM ก่อน (avml) → ค่อย disk
3. sync → Export manifest.sig → shutdown

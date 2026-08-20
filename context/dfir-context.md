# 📁 dfir-context.md — PHOENIX Project Context Package
**Version 1.4.1 | Status: Design approved, awaiting hardware (next month)**
*Paste ไฟล์นี้เป็นข้อความแรกของแชทใหม่ เพื่อฟื้นฟู context ทั้งหมด*

## 0. วิธีใช้
- ผู้ช่วยต้องสวมบทบาท "PHOENIX co-architect" ต่อจากจุดนี้
- โปรเจกต์ยังไม่ field-proven — ห้ามเรียก production-ready จนกว่าจะผ่าน validation 6 ข้อ

## 1. Contract (ห้ามละเมิด)
> PHOENIX Field เป็นเครื่องมือ acquire/triage/recovery แบบ portable;
> Evidence Mode ต้องพิสูจน์ว่า source read-only และบันทึกความสมบูรณ์ได้;
> Recovery และ Triage ไม่อ้างสถานะ forensic acquisition

## 2. User Profile
- Daily driver: EndeavourOS + Hyprland + Quickshell + Zen Browser
- เครื่องหลัก: i5-4300U (2C/4T), RAM 4GB + zram 12G (โน้ตบุ๊กยุค 2013)
- มี: HDD 1TB ว่าง (จะทำ lab drive), เน็ตคณะแรง
- แผนเดือนหน้า: USB-C NVMe enclosure 1TB (ชิป RTL9210B), YubiKey 5, sacrificial disk
- สไตล์: Gray Man, ชอบคุยเชิง architecture ลึกๆ, ภาษาไทยคำเทคนิคอังกฤษ

## 3. หลักการแกนกลาง
- Minimize, Measure, Document, Justify (zero-touch เป็นไปไม่ได้ใต้ UEFI)
- Evidence ≠ Triage ≠ Recovery (แยก boot entry / banner / audit trail)
- Split profiles: Field=Debian bookworm, Lab=Fedora; Gentoo=static binary forge (เฟส 2, optional)
- External-first: NVMe=staging; network/external disk=preferred destination
- FIDO2 ห้ามผูก TPM2 (สื่อพกพา); passphrase fallback อยู่ในซองปิดผนึก
- Software write-blocker ≠ hardware write-blocker (คดีศาลต้องใช้ Tableau/WiebeTech)

## 4. Storage Layout (NVMe 1TB)
P1 ESP 1G FAT32 | P2 CORE 10G ext4-RO (core.squashfs) | P3 BRAIN 16G LUKS2→ext4 | P4 VAULT 120G LUKS2→Btrfs | P5 EVIDENCE ~850G LUKS2→Btrfs

## 5. Boot Chain & Modes
UEFI → systemd-boot → UKI (ukify) → dracut + phoenix hooks + dmsquash-live
- evidence: ro, rd.live.dir=/ rd.live.squashimg=core.squashfs rd.live.overlay=tmpfs
- triage: เหมือน evidence แต่ mode=triage (used-block เท่านั้น, ป้าย NON-PHYSICAL)
- recovery: rw, mode=recovery, warning 10 วิ, ไม่ล็อก

## 6. Write-Block Layers (4 ชั้น)
1. initramfs lock ก่อน userspace
2. boot set-equality assertion
3. fail-closed lock ณ acquisition
4. continuous guard + TERM→grace→KILL

## 7. Calibration
- ewfverify calibration-gated + human-in-the-loop
- UNCALIBRATED = ปฏิเสธการเซ็น manifest

## 8. Reviewer History
- ChatGPT Terra: architecture/boot-chain (retired — converge)
- Gemini Pro 3: operational reality (retired — converge)
- Claude Sonnet 5: TOCTOU/calibration (current)

## 9. Validation 6 ข้อ (ก่อน prototype-ready)
1. บูตได้บนเครื่องทดสอบ
2. ล็อก internal+hot-plug disk
3. whole-disk acquire ผ่าน invariants
4. stream one-read hash/bytes ตรง
5. mounts+FIDO/recovery flow ทำงาน
6. test suite รันบน sacrificial media เท่านั้น

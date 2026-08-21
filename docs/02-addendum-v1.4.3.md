# TODO: Stage 2 - content coming next month
# 📌 PHOENIX Addendum v1.4.3 — Calibration Script Fixes
**Status:** Converged with Sonnet 5 review
**Scope:** แก้ไข 3 จุดใน `calibrate-ewfverify.sh` ก่อน deploy

---

## 🔴 Bug #1: `dd conv=notrunc` ตำแหน่งกลางไฟล์

### ปัญหาเดิม
```bash
printf '\x00' | dd of=/tmp/corrupt.E01 bs=1 seek=$(( $(stat -c%s /tmp/sample.E01) / 2 )) conv=notrunc
```

**ความเสี่ยง:**
- E01 มี internal structure (section headers, chunk tables, CRC per chunk)
- ตำแหน่ง 50% อาจตกใน:
  - **padding/metadata** → ewfverify ตรวจไม่เจอ → calibration จำแนก failure mode ผิด
  - **section header** → ewfverify อ่านไฟล์ไม่ออกเลย (parse error) แทน hash mismatch
- Calibration script จะ capture "parse error" ผิดไปตีความเป็น "hash mismatch"

### การแก้ไข
ทำลาย data-plane เท่านั้น โดยเจาะใกล้ท้ายไฟล์ (มักเป็น data chunk สุดท้าย ไม่ใช่ header):
```bash
FSIZE=$(stat -c%s /tmp/sample.E01)
printf '\x00' | dd of=/tmp/corrupt.E01 bs=1 seek=$((FSIZE - 100)) conv=notrunc 2>/dev/null
```

---

## 🔴 Bug #2: n=1 ต่อ class — ไม่พอสำหรับตัดสิน pattern

### ปัญหาเดิม
- Calibration ทดสอบแค่ known-good 1 + known-corrupted 1
- ถ้า corrupted sample ดันตกในตำแหน่ง boundary ของ chunk พอดี → ผลกำกวม
- Mode ที่เลือกอาจไม่ robust พอสำหรับทุก case ในอนาคต

### การแก้ไข
รันอย่างน้อย 3 รอบด้วยตำแหน่งการทำลายข้อมูลที่ต่างกัน:
```bash
for pos in "$POS1" "$POS2" "$POS3"; do
    cp good.E01 "corrupt-$pos.E01"
    printf '\x00' | dd of="corrupt-$pos.E01" bs=1 seek="$pos" conv=notrunc status=none
done
```

ตำแหน่ง: 75%, 85%, ท้ายไฟล์-100 byte (โซนท้ายที่เป็น data chunk)

---

## 🔴 Bug #3: ไม่มี auto-decision logic ที่สมบูรณ์

### ปัญหาเดิม
```bash
echo "good_exit=$GOOD bad_exit=$BAD"
# เกณฑ์: ถ้า exit แยกได้ → exitcode; ...
```

**ความเสี่ยง:**
- Script แค่ print ตัวเลขให้มนุษย์อ่าน → ขัดกับหลัก "prove it, don't hope it"
- ถ้าจะ automate การตัดสินใจตอนนี้โดยไม่มีข้อมูลจริง → over-engineer + อาจเดา pattern ผิด

### การแก้ไข
**Human-in-the-loop บังคับรอบแรก:**
```bash
cat <<'EOF'
=== HUMAN DECISION REQUIRED (ห้ามข้าม) ===
อ่าน good.log + bad-*.logs ทุกไฟล์ก่อนเลือก:
 1) ทุก corrupted exit!=0 และ good exit==0        → พิมพ์ exitcode
 2) exit แยกไม่ได้ แต่มี verdict token ต่างกันชัด  → พิมพ์ text
 3) แยกไม่ได้ทั้งสองแบบ                            → พิมพ์ export-hash
EOF

read -r -p "โหมดที่จะเขียน (exitcode|text|export-hash|skip): " CHOICE
case "$CHOICE" in
  exitcode|text|export-hash)
      echo "$CHOICE" > /etc/phoenix/ewf-verify-mode
      echo "calibration: mode=$CHOICE chosen by $(whoami) at $(date -u +%FT%TZ)" \
          > "$WORK/calibration-record.txt"
      ;;
  *)
      echo "UNCALIBRATED" > /etc/phoenix/ewf-verify-mode
      echo "Skipped → UNCALIBRATED: การเซ็น manifest จะถูกปฏิเสธ"
      ;;
esac
```

---

## 🎯 สรุปการแก้ไข

| จุด | Before | After |
|-----|--------|-------|
| ตำแหน่งเจาะ | 50% (เสี่ยง parse error) | 75%/85%/ท้ายไฟล์ (data chunk) |
| Sample size | n=1 ต่อ class | n=3 corrupted samples |
| Decision | auto (เสี่ยงผิด pattern) | human-in-the-loop รอบแรก |
| Audit trail | ไม่มี | `calibration-record.txt` |

---

## ✅ Sign-off

**Sonnet 5:** ผ่านทั้ง 3 จุด แก้ตรงตาม engineering reality
**Qwen:** รับทั้ง 3 จุด ไม่มีข้อค้าน
**Owner:** ยืนยันว่าต้องแก้ก่อนเรียกว่า "เครื่องมือ calibration พร้อมใช้"

---

## 📎 Next Steps

1. Script ฉบับแก้แล้วอยู่ใน `build/calibrate-ewfverify.sh` (รอ populate เดือนหน้า)
2. เมื่อ hardware พร้อม → รัน calibration → human decision → deploy
3. ทุกครั้งที่เปลี่ยน libewf version → ต้อง calibrate ใหม่

---

**Document Status:** FROZEN
**Last Updated:** v1.4.3
**Dependencies:** Requires hardware (NVMe + YubiKey) for actual calibration

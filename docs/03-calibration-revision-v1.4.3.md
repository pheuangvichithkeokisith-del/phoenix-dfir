# TODO: Stage 2 - content coming next month
# 🛠️ PHOENIX Calibration Script v1.4.3 — Full Revision
**File:** `build/calibrate-ewfverify.sh`
**Status:** Final converged version (Sonnet 5 + Qwen)

---

## วัตถุประสงค์

Calibrate `ewfverify` ให้แยกแยะระหว่าง:
- **Known-good E01** (exit code + verdict token)
- **Known-corrupted E01** (hash mismatch ที่ตำแหน่งต่างกัน)

Output: `/etc/phoenix/ewf-verify-mode` = `exitcode` | `text` | `export-hash` | `UNCALIBRATED`

---

## Script ฉบับเต็ม

```bash
#!/bin/bash
# calibrate-ewfverify.sh — human-in-the-loop calibration
set -euo pipefail

# ========== Setup ==========
WORK=$(mktemp -d)
trap 'rm -rf "$WORK"' EXIT
cd "$WORK"

echo "📍 Working in: $WORK"
echo ""

# ========== Create known-good E01 ==========
echo "=== Creating known-good E01 from /dev/urandom ==="
dd if=/dev/urandom of=sample.raw bs=1M count=10 status=progress
ewfacquire -f encase7 -t sample -S 0 -C 1 -D "calibration" -M logical -d sha256 -u sample.raw 2>/dev/null

if [ ! -f sample.E01 ]; then
    echo "❌ Failed to create good.E01"
    exit 1
fi

# ========== Verify good ==========
echo "=== Verifying good.E01 ==="
ewfverify -c sha256 sample.E01 > good.log 2>&1 || true
GOOD_EXIT=$?

echo "Good exit code: $GOOD_EXIT"
cat good.log | grep -i "verify\|hash\|valid\|corrupt" || true
echo ""

# ========== Create corrupted samples (n=3) ==========
echo "=== Creating 3 corrupted samples ==="

FSIZE=$(stat -c%s sample.E01)
POS1=$(( FSIZE * 75 / 100 ))
POS2=$(( FSIZE * 85 / 100 ))
POS3=$(( FSIZE - 100 ))

cp sample.E01 corrupt-1.E01
cp sample.E01 corrupt-2.E01
cp sample.E01 corrupt-3.E01

printf '\x00' | dd of=corrupt-1.E01 bs=1 seek=$POS1 conv=notrunc 2>/dev/null
printf '\x00' | dd of=corrupt-2.E01 bs=1 seek=$POS2 conv=notrunc 2>/dev/null
printf '\x00' | dd of=corrupt-3.E01 bs=1 seek=$POS3 conv=notrunc 2>/dev/null

# ========== Verify corrupted samples ==========
echo "=== Verifying corrupted samples ==="

for i in 1 2 3; do
    echo "--- corrupt-$i.E01 ---"
    ewfverify -c sha256 "corrupt-$i.E01" > "bad-$i.log" 2>&1 || true
    BAD_EXIT=$?
    echo "Exit code: $BAD_EXIT"
    cat "bad-$i.log" | grep -i "verify\|hash\|valid\|corrupt\|mismatch" || true
    echo ""
done

# ========== Human decision ==========
echo ""
echo "=========================================="
echo "=== HUMAN DECISION REQUIRED (ห้ามข้าม) ==="
echo "=========================================="
echo ""
echo "อ่านไฟล์ทั้งหมดก่อนเลือก:"
echo "  good.log"
echo "  bad-1.log, bad-2.log, bad-3.log"
echo ""
echo "ไฟล์อยู่ใน: $WORK"
echo ""
echo "เกณฑ์การเลือก:"
echo "  1) ทุก corrupted exit!=0 และ good exit==0        → พิมพ์ 'exitcode'"
echo "  2) exit แยกไม่ได้ แต่มี verdict token ต่างกันชัด  → พิมพ์ 'text'"
echo "  3) แยกไม่ได้ทั้งสองแบบ                            → พิมพ์ 'export-hash'"
echo ""

read -r -p "โหมดที่จะเขียน (exitcode|text|export-hash|skip): " CHOICE

case "$CHOICE" in
    exitcode|text|export-hash)
        echo "$CHOICE" > /etc/phoenix/ewf-verify-mode
        echo "✅ Wrote mode: $CHOICE"
        
        # Audit trail
        mkdir -p /var/log/phoenix
        echo "calibration: mode=$CHOICE chosen by $(whoami) at $(date -u +%FT%TZ)" \
            > /var/log/phoenix/calibration-record.txt
        echo "📝 Audit log: /var/log/phoenix/calibration-record.txt"
        ;;
    *)
        echo "UNCALIBRATED" > /etc/phoenix/ewf-verify-mode
        echo "⚠️  Wrote: UNCALIBRATED"
        echo "⚠️  การเซ็น manifest จะถูกปฏิเสธจนกว่าจะ calibrate ใหม่"
        ;;
esac

echo ""
echo "✅ Calibration complete"
```

---

## 🎯 Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **n=3 corrupted samples** | ป้องกัน boundary case ที่ตำแหน่ง 50% อาจตกใน metadata |
| **ตำแหน่ง 75%/85%/ท้ายไฟล์** | หลีกเลี่ยง E01 headers (มักอยู่ต้นไฟล์) |
| **Human-in-the-loop** | libewf output format เปลี่ยนตาม version — ไม่สามารถ automate ได้ |
| **Audit trail** | `/var/log/phoenix/calibration-record.txt` สำหรับ accountability |
| **UNCALIBRATED = hard gate** | ถ้า skip → ปฏิเสธการเซ็น manifest ทุกกรณี |

---

## 📋 เมื่อไหร่ต้อง calibrate ใหม่?

1. **Initial setup** บน build host
2. **ทุกครั้งที่เปลี่ยน libewf version** (apt upgrade, rebuild image)
3. **ถ้า `phoenix-verify-e01` fail** แบบที่คาดไม่ถึง

---

## ⚠️ ข้อควรระวัง

- **ห้าม automate human decision step** — ถ้าจะ automate ต้องมี training data จริงหลายร้อย samples
- **ห้าม cache mode นานเกินไป** — libewf อาจเปลี่ยน behavior ใน minor version
- **Operator ต้องอ่าน log จริงๆ** — ไม่ใช่แค่กด enter ผ่าน

---

**Document Status:** FROZEN
**Last Updated:** v1.4.3
**Dependencies:** Requires `ewftools` installed on build host

# Linux Hardening & Security Audit Tool

Home lab hardening Ubuntu Server berbasis CIS Benchmark, plus audit tool Bash untuk mengecek compliance secara otomatis.

---

## Deskripsi

Project ini mempelajari struktur filesystem Linux (FHS), menerapkan hardening berbasis CIS Benchmark secara manual pada Ubuntu Server 24.04 LTS di VirtualBox, lalu membangun audit tool Bash yang mengecek compliance (PASS/FAIL + rekomendasi). Hasil akhir yang ditargetkan: audit tool, laporan before/after hardening, dan writeup. Status saat ini: environment dan pembelajaran FHS selesai, hardening manual sedang berjalan, audit script belum dimulai.

## Tech Stack & Tools

- VirtualBox: virtualisasi untuk VM target hardening
- Ubuntu Server 24.04.4 LTS: sistem target (custom partitioning, tanpa GUI)
- OpenSSH: akses remote dari host Windows (NAT + Host-only)
- Bash: audit script (8 check function dari kerangka awal, ditambah check lain yang direncanakan)
- Git/GitHub: version control dan dokumentasi before/after
- Python (opsional, planning): report generator dengan output HTML
- auditd, AIDE, ufw (planning): logging, integrity checking, firewall di Fase 3

## Struktur Folder

```
linux-hardening-audit/
├── README.md
├── writeups/                       (planning)
├── scripts/
│   └── linux-hardening-audit.sh    (planning, kerangka sudah ada)
├── checklist/
│   └── cis-benchmark-checklist.md
└── docs/
    ├── screenshots/
    │   ├── fase1/
    │   ├── fase2/
    │   ├── fase3/
    │   └── fase4/
    ├── notes/
    └── reports/                    (audit report before/after, planning)
```

## Progress

| Fase | Status |
|---|---|
| Fase 1: Setup Environment | ✅ Selesai |
| Fase 2: Pembelajaran FHS | ✅ Selesai (sisa yang opsional: `/usr`, `/dev`, `man hier`) |
| Fase 3: Hardening Manual (CIS) | 🔄 Berjalan (mount option `/tmp` selesai, `/dev/shm` berikutnya) |
| Fase 4: Build Audit Script | ⬜ Belum |
| Fase 5: Dokumentasi / Writeup | ⬜ Belum |

<!-- **Progress keseluruhan: 14%** (21 dari 151 item pada checklist; detail di `project-snapshot-linux-hardening-audit-2026-10-23.md`) -->
**Progress keseluruhan: 14%** (21 dari 151 item pada checklist)

## Dokumentasi

- Checklist CIS lengkap: [`checklist/cis-benchmark-checklist.md`](checklist/cis-benchmark-checklist.md)
- Catatan belajar dan troubleshooting: folder `docs/notes/`
- Bukti per langkah: `docs/screenshots/fase1` sampai `fase3`
<!-- - Writeup lengkap (`writeup.md`) masih disusun, akan menyusul setelah Fase 3 sampai 5 selesai -->

## Disclaimer

Project ini dibuat untuk tujuan pembelajaran dan portofolio. Semua konfigurasi dan simulasi (termasuk percobaan login SSH gagal) dijalankan hanya di VM lab milik sendiri. Jangan jalankan script atau teknik dari project ini di sistem yang bukan milik sendiri atau tanpa izin tertulis. Rekomendasi hardening mengacu pada CIS Benchmark yang disederhanakan dan bukan pengganti audit resmi.

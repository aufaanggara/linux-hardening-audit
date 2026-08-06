# CIS Benchmark Checklist — Ubuntu Server Hardening

> Checklist ini dipakai sebagai acuan untuk Fase 3 (Hardening Manual) dan Fase 4 (Audit Script).
> Centang `[x]` setiap item setelah diterapkan dan diverifikasi manual, lalu catat command yang dipakai di `docs/notes/`.

---

## A. Filesystem & Initial Setup

- [x] Partisi terpisah untuk `/tmp`, `/var`, `/var/log`, `/home` — **selesai di Fase 1** (root 15G, swap 4G, tmp 3G, var 5G, var/log 5G, home 5G)
- [ ] `nodev`, `nosuid`, `noexec` diset pada `/tmp` di `/etc/fstab`
- [ ] AIDE terinstall untuk filesystem integrity checking

## B. Services

- [ ] Disable service yang tidak perlu: `avahi-daemon`, `cups`, `isc-dhcp-server`, `nfs-server`, `rpcbind`, `samba`, `xinetd`
- [ ] Cek service aktif: `systemctl list-units --type=service --state=running`

## C. Network Configuration

- [ ] Disable IP forwarding: `net.ipv4.ip_forward = 0`
- [ ] Disable ICMP redirects: `net.ipv4.conf.all.accept_redirects = 0`
- [ ] Enable TCP SYN cookies: `net.ipv4.tcp_syncookies = 1`
- [ ] Disable IPv6 kalau tidak dipakai

## D. Logging & Auditing

- [ ] `auditd` terinstall dan enabled
- [ ] `rsyslog` terinstall dan enabled
- [ ] Logrotate dikonfigurasi

## E. Access, Authentication, Authorization

- [ ] `cron` dibatasi hanya untuk user tertentu (`/etc/cron.allow`)
- [ ] SSH hardening: `PermitRootLogin no`, `PasswordAuthentication no`, `MaxAuthTries 4`, `LogLevel VERBOSE`, banner, idle timeout
- [ ] PAM password policy: minimum length, complexity, history (`pam_pwquality`)
- [ ] Password expiration: `PASS_MAX_DAYS`, `PASS_MIN_DAYS`, `PASS_WARN_AGE` di `/etc/login.defs`
- [ ] Default `umask` diset ke 027 atau lebih ketat
- [ ] Root `PATH` tidak mengandung direktori writable oleh user lain

## F. System Maintenance / File Permissions

- [x] Permission `/etc/passwd` = 644, owner root:root — **terverifikasi di Fase 2** (`-rw-r--r-- root root`)
- [x] Permission `/etc/shadow` = 640, owner root:shadow — **terverifikasi di Fase 2** (`-rw-r----- root shadow`)
- [ ] Tidak ada duplicate UID/GID
- [ ] Hanya `root` yang punya UID 0
- [ ] Tidak ada file world-writable tanpa sticky bit di direktori umum

---

## Progress Log

| Tanggal | Kategori | Item | Status | Catatan |
|---|---|---|---|---|
| 2026-07-31 | A | Custom partitioning | ✅ Selesai | 6 partisi terpisah, lihat `docs/screenshots/fase1/` |
| 2026-08-01 | F | Cek permission passwd/shadow | ✅ Terverifikasi | Lihat `docs/screenshots/fase2/00-permission-passwd-vs-shadow.png` |
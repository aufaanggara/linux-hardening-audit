# CIS Benchmark Checklist: Linux Hardening & Security Audit Tool

Terakhir diupdate: 21 Oktober 2026

Dokumen hidup untuk Fase 3 (Hardening Manual) dan Fase 4 (Audit Script). Simpan di `checklist/cis-benchmark-checklist.md`.

Legenda:

- `[x]` = selesai dan terverifikasi (ada bukti screenshot atau output)
- `[ ]` = belum dikerjakan atau belum dikonfirmasi
- Item tanpa tag = dari checklist awal, dipertahankan persis
- Item bertanda **(tambahan)** = ditambahkan karena relevan dengan CIS Benchmark Ubuntu, dengan penjelasan singkat bila perlu
- Setiap item `[x]` dicatat bukti dan tanggalnya di Progress Log (bagian paling bawah)

---

## Ringkasan Progress Hardening (Kategori A sampai F)

| Kategori | Selesai | Total item | Asli / tambahan |
|---|---|---|---|
| A. Filesystem & Initial Setup | 2 | 18 | 3 / 15 |
| B. Services | 0 | 7 | 2 / 5 |
| C. Network Configuration | 0 | 16 | 4 / 12 |
| D. Logging & Auditing | 0 | 12 | 3 / 9 |
| E. Access, Authentication, Authorization | 0 | 28 | 6 / 22 |
| F. System Maintenance / File Permissions | 2 | 15 | 5 / 10 |
| **Total** | **4** | **96** | **23 / 73** |

- Progress: 4 dari 96 item (4%)
- Item asli dari checklist awal: 23; item tambahan: 73
- Item `[x]` di kategori F adalah verifikasi baseline (sudah sesuai sejak instalasi), bukan hasil perubahan
- Item berikutnya: `/dev/shm` `noexec`, lalu AIDE

---

## Tujuan Project

Membangun pemahaman mendalam tentang struktur filesystem Linux (FHS), lalu menerapkan hardening berbasis CIS Benchmark, dan menghasilkan tool (Bash script) yang bisa mengaudit compliance sistem secara otomatis. Output akhirnya bukan cuma "paham Linux", tapi ada artifact nyata: **audit tool + laporan before/after hardening + writeup**.

---

## Tech Stack

- **VirtualBox**: Ubuntu Server 22.04/24.04 LTS (target hardening). Yang dipakai: Ubuntu Server 24.04.4 LTS
- **Bash** untuk audit script
- **Python** (opsional, kalau mau bikin report generator yang lebih rapi / HTML output)
- **Git** untuk dokumentasi & version control (before/after)

---

## Struktur Fase

### Fase 1: Setup Environment (selesai)

- [x] Install Ubuntu Server di VBox (minimal install, no GUI). Catatan: dipakai instalasi "Ubuntu Server" standar, bukan "minimized", agar baseline lengkap
- [x] Setup networking (bridged/NAT), SSH access. Catatan: dipakai NAT + Host-only (dual adapter), SSH dari Windows host berhasil
- [x] Snapshot VM di kondisi fresh install (buat baseline sebelum hardening): `fresh-install-before-hardening`
- [x] Custom partitioning: `/` 15G, swap 4G, `/tmp` 3G, `/var` 5G, **(tambahan)** `/var/log` 5G, `/home` 5G
- [x] **(tambahan)** Struktur folder project dibuat dan di-push ke Git
- [ ] **(tambahan)** Akar masalah VM stuck di "Loading essential drivers" belum terkonfirmasi; workaround GRUB `e` lalu `F10`

### Fase 2: Pembelajaran FHS (Filesystem Hierarchy Standard) (selesai)

Jangan cuma `cd` manual satu-satu. Pelajari dengan kerangka:

- [x] Apa fungsi tiap direktori top-level (`/etc`, `/var`, `/tmp`, `/usr`, `/bin`, `/sbin`, `/lib`, `/opt`, `/proc`, `/sys`, `/dev`, `/home`, `/root`, `/boot`, `/mnt`, `/media`, `/srv`)
  - [x] `/etc` (lewat `passwd`, `shadow`, `fstab`)
  - [x] `/var` dan `/var/log` (`auth.log`, `last`, `lastb`)
  - [x] `/tmp` (permission 1777, sticky bit, `noexec`)
  - [x] `/bin`, `/sbin`, `/lib` (symlink merged-usr)
  - [x] `/opt`, `/mnt`, `/media`, `/srv` (kosong, baseline)
  - [x] `/proc`, `/sys`
  - [x] `/home`, `/root`, `/boot`
  - [ ] `/usr` dan `/dev` (baru sebagian, dibahas lewat symlink dan `/dev/shm`)
- [x] Kenapa itu relevan buat blue team: `/tmp` dan `/dev/shm` sering dipakai attacker buat staging payload karena writable & world-accessible; `/var/log` isinya log yang perlu diketahui lokasinya buat investigasi; `/etc/passwd` vs `/etc/shadow` beda fungsi & risk-nya
- [ ] Referensi resmi: `man hier` (built-in di Linux) dan FHS spec dari Linux Foundation (belum dibaca)

### Fase 3: Hardening Manual (berjalan)

Lakukan satu per satu secara manual dulu sebelum bikin script, supaya paham konsep di balik tiap konfigurasi, bukan cuma copy-paste command.

- Urutan mengikuti kategori A sampai F di bawah
- Setiap item: baseline (sebelum), konfigurasi, verifikasi, screenshot (sesudah)
- Backup file konfigurasi sebelum diedit; snapshot VM sebagai jalan kembali
- Uji perubahan SSH dari sesi kedua sebelum menutup sesi lama

### Fase 4: Build Audit Script (belum dimulai)

Bikin Bash script yang mengecek compliance tiap item di checklist, output PASS/FAIL + rekomendasi. Kerangka ada di bagian "Kerangka Audit Script" di bawah. Pemetaan item checklist ke check function ada di bagian "Pemetaan Check Function".

### Fase 5: Dokumentasi / Writeup (belum dimulai)

- [ ] Screenshot before/after tiap kategori hardening
- [ ] Command yang dipakai + penjelasan kenapa
- [ ] Hasil run audit script (before vs after hardening)
- [ ] Push ke GitHub repo
- [ ] **(tambahan)** `README.md` repo dan `writeup.md` (apa yang dipelajari, kesulitan, temuan menarik)
- [ ] **(tambahan)** Bagian troubleshooting jujur: yang terbukti vs yang masih dugaan

---
## CIS Benchmark Checklist (Ubuntu, simplified + tambahan)

### A. Filesystem & Initial Setup

- [x] Partisi terpisah untuk `/tmp`, `/var`, `/var/log`, `/home`
- [x] `nodev`, `nosuid`, `noexec` diset pada `/tmp` di `/etc/fstab`
- [ ] AIDE terinstall untuk filesystem integrity checking

Tambahan:

- [ ] **(tambahan)** `/dev/shm` di-mount dengan `nodev`, `nosuid`, `noexec`. Baseline: `rw,nosuid,nodev,inode64` (tanpa `noexec`); langkah sudah diberikan, hasil belum dikonfirmasi
- [ ] **(tambahan)** `nodev` dan `nosuid` pada `/var` dan `/home`; `nodev`, `nosuid`, `noexec` pada `/var/log`
- [ ] **(tambahan)** Partisi terpisah untuk `/var/tmp` dan `/var/log/audit` (opsional; `/var/tmp` ikut dikunci seperti `/tmp`)
- [ ] **(tambahan)** Database AIDE diinisialisasi dan pengecekan dijadwalkan (cron atau systemd timer)
- [ ] **(tambahan)** Filesystem yang tidak dipakai dinonaktifkan lewat modprobe: `cramfs`, `freevxfs`, `hfs`, `hfsplus`, `jffs2`, `udf`. `squashfs` dikecualikan selama `snap` dipakai
- [ ] **(tambahan)** `usb-storage` di-blacklist (mencegah media removable di server)
- [ ] **(tambahan)** `autofs` tidak terinstall atau dinonaktifkan (cegah automount)
- [ ] **(tambahan)** Permission `/boot/grub/grub.cfg` dibatasi (root, tidak bisa dibaca group/other); password GRUB opsional untuk VM lab
- [ ] **(tambahan)** ASLR aktif: `kernel.randomize_va_space = 2`
- [ ] **(tambahan)** Core dump dibatasi: `fs.suid_dumpable = 0` dan `* hard core 0` di `limits.conf`
- [ ] **(tambahan)** `kernel.yama.ptrace_scope` minimal 1
- [ ] **(tambahan)** AppArmor aktif dan profile dalam mode enforce
- [ ] **(tambahan)** Banner `/etc/issue`, `/etc/issue.net`, `/etc/motd` tidak membocorkan versi OS
- [ ] **(tambahan)** Paket dan security update terbaru (`apt update && apt upgrade`); baseline: 34 update tertunda; `unattended-upgrades` aktif
- [ ] **(tambahan)** Time synchronization aktif (`systemd-timesyncd` atau `chrony`), penting agar timestamp log konsisten

### B. Services

- [ ] Disable service yang tidak perlu: `avahi-daemon`, `cups`, `isc-dhcp-server`, `nfs-server`, `rpcbind`, `samba`, `xinetd`
- [ ] Cek service aktif: `systemctl list-units --type=service --state=running`

Tambahan:

- [ ] **(tambahan)** Simpan baseline daftar service yang berjalan sebelum hardening (untuk perbandingan before/after)
- [ ] **(tambahan)** Tidak terinstall atau disabled bila tidak dipakai: `slapd`, `snmpd`, `vsftpd`, `telnetd`, `tftpd`, `dovecot`, `squid`, `bind9`, `apache2`, `nginx`, `nis`, `rsync` (service), `dnsmasq`
- [ ] **(tambahan)** MTA (misal `postfix`) hanya listen di localhost, bila terinstall
- [ ] **(tambahan)** `xserver-xorg` tidak terinstall (server tanpa GUI)
- [ ] **(tambahan)** Tinjau kebutuhan `snapd` dan `lxd`

### C. Network Configuration

- [ ] Disable IP forwarding: `net.ipv4.ip_forward = 0` (baseline sudah `0`, tinggal didokumentasikan)
- [ ] Disable ICMP redirects: `net.ipv4.conf.all.accept_redirects = 0`
- [ ] Enable TCP SYN cookies: `net.ipv4.tcp_syncookies = 1`
- [ ] Disable IPv6 kalau tidak dipakai

Tambahan:

- [ ] **(tambahan)** `net.ipv4.conf.default.accept_redirects = 0` dan `net.ipv6.conf.all.accept_redirects = 0`
- [ ] **(tambahan)** `net.ipv4.conf.all.send_redirects = 0` dan `net.ipv4.conf.default.send_redirects = 0`
- [ ] **(tambahan)** `net.ipv4.conf.all.secure_redirects = 0` dan `net.ipv4.conf.default.secure_redirects = 0`
- [ ] **(tambahan)** `net.ipv4.conf.all.accept_source_route = 0` (default dan IPv6 juga)
- [ ] **(tambahan)** `net.ipv4.conf.all.log_martians = 1` (paket dengan alamat mustahil dicatat)
- [ ] **(tambahan)** `net.ipv4.icmp_echo_ignore_broadcasts = 1` dan `net.ipv4.icmp_ignore_bogus_error_responses = 1`
- [ ] **(tambahan)** `net.ipv4.conf.all.rp_filter = 1` (anti IP spoofing)
- [ ] **(tambahan)** `net.ipv6.conf.all.accept_ra = 0` bila IPv6 dipakai
- [ ] **(tambahan)** Setting sysctl dibuat permanen di `/etc/sysctl.d/*.conf`, bukan hanya `sysctl -w` (hilang saat reboot)
- [ ] **(tambahan)** Firewall host (`ufw`) aktif: default deny incoming, izinkan SSH dari jaringan Host-only saja. Pastikan aturan SSH ada sebelum firewall diaktifkan
- [ ] **(tambahan)** Protokol jarang dipakai dinonaktifkan: `dccp`, `sctp`, `rds`, `tipc`
- [ ] **(tambahan)** Port yang listening ditinjau (`ss -tulpn`); yang terbuka hanya yang dibutuhkan (SSH)

### D. Logging & Auditing

- [ ] `auditd` terinstall dan enabled
- [ ] `rsyslog` terinstall dan enabled
- [ ] Logrotate dikonfigurasi

Tambahan:

- [ ] **(tambahan)** Rule auditd untuk: perubahan waktu sistem, file identitas (`/etc/passwd`, `/etc/group`, `/etc/shadow`, `/etc/gshadow`), `/etc/sudoers`, `/etc/ssh/sshd_config`, login/logout (`lastlog`, `faillog`), sesi (`utmp`, `wtmp`, `btmp`), perubahan mount, penghapusan file, pemuatan modul kernel, perubahan permission dan ownership
- [ ] **(tambahan)** Rule auditd khusus blue team: eksekusi dari direktori writable (`/tmp`, `/dev/shm`) dan perubahan `/etc/fstab`
- [ ] **(tambahan)** Konfigurasi auditd: `max_log_file`, `max_log_file_action`, `space_left_action`, `admin_space_left_action`; parameter kernel `audit=1`
- [ ] **(tambahan)** Audit config immutable (`-e 2`) di akhir rules (opsional; ubah rule butuh reboot)
- [ ] **(tambahan)** Permission file di `/var/log` dan `/var/log/audit` dibatasi (umumnya 0640 atau lebih ketat)
- [ ] **(tambahan)** `journald` persistent dan terkompresi
- [ ] **(tambahan)** Retensi `auth.log` cukup panjang (log lama pindah ke `auth.log.1`; bukti brute-force tidak boleh hilang cepat)
- [ ] **(tambahan)** Pengiriman log ke remote/SIEM (opsional, lanjutan: Wazuh atau Splunk di lab yang sudah ada)
- [ ] **(tambahan)** Hasil pengecekan AIDE tercatat di log

### E. Access, Authentication, Authorization

- [ ] `cron` dibatasi hanya untuk user tertentu (`/etc/cron.allow`)
- [ ] SSH hardening: `PermitRootLogin no`, `PasswordAuthentication no`, `MaxAuthTries 4`, `LogLevel VERBOSE`, banner, idle timeout
- [ ] PAM password policy: minimum length, complexity, history (`pam_pwquality`)
- [ ] Password expiration: `PASS_MAX_DAYS`, `PASS_MIN_DAYS`, `PASS_WARN_AGE` di `/etc/login.defs`
- [ ] Default `umask` diset ke 027 atau lebih ketat
- [ ] Root `PATH` tidak mengandung direktori writable oleh user lain

Tambahan, cron dan at:

- [ ] **(tambahan)** Permission `/etc/crontab` dan `/etc/cron.hourly`, `cron.daily`, `cron.weekly`, `cron.monthly`, `cron.d` dimiliki root dan tidak bisa diakses group/other
- [ ] **(tambahan)** `at` dibatasi (`/etc/at.allow`) atau tidak terinstall

Tambahan, SSH:

- [ ] **(tambahan)** SSH key dibuat dan **diuji login-nya** sebelum `PasswordAuthentication no` diterapkan, supaya tidak terkunci
- [ ] **(tambahan)** Permission `/etc/ssh/sshd_config` (0600 root) dan private host key (0600)
- [ ] **(tambahan)** `PermitEmptyPasswords no`, `HostbasedAuthentication no`, `IgnoreRhosts yes`, `PermitUserEnvironment no`
- [ ] **(tambahan)** `X11Forwarding no` dan `AllowTcpForwarding no` (atau `DisableForwarding yes`)
- [ ] **(tambahan)** Idle timeout: `ClientAliveInterval` dan `ClientAliveCountMax`; plus `LoginGraceTime 60`, `MaxSessions`, `MaxStartups`
- [ ] **(tambahan)** `AllowUsers` atau `AllowGroups` membatasi siapa yang boleh login SSH
- [ ] **(tambahan)** Algoritma kuat (`Ciphers`, `MACs`, `KexAlgorithms`) sesuai CIS
- [ ] **(tambahan)** Banner SSH diarahkan ke `/etc/issue.net`
- [ ] **(tambahan)** Config divalidasi dengan `sudo sshd -t` sebelum reload; uji dari sesi kedua sebelum menutup sesi lama

Tambahan, PAM dan password:

- [ ] **(tambahan)** Lockout akun setelah gagal login (`pam_faillock`, misal `deny=5`, `unlock_time=900`)
- [ ] **(tambahan)** `pam_pwhistory` (`remember=24`) selain `pam_pwquality`
- [ ] **(tambahan)** Algoritma hash password kuat (`yescrypt` atau `sha512`)
- [ ] **(tambahan)** `INACTIVE` di `useradd -D` maksimal 30 hari
- [ ] **(tambahan)** Tidak ada akun dengan password kosong

Tambahan, sudo, su, dan sesi:

- [ ] **(tambahan)** `sudo` memakai `use_pty` dan `logfile`, `timestamp_timeout` dibatasi
- [ ] **(tambahan)** `su` dibatasi ke grup tertentu (`pam_wheel`)
- [ ] **(tambahan)** Anggota grup `sudo` hanya user yang berhak. Baseline: hanya `linuxaufa`. Tinjau juga grup `lxd` (anggotanya dapat dipakai untuk eskalasi ke root) dan `adm` (akses baca log)
- [ ] **(tambahan)** Timeout shell interaktif (`TMOUT=900`, readonly). Terkait sesi `tty1` yang menganggur di baseline
- [ ] **(tambahan)** Akun sistem memakai `nologin` dan terkunci
- [ ] **(tambahan)** Home directory user 750 atau lebih ketat; dot file tidak writable group/other; tidak ada `.netrc`, `.forward`, `.rhosts`

### F. System Maintenance / File Permissions

- [x] Permission `/etc/passwd` = 644, owner root:root (baseline terverifikasi)
- [x] Permission `/etc/shadow` = 640/000, owner root:shadow (baseline terverifikasi: 640)
- [ ] Tidak ada duplicate UID/GID
- [ ] Hanya `root` yang punya UID 0
- [ ] Tidak ada file world-writable tanpa sticky bit di direktori umum

Tambahan:

- [ ] **(tambahan)** Permission `/etc/group` dan `/etc/shells` = 644; `/etc/passwd-` = 644; `/etc/group-` = 644; `/etc/shadow-`, `/etc/gshadow`, `/etc/gshadow-` = 640 root:shadow
- [ ] **(tambahan)** Tidak ada duplicate username dan group name
- [ ] **(tambahan)** Semua GID di `/etc/passwd` ada di `/etc/group`
- [ ] **(tambahan)** Grup `shadow` tidak punya anggota
- [ ] **(tambahan)** Tidak ada file atau direktori tanpa owner atau group (`-nouser`, `-nogroup`)
- [ ] **(tambahan)** File SUID/SGID ditinjau dan baseline-nya dicatat
- [ ] **(tambahan)** File world-writable (bukan direktori sticky) ditinjau
- [ ] **(tambahan)** Home directory semua user interaktif ada dan dimiliki user yang benar
- [ ] **(tambahan)** Tidak ada entri legacy `+` di `/etc/passwd`, `/etc/shadow`, `/etc/group`
- [ ] **(tambahan)** Pengecekan `find` memakai path asli (`/usr/bin`) atau `-L`, karena `/bin`, `/sbin`, `/lib` adalah symlink (hindari false negative)

---
## Pemetaan Check Function (Fase 4)

Function dari kerangka awal (diisi sendiri dulu, minta hint hanya kalau stuck):

- [ ] `check_ssh_root_login` (kategori E)
- [ ] `check_ssh_password_auth` (kategori E)
- [ ] `check_unnecessary_services` (kategori B)
- [ ] `check_ip_forwarding` (kategori C)
- [ ] `check_passwd_permissions` (kategori F)
- [ ] `check_shadow_permissions` (kategori F)
- [ ] `check_password_policy` (kategori E)
- [ ] `check_auditd_status` (kategori D)

Function tambahan yang diusulkan:

- [ ] **(tambahan)** `check_tmp_mount_options` (A): `nodev`, `nosuid`, `noexec` di `/tmp`
- [ ] **(tambahan)** `check_dev_shm_mount_options` (A): `noexec` di `/dev/shm`
- [ ] **(tambahan)** `check_aide_installed` (A)
- [ ] **(tambahan)** `check_sysctl_network` (C): redirects, source route, SYN cookies, `rp_filter`, `log_martians`
- [ ] **(tambahan)** `check_firewall_status` (C)
- [ ] **(tambahan)** `check_rsyslog_status` dan `check_logrotate` (D)
- [ ] **(tambahan)** `check_cron_permissions` (E)
- [ ] **(tambahan)** `check_umask` dan `check_root_path` (E)
- [ ] **(tambahan)** `check_sudo_group_members` (E): anggota grup `sudo` dan `lxd`
- [ ] **(tambahan)** `check_ssh_bruteforce_log` (D/E): hitung `Failed password`, `Invalid user`, `authentication failure` per IP, hanya dari proses `sshd`, dengan threshold
- [ ] **(tambahan)** `check_uid_zero` dan `check_duplicate_uid_gid` (F)
- [ ] **(tambahan)** `check_world_writable_files` (F): path asli atau `-L`, bukan lewat symlink
- [ ] **(tambahan)** `check_aslr` (A)

Catatan desain untuk semua function:

- Hindari false positive: jejak `sudo grep` ikut tercatat di `auth.log`; saring ke proses `sshd`
- Hindari false negative: satu pola `grep` tidak cukup; `lastb` jadi pembanding
- Log berotasi (`auth.log.1`) dan timestamp memakai UTC
- Pastikan pengecekan benar-benar memeriksa sesuatu (hasil kosong tidak sama dengan PASS)

---

## Kerangka Audit Script (Bash)

```bash
#!/bin/bash
# linux-hardening-audit.sh
# Kerangka audit tool - isi logic tiap check function sendiri

REPORT_FILE="audit_report_$(date +%Y%m%d_%H%M%S).txt"
PASS_COUNT=0
FAIL_COUNT=0

log_result() {
    local check_name="$1"
    local status="$2"
    local recommendation="$3"

    echo "[$status] $check_name" | tee -a "$REPORT_FILE"
    if [ "$status" == "FAIL" ]; then
        echo "    -> Rekomendasi: $recommendation" | tee -a "$REPORT_FILE"
        ((FAIL_COUNT++))
    else
        ((PASS_COUNT++))
    fi
}

check_ssh_root_login() {
    # TODO: cek isi /etc/ssh/sshd_config, cari "PermitRootLogin"
    :
}

check_ssh_password_auth() {
    # TODO: cek "PasswordAuthentication" di sshd_config
    :
}

check_unnecessary_services() {
    # TODO: loop daftar service yang seharusnya disabled
    :
}

check_ip_forwarding() {
    # TODO: cek `sysctl net.ipv4.ip_forward`
    :
}

check_passwd_permissions() {
    # TODO: cek permission /etc/passwd pakai `stat -c "%a %U:%G" /etc/passwd`
    :
}

check_shadow_permissions() {
    # TODO: sama seperti di atas untuk /etc/shadow
    :
}

check_password_policy() {
    # TODO: cek PASS_MAX_DAYS, PASS_MIN_DAYS di /etc/login.defs
    :
}

check_auditd_status() {
    # TODO: cek `systemctl is-enabled auditd`
    :
}

main() {
    echo "=== Linux Hardening Audit Report ===" | tee "$REPORT_FILE"
    echo "Tanggal: $(date)" | tee -a "$REPORT_FILE"
    echo "" | tee -a "$REPORT_FILE"

    check_ssh_root_login
    check_ssh_password_auth
    check_unnecessary_services
    check_ip_forwarding
    check_passwd_permissions
    check_shadow_permissions
    check_password_policy
    check_auditd_status

    echo "" | tee -a "$REPORT_FILE"
    echo "=== Summary: $PASS_COUNT PASS, $FAIL_COUNT FAIL ===" | tee -a "$REPORT_FILE"
}

main
```

Catatan cara isi tiap function: pola umum `grep` konfigurasi dari file terkait, bandingkan value dengan expected, lalu panggil `log_result`. Coba isi 2-3 function dulu sendiri, kalau stuck baru minta hint.

---

## Output Akhir yang Diharapkan

1. Repo GitHub berisi: script, checklist markdown, report before/after
2. Writeup: apa yang dipelajari, kesulitan, temuan menarik
3. (Opsional) Versi Python dengan HTML output report

---

## Baseline Sebelum Hardening

Kondisi awal yang sudah tercatat (bahan laporan "before"):

| Item | Kondisi awal | Bukti |
|---|---|---|
| `/tmp` mount option | `rw,relatime` | `fase2/23` |
| `/dev/shm` | `rw,nosuid,nodev,inode64`, script bisa dijalankan | `fase2/27`, `fase2/28` |
| `ip_forward` | `0` (sudah aman) | `fase2/21` |
| `/etc/passwd`, `/etc/shadow` | 644 root:root, 640 root:shadow | `fase2/00` |
| Anggota grup `sudo` | hanya `linuxaufa` | `fase2/02` |
| Grup user `linuxaufa` | `adm`, `cdrom`, `sudo`, `dip`, `plugdev`, `lxd` | `fase2/01` |
| SSH | password authentication aktif | `fase1/21` |
| Sesi console `tty1` | menganggur sejak login | `fase2/08`, `fase2/11` |
| Update tertunda | 34 paket | `fase1/24` |

---

## Progress Log

| Tanggal | Item | Status | Bukti |
|---|---|---|---|
| 2026-07-31 | A: partisi terpisah | Selesai | `fase1/17`, `fase2/22` |
| 2026-08-01 | F: permission passwd dan shadow | Terverifikasi (baseline) | `fase2/00` |
| 2026-10 | A: `/tmp` `nodev,nosuid,noexec` | Selesai | `fase3/00`, `fase3/01` |
| (menunggu) | A: `/dev/shm` `noexec` | Langkah diberikan | `fase3/02`, `fase3/03` |

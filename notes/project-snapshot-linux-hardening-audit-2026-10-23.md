# Project Snapshot: Linux Hardening & Security Audit Tool

> Deskripsi project untuk context LinkedIn, CV, resume, atau web portofolio. Hanya memuat yang sudah dikerjakan dan dikonfirmasi; yang masih rencana ditandai jelas.

**Nama Project:** Linux Hardening & Security Audit Tool (Ubuntu Server 24.04, CIS Benchmark)  
**Periode:** Juli 2026 (mulai 31 Juli) sampai sekarang, masih berjalan (sempat jeda Agustus sampai Oktober 2026)  
**Kategori:** Blue Team / Linux Hardening / Security Auditing  
**Stack/Tools:** VirtualBox, Ubuntu Server 24.04.4 LTS, OpenSSH, Bash, Git/GitHub, PowerShell (host Windows 11), nano. Direncanakan: auditd, AIDE, ufw, Python (opsional)

**Deskripsi:**  
Home lab untuk mempelajari struktur filesystem Linux (FHS) dan menerapkan hardening berbasis CIS Benchmark pada Ubuntu Server 24.04 LTS di VirtualBox, sambil membangun audit tool berbasis Bash yang mengecek compliance secara otomatis (PASS/FAIL + rekomendasi). Hasil akhir yang ditargetkan adalah audit tool, laporan before/after hardening, dan writeup. Sampai saat ini environment, pembelajaran FHS, dan hardening pertama (mount option `/tmp`) sudah dikerjakan dan terdokumentasi; audit script masih tahap perencanaan.

**Yang sudah dikerjakan:**
- Menyiapkan VM Ubuntu Server 24.04.4 LTS (4 GB RAM, 2 vCPU, disk 40 GB) dengan dua network adapter (NAT dan Host-only) dan akses SSH dari host Windows
- Membuat custom partitioning 6 partisi terpisah (`/`, swap, `/tmp`, `/var`, `/var/log`, `/home`) supaya mount option bisa diatur per partisi
- Membuat snapshot baseline `fresh-install-before-hardening` sebagai titik kembali sebelum hardening
- Mempelajari FHS secara hands-on: `/etc/passwd` vs `/etc/shadow`, `/var/log`, `/proc`, `/sys`, `/boot`, `tmpfs`, `/etc/fstab`, symlink merged-usr
- Menganalisis log autentikasi (`auth.log`, `last`, `lastb`): mensimulasikan login SSH gagal di VM sendiri, menemukan bahwa satu pola `grep` bisa melewatkan event (`Invalid user`) dan menghasilkan false positive dari jejak `sudo`
- Mencatat kondisi baseline sebelum hardening (mount option `/tmp` dan `/dev/shm`, `ip_forward`, permission file sensitif, anggota grup `sudo`)
- Menerapkan dan memverifikasi hardening pertama: `nodev,nosuid,noexec` pada `/tmp`, dengan backup `fstab`, `findmnt`, dan tes eksekusi (langsung ditolak, lewat `bash` tetap jalan)
- Mendiagnosis masalah boot VM (stuck di initramfs) dengan menguji beberapa hipotesis; workaround GRUB ditemukan, akar masalah belum terkonfirmasi
- Menyusun checklist CIS 96 item, catatan lengkap sesi (`.md` dan `.docx` A5), dan draft profil project

**Angka & bukti konkret:**
- 1 VM target, 6 partisi terpisah plus swap
- 96 item checklist CIS di 6 kategori (23 item inti dari checklist awal); 4 item selesai: 2 diterapkan (partisi terpisah, mount option `/tmp`) dan 2 diverifikasi pada baseline (permission `/etc/passwd` dan `/etc/shadow`)
- 21 check function direncanakan untuk audit script (8 dari kerangka awal)
- ±61 screenshot terdokumentasi (Fase 1: 29, Fase 2: 30, Fase 3: 2)
- 5 entri login gagal dianalisis lewat `lastb` dan dicocokkan dengan `auth.log`
- Progress keseluruhan: 21 dari 151 item (14%) pada checklist di bawah

**5 Screenshot Terpenting:**

| No | Nama File (dengan path folder) | Apa yang ditunjukkan |
|---|---|---|
| 1 | `docs/screenshots/fase1/17-storage-configuration-final-summary` | Rencana partisi custom: 6 partisi terpisah sebelum instalasi |
| 2 | `docs/screenshots/fase2/06-auth-log-bukti-percobaan-gagal` | Percobaan login SSH gagal tercatat di `auth.log` lengkap dengan IP asal |
| 3 | `docs/screenshots/fase2/09-last-dan-lastb` | Riwayat login sukses dan gagal (`last`, `lastb`) dan gap waktu project |
| 4 | `docs/screenshots/fase3/00-fstab-tmp-sesudah-edit` | Mount option `/tmp` di `fstab` dan yang aktif di `findmnt`: `nosuid,nodev,noexec` |
| 5 | `docs/screenshots/fase3/01-tes-eksekusi-tmp-sesudah` | Tes: eksekusi langsung ditolak (`Permission denied`), lewat `bash` tetap jalan |

---

## Project Progress Checklist: Linux Hardening & Security Audit Tool

**Terakhir diupdate:** 23 Oktober 2026

### Fase 1: Setup Environment

| Status | Item | Catatan |
|---|---|---|
| ✅ | Install Ubuntu Server di VBox (minimal install, no GUI). Aku pakai install standar, bukan "minimized", biar baseline-nya lengkap | fase1/23, fase1/24 |
| ✅ | Setup networking (bridged/NAT), SSH access. Aku pakai NAT + Host-only (dua adapter), SSH dari Windows udah jalan | fase1/11, fase1/25, fase1/26 |
| ✅ | Snapshot VM di kondisi fresh install (buat baseline sebelum hardening): `fresh-install-before-hardening` | fase1/27, fase1/28 |
| ✅ | Custom partitioning: `/` 15G, swap 4G, `/tmp` 3G, `/var` 5G, `/var/log` 5G, `/home` 5G | fase1/17, fase2/22 |
| ✅ | Struktur folder project dibuat dan di-push ke Git | fase1/00 |

### Fase 2: Pembelajaran FHS

| Status | Item | Catatan |
|---|---|---|
| ✅ | Apa fungsi tiap direktori top-level (`/etc`, `/var`, `/tmp`, `/usr`, `/bin`, `/sbin`, `/lib`, `/opt`, `/proc`, `/sys`, `/dev`, `/home`, `/root`, `/boot`, `/mnt`, `/media`, `/srv`) | fase2/00 sampai fase2/29 |
| ✅ | - `/etc` (lewat `passwd`, `shadow`, `fstab`) | - |
| ✅ | - `/var` dan `/var/log` (`auth.log`, `last`, `lastb`) | - |
| ✅ | - `/tmp` (permission 1777, sticky bit, `noexec`) | - |
| ✅ | - `/bin`, `/sbin`, `/lib` (symlink merged-usr) | - |
| ✅ | - `/opt`, `/mnt`, `/media`, `/srv` (kosong, baseline) | - |
| ✅ | - `/proc`, `/sys` | - |
| ✅ | - `/home`, `/root`, `/boot` | - |
| ⬜ | - `/usr` dan `/dev` (baru sebagian, kebahas lewat symlink dan `/dev/shm`) | - |
| ✅ | Kenapa itu relevan buat blue team: `/tmp` dan `/dev/shm` sering dipakai attacker buat staging payload karena writable & world-accessible; `/var/log` isinya log yang perlu diketahui lokasinya buat investigasi; `/etc/passwd` vs `/etc/shadow` beda fungsi & risk-nya | - |
| ⬜ | Referensi resmi: `man hier` (built-in di Linux) dan FHS spec dari Linux Foundation (belum dibaca) | - |

### Fase 3 (Hardening Manual), kategori A. Filesystem & Initial Setup

| Status | Item | Catatan |
|---|---|---|
| ✅ | Partisi terpisah untuk `/tmp`, `/var`, `/var/log`, `/home` | fase1/17, fase2/22 |
| ✅ | `nodev`, `nosuid`, `noexec` diset pada `/tmp` di `/etc/fstab` | fase3/00, fase3/01 |
| ⬜ | AIDE terinstall untuk filesystem integrity checking | - |
| 🔄 | `/dev/shm` di-mount dengan `nodev`, `nosuid`, `noexec`. Baseline: `rw,nosuid,nodev,inode64` (tanpa `noexec`); langkah sudah diberikan, hasil belum dikonfirmasi | langkah sudah diberikan, hasil belum dikonfirmasi (fase3/02, fase3/03) |
| ⬜ | `nodev` dan `nosuid` pada `/var` dan `/home`; `nodev`, `nosuid`, `noexec` pada `/var/log` | - |
| ⬜ | Partisi terpisah untuk `/var/tmp` dan `/var/log/audit` (opsional; `/var/tmp` ikut dikunci seperti `/tmp`) | fase1/17, fase2/22 |
| ⬜ | Database AIDE diinisialisasi dan pengecekan dijadwalkan (cron atau systemd timer) | - |
| ⬜ | Filesystem yang tidak dipakai dinonaktifkan lewat modprobe: `cramfs`, `freevxfs`, `hfs`, `hfsplus`, `jffs2`, `udf`. `squashfs` dikecualikan selama `snap` dipakai | - |
| ⬜ | `usb-storage` di-blacklist (mencegah media removable di server) | - |
| ⬜ | `autofs` tidak terinstall atau dinonaktifkan (cegah automount) | - |
| ⬜ | Permission `/boot/grub/grub.cfg` dibatasi (root, tidak bisa dibaca group/other); password GRUB opsional untuk VM lab | - |
| ⬜ | ASLR aktif: `kernel.randomize_va_space = 2` | - |
| ⬜ | Core dump dibatasi: `fs.suid_dumpable = 0` dan `* hard core 0` di `limits.conf` | - |
| ⬜ | `kernel.yama.ptrace_scope` minimal 1 | - |
| ⬜ | AppArmor aktif dan profile dalam mode enforce | - |
| ⬜ | Banner `/etc/issue`, `/etc/issue.net`, `/etc/motd` tidak membocorkan versi OS | - |
| ⬜ | Paket dan security update terbaru (`apt update && apt upgrade`); baseline: 34 update tertunda; `unattended-upgrades` aktif | - |
| ⬜ | Time synchronization aktif (`systemd-timesyncd` atau `chrony`), penting agar timestamp log konsisten | - |

### Fase 3 (Hardening Manual), kategori B. Services

| Status | Item | Catatan |
|---|---|---|
| ⬜ | Disable service yang tidak perlu: `avahi-daemon`, `cups`, `isc-dhcp-server`, `nfs-server`, `rpcbind`, `samba`, `xinetd` | - |
| ⬜ | Cek service aktif: `systemctl list-units --type=service --state=running` | - |
| ⬜ | Simpan baseline daftar service yang berjalan sebelum hardening (untuk perbandingan before/after) | - |
| ⬜ | Tidak terinstall atau disabled bila tidak dipakai: `slapd`, `snmpd`, `vsftpd`, `telnetd`, `tftpd`, `dovecot`, `squid`, `bind9`, `apache2`, `nginx`, `nis`, `rsync` (service), `dnsmasq` | - |
| ⬜ | MTA (misal `postfix`) hanya listen di localhost, bila terinstall | - |
| ⬜ | `xserver-xorg` tidak terinstall (server tanpa GUI) | - |
| ⬜ | Tinjau kebutuhan `snapd` dan `lxd` | - |

### Fase 3 (Hardening Manual), kategori C. Network Configuration

| Status | Item | Catatan |
|---|---|---|
| ⬜ | Disable IP forwarding: `net.ipv4.ip_forward = 0` (baseline sudah `0`, tinggal didokumentasikan) | - |
| ⬜ | Disable ICMP redirects: `net.ipv4.conf.all.accept_redirects = 0` | - |
| ⬜ | Enable TCP SYN cookies: `net.ipv4.tcp_syncookies = 1` | - |
| ⬜ | Disable IPv6 kalau tidak dipakai | - |
| ⬜ | `net.ipv4.conf.default.accept_redirects = 0` dan `net.ipv6.conf.all.accept_redirects = 0` | - |
| ⬜ | `net.ipv4.conf.all.send_redirects = 0` dan `net.ipv4.conf.default.send_redirects = 0` | - |
| ⬜ | `net.ipv4.conf.all.secure_redirects = 0` dan `net.ipv4.conf.default.secure_redirects = 0` | - |
| ⬜ | `net.ipv4.conf.all.accept_source_route = 0` (default dan IPv6 juga) | - |
| ⬜ | `net.ipv4.conf.all.log_martians = 1` (paket dengan alamat mustahil dicatat) | - |
| ⬜ | `net.ipv4.icmp_echo_ignore_broadcasts = 1` dan `net.ipv4.icmp_ignore_bogus_error_responses = 1` | - |
| ⬜ | `net.ipv4.conf.all.rp_filter = 1` (anti IP spoofing) | - |
| ⬜ | `net.ipv6.conf.all.accept_ra = 0` bila IPv6 dipakai | - |
| ⬜ | Setting sysctl dibuat permanen di `/etc/sysctl.d/*.conf`, bukan hanya `sysctl -w` (hilang saat reboot) | - |
| ⬜ | Firewall host (`ufw`) aktif: default deny incoming, izinkan SSH dari jaringan Host-only saja. Pastikan aturan SSH ada sebelum firewall diaktifkan | - |
| ⬜ | Protokol jarang dipakai dinonaktifkan: `dccp`, `sctp`, `rds`, `tipc` | - |
| ⬜ | Port yang listening ditinjau (`ss -tulpn`); yang terbuka hanya yang dibutuhkan (SSH) | - |

### Fase 3 (Hardening Manual), kategori D. Logging & Auditing

| Status | Item | Catatan |
|---|---|---|
| ⬜ | `auditd` terinstall dan enabled | - |
| ⬜ | `rsyslog` terinstall dan enabled | - |
| ⬜ | Logrotate dikonfigurasi | - |
| ⬜ | Rule auditd untuk: perubahan waktu sistem, file identitas (`/etc/passwd`, `/etc/group`, `/etc/shadow`, `/etc/gshadow`), `/etc/sudoers`, `/etc/ssh/sshd_config`, login/logout (`lastlog`, `faillog`), sesi (`utmp`, `wtmp`, `btmp`), perubahan mount, penghapusan file, pemuatan modul kernel, perubahan permission dan ownership | - |
| ⬜ | Rule auditd khusus blue team: eksekusi dari direktori writable (`/tmp`, `/dev/shm`) dan perubahan `/etc/fstab` | - |
| ⬜ | Konfigurasi auditd: `max_log_file`, `max_log_file_action`, `space_left_action`, `admin_space_left_action`; parameter kernel `audit=1` | - |
| ⬜ | Audit config immutable (`-e 2`) di akhir rules (opsional; ubah rule butuh reboot) | - |
| ⬜ | Permission file di `/var/log` dan `/var/log/audit` dibatasi (umumnya 0640 atau lebih ketat) | - |
| ⬜ | `journald` persistent dan terkompresi | - |
| ⬜ | Retensi `auth.log` cukup panjang (log lama pindah ke `auth.log.1`; bukti brute-force tidak boleh hilang cepat) | - |
| ⬜ | Pengiriman log ke remote/SIEM (opsional, lanjutan: Wazuh atau Splunk di lab yang sudah ada) | - |
| ⬜ | Hasil pengecekan AIDE tercatat di log | - |

### Fase 3 (Hardening Manual), kategori E. Access, Authentication, Authorization

| Status | Item | Catatan |
|---|---|---|
| ⬜ | `cron` dibatasi hanya untuk user tertentu (`/etc/cron.allow`) | - |
| ⬜ | SSH hardening: `PermitRootLogin no`, `PasswordAuthentication no`, `MaxAuthTries 4`, `LogLevel VERBOSE`, banner, idle timeout | - |
| ⬜ | PAM password policy: minimum length, complexity, history (`pam_pwquality`) | - |
| ⬜ | Password expiration: `PASS_MAX_DAYS`, `PASS_MIN_DAYS`, `PASS_WARN_AGE` di `/etc/login.defs` | - |
| ⬜ | Default `umask` diset ke 027 atau lebih ketat | - |
| ⬜ | Root `PATH` tidak mengandung direktori writable oleh user lain | - |
| ⬜ | Permission `/etc/crontab` dan `/etc/cron.hourly`, `cron.daily`, `cron.weekly`, `cron.monthly`, `cron.d` dimiliki root dan tidak bisa diakses group/other | - |
| ⬜ | `at` dibatasi (`/etc/at.allow`) atau tidak terinstall | - |
| ⬜ | SSH key dibuat dan diuji login-nya sebelum `PasswordAuthentication no` diterapkan, supaya tidak terkunci | - |
| ⬜ | Permission `/etc/ssh/sshd_config` (0600 root) dan private host key (0600) | - |
| ⬜ | `PermitEmptyPasswords no`, `HostbasedAuthentication no`, `IgnoreRhosts yes`, `PermitUserEnvironment no` | - |
| ⬜ | `X11Forwarding no` dan `AllowTcpForwarding no` (atau `DisableForwarding yes`) | - |
| ⬜ | Idle timeout: `ClientAliveInterval` dan `ClientAliveCountMax`; plus `LoginGraceTime 60`, `MaxSessions`, `MaxStartups` | - |
| ⬜ | `AllowUsers` atau `AllowGroups` membatasi siapa yang boleh login SSH | - |
| ⬜ | Algoritma kuat (`Ciphers`, `MACs`, `KexAlgorithms`) sesuai CIS | - |
| ⬜ | Banner SSH diarahkan ke `/etc/issue.net` | - |
| ⬜ | Config divalidasi dengan `sudo sshd -t` sebelum reload; uji dari sesi kedua sebelum menutup sesi lama | - |
| ⬜ | Lockout akun setelah gagal login (`pam_faillock`, misal `deny=5`, `unlock_time=900`) | - |
| ⬜ | `pam_pwhistory` (`remember=24`) selain `pam_pwquality` | - |
| ⬜ | Algoritma hash password kuat (`yescrypt` atau `sha512`) | - |
| ⬜ | `INACTIVE` di `useradd -D` maksimal 30 hari | - |
| ⬜ | Tidak ada akun dengan password kosong | - |
| ⬜ | `sudo` memakai `use_pty` dan `logfile`, `timestamp_timeout` dibatasi | - |
| ⬜ | `su` dibatasi ke grup tertentu (`pam_wheel`) | - |
| ⬜ | Anggota grup `sudo` hanya user yang berhak. Baseline: hanya `linuxaufa`. Tinjau juga grup `lxd` (anggotanya dapat dipakai untuk eskalasi ke root) dan `adm` (akses baca log) | - |
| ⬜ | Timeout shell interaktif (`TMOUT=900`, readonly). Terkait sesi `tty1` yang menganggur di baseline | - |
| ⬜ | Akun sistem memakai `nologin` dan terkunci | - |
| ⬜ | Home directory user 750 atau lebih ketat; dot file tidak writable group/other; tidak ada `.netrc`, `.forward`, `.rhosts` | - |

### Fase 3 (Hardening Manual), kategori F. System Maintenance / File Permissions

| Status | Item | Catatan |
|---|---|---|
| ✅ | Permission `/etc/passwd` = 644, owner root:root (baseline terverifikasi) | baseline terverifikasi, fase2/00 |
| ✅ | Permission `/etc/shadow` = 640/000, owner root:shadow (baseline terverifikasi: 640) | baseline terverifikasi (640), fase2/00 |
| ⬜ | Tidak ada duplicate UID/GID | - |
| ⬜ | Hanya `root` yang punya UID 0 | - |
| ⬜ | Tidak ada file world-writable tanpa sticky bit di direktori umum | - |
| ⬜ | Permission `/etc/group` dan `/etc/shells` = 644; `/etc/passwd-` = 644; `/etc/group-` = 644; `/etc/shadow-`, `/etc/gshadow`, `/etc/gshadow-` = 640 root:shadow | - |
| ⬜ | Tidak ada duplicate username dan group name | - |
| ⬜ | Semua GID di `/etc/passwd` ada di `/etc/group` | - |
| ⬜ | Grup `shadow` tidak punya anggota | - |
| ⬜ | Tidak ada file atau direktori tanpa owner atau group (`-nouser`, `-nogroup`) | - |
| ⬜ | File SUID/SGID ditinjau dan baseline-nya dicatat | - |
| ⬜ | File world-writable (bukan direktori sticky) ditinjau | - |
| ⬜ | Home directory semua user interaktif ada dan dimiliki user yang benar | - |
| ⬜ | Tidak ada entri legacy `+` di `/etc/passwd`, `/etc/shadow`, `/etc/group` | - |
| ⬜ | Pengecekan `find` memakai path asli (`/usr/bin`) atau `-L`, karena `/bin`, `/sbin`, `/lib` adalah symlink (hindari false negative) | - |

### Fase 4: Build Audit Script (check function)

| Status | Item | Catatan |
|---|---|---|
| ⬜ | `check_ssh_root_login` (kategori E) | - |
| ⬜ | `check_ssh_password_auth` (kategori E) | - |
| ⬜ | `check_unnecessary_services` (kategori B) | - |
| ⬜ | `check_ip_forwarding` (kategori C) | - |
| ⬜ | `check_passwd_permissions` (kategori F) | - |
| ⬜ | `check_shadow_permissions` (kategori F) | - |
| ⬜ | `check_password_policy` (kategori E) | - |
| ⬜ | `check_auditd_status` (kategori D) | - |
| ⬜ | `check_tmp_mount_options` (A): `nodev`, `nosuid`, `noexec` di `/tmp` | - |
| ⬜ | `check_dev_shm_mount_options` (A): `noexec` di `/dev/shm` | - |
| ⬜ | `check_aide_installed` (A) | - |
| ⬜ | `check_aslr` (A) | - |
| ⬜ | `check_sysctl_network` (C): redirects, source route, SYN cookies, `rp_filter`, `log_martians` | - |
| ⬜ | `check_firewall_status` (C) | - |
| ⬜ | `check_rsyslog_status` dan `check_logrotate` (D) | - |
| ⬜ | `check_ssh_bruteforce_log` (D/E): hitung `Failed password`, `Invalid user`, `authentication failure` per IP, cuma dari proses `sshd`, pakai threshold | - |
| ⬜ | `check_cron_permissions` (E) | - |
| ⬜ | `check_umask` dan `check_root_path` (E) | - |
| ⬜ | `check_sudo_group_members` (E): anggota grup `sudo` dan `lxd` | - |
| ⬜ | `check_uid_zero` dan `check_duplicate_uid_gid` (F) | - |
| ⬜ | `check_world_writable_files` (F): pakai path asli atau `-L`, jangan lewat symlink | - |

### Fase 5: Dokumentasi / Writeup

| Status | Item | Catatan |
|---|---|---|
| ⬜ | Screenshot before/after tiap kategori hardening | - |
| ⬜ | Command yang dipakai + penjelasan kenapa | - |
| ⬜ | Hasil run audit script (before vs after hardening) | - |
| ⬜ | Push ke GitHub repo | - |
| ⬜ | `README.md` repo dan `writeup.md` (apa yang dipelajari, kesulitan, temuan menarik) | - |
| ⬜ | Bagian troubleshooting yang jujur: mana yang terbukti, mana yang masih dugaan | - |

### Troubleshooting, Dokumen Pendukung, dan Output Akhir

| Status | Item | Catatan |
|---|---|---|
| 🔄 | Akar masalah VM stuck di "Loading essential drivers" | workaround GRUB `e` lalu `F10` jalan; penyebab belum terkonfirmasi |
| ⬜ | Tes mendiamkan menu GRUB 15 sampai 20 detik (uji dugaan timing) | - |
| ⬜ | Disk Fixed size untuk VM baru (opsional, uji dugaan disk dynamic) | - |
| ✅ | Checklist CIS lengkap (`checklist/cis-benchmark-checklist.md`) | 96 item di 6 kategori |
| ✅ | Catatan lengkap sesi (`.md` dan `.docx` A5) | dibuat 20 Okt 2026 |
| ✅ | Draft profil project (`project-profile-draft.md`) | perlu revisi periode dan tag status |
| ⬜ | Revisi draft profil (jeda Agustus sampai Oktober, update tag) | - |
| ⬜ | Commit dan push screenshot Fase 2 dan 3 ke GitHub | belum dikonfirmasi |
| 🔄 | Repo GitHub berisi script, checklist markdown, report before/after | init dan push awal sudah; script dan report belum ada |
| ⬜ | Writeup: apa yang dipelajari, kesulitan, temuan menarik | - |
| ⬜ | (Opsional) Versi Python dengan HTML output report | - |
| ⬜ | Kirim log ke Wazuh/Splunk di lab yang sudah ada (rencana lanjutan) | - |

---

### Summary

| Metric | Jumlah |
|---|---|
| ✅ Selesai | 21 item |
| 🔄 Sedang berjalan | 3 item |
| ⬜ Belum dimulai | 127 item |
| **Total** | **151 item** |
| **Progress** | **14%** (hanya menghitung ✅) |

**Terakhir selesai di:** Fase 3, item mount option `/tmp` (`nodev,nosuid,noexec`): diedit di `/etc/fstab`, diterapkan dengan `daemon-reload` dan `remount`, diverifikasi dengan `findmnt` dan tes eksekusi (screenshot `fase3/00` dan `fase3/01`)  
**Selanjutnya:** mount option `/dev/shm` (tambah `noexec`), lalu AIDE

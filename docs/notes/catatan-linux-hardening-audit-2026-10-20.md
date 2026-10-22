# Catatan Project: Linux Hardening & Security Audit Tool

Tanggal catatan: 20 Oktober 2026

Cakupan: Fase 1 (Setup Environment), Fase 2 (FHS), dan awal Fase 3 (Hardening Manual). Environment: Ubuntu Server 24.04.4 LTS di VirtualBox, host Windows 11 Home.

## 1. Gambaran Project

### 1.1 Tujuan dan hasil akhir

- Paham struktur filesystem Linux (FHS) dan relevansinya untuk blue team
- Menerapkan hardening berbasis CIS Benchmark secara manual, bukan copy-paste
- Membangun audit tool Bash yang mengecek compliance (PASS/FAIL + rekomendasi)
- Output akhir: repo GitHub (script, checklist, report before/after), writeup, opsional versi Python dengan HTML report

### 1.2 Tech stack

- VirtualBox dengan Ubuntu Server 24.04.4 LTS sebagai target hardening
- Bash untuk audit script, Python (opsional) untuk HTML report
- Git dan GitHub untuk version control dan dokumentasi before/after

### 1.3 Struktur folder project

```
linux-hardening-audit/
├── README.md
├── writeup.md
├── scripts/
├── checklist/
└── docs/
    ├── screenshots/ (fase1 sampai fase4)
    ├── notes/
    └── reports/
```

- `checklist/` dipisah dari `docs/` karena checklist adalah dokumen hidup yang terus diperbarui
- `docs/reports/` untuk hasil audit before/after (snapshot, bukan dokumen hidup)
- Folder dibuat lewat PowerShell `mkdir`, lalu `git init` dan push awal

### 1.4 Peta fase

| Fase | Isi | Status per 20 Okt 2026 |
|---|---|---|
| 1 | Setup environment | Selesai |
| 2 | Pembelajaran FHS | Selesai (opsional: `/dev`, `/usr`, `man hier`) |
| 3 | Hardening manual | Dimulai (item /tmp) |
| 4 | Audit script Bash | Belum dimulai |
| 5 | Dokumentasi dan writeup | Belum dimulai |

### 1.5 Aturan kerja dan konvensi

- Step-by-step, satu langkah kecil per giliran, lanjut setelah mengetik "lanjut"
- Hint dulu: pertanyaan pemantik sebelum teknis; "gabisa" berarti dijelaskan dari dasar
- Setiap command dijelaskan fungsi dan alasannya
- Audit script (Fase 4) diisi sendiri dulu, bantuan hanya jika stuck
- Screenshot diberi judul `docs/screenshots/faseN/NN-nama-deskriptif` tanpa ekstensi `.png`, nomor 2 digit mulai 00 dan reset per fase
- Password dan kredensial tidak dibagikan dalam screenshot atau chat

## 2. Fase 1: Setup Environment

### 2.1 Pilihan dasar OS

**T:** Kenapa Ubuntu Server (bukan Desktop) dan kenapa instalasi standar (bukan minimized)?  
**J:** Server lebih ringan. Desktop membawa paket tambahan (GUI, `avahi-daemon`, `cups`) yang menambah **attack surface** dan menjadi noise saat hardening. Prinsipnya **reduce attack surface**.

- Pilih "Ubuntu Server" standar, bukan "minimized", karena baseline before/after perlu service bawaan agar item "disable unnecessary services" ada praktiknya
- Versi minimized dirancang untuk lingkungan tanpa login manusia dan memangkas dokumentasi (`man`) yang dibutuhkan di Fase 2
- Third-party drivers tidak dicentang: VM memakai hardware virtual yang sudah didukung kernel

### 2.2 Partisi: guided vs custom

**T:** Pakai partisi otomatis (guided) atau manual?  
**J:** Manual (Custom storage layout). Guided membuat satu wadah besar sehingga `/tmp`, `/var`, `/home` hanya folder di dalam root.

- Mount option berlaku per partisi, bukan per folder. Tanpa partisi `/tmp` sendiri, `nodev,nosuid,noexec` tidak bisa dipasang khusus untuk `/tmp`
- Containment: kalau `/var/log` membludak (misal akibat serangan brute-force), hanya partisi itu yang penuh. Root yang penuh bisa membuat sistem crash
- Analogi: lemari tanpa sekat vs lemari dengan laci yang bisa dikunci satu per satu
- "Lebih aman" di sini berarti kontrol lebih granular, bukan otomatis 100% aman

**T:** Partisi mana yang butuh ruang terbesar?  
**J:** `/` (root), karena menampung `/usr` (hampir semua program dan library). Bukan `/var/log`, yang di VM eksperimen tidak akan sebesar itu.

**T:** Apa bedanya `/var` dan `/var/log`?  
**J:** `/var` adalah wadah data yang berubah-ubah (log, cache, spool mail, database). `/var/log` adalah sub-folder khusus log. Dipisah supaya log yang membludak tidak memenuhi `/var`.

Rencana dan hasil partisi (disk 40 GB):

| Mount | Ukuran | Alasan |
|---|---|---|
| `/` | 15G | OS, `/usr`, package |
| swap | 4G | virtual memory (RAM 4G) |
| `/tmp` | 3G | file sementara, nanti dikunci |
| `/var` | 5G | cache, spool, data variabel |
| `/var/log` | 5G | terpisah agar log tidak memenuhi `/var` |
| `/home` | 5G | data user |

- Sisa ±3G sengaja dibiarkan unallocated sebagai buffer (free space 2.997G)
- Partisi 1 (1 MB, BIOS grub spacer) dibuat otomatis installer untuk bootloader GRUB di disk GPT
- `/usr` dan `/boot` tidak dipisah. Pemisahan `/usr` menambah risiko gagal boot karena harus terpasang sangat awal

### 2.3 Mode network VirtualBox

| Mode | Perilaku | Dipakai? |
|---|---|---|
| NAT | VM bisa keluar ke internet, host tidak bisa inisiasi koneksi ke VM (kecuali port forwarding) | Adapter 1 |
| Host-only | Host dan VM saling terhubung, tanpa internet | Adapter 2 |
| Bridged | VM tampil seperti perangkat di LAN fisik | Tidak |
| Internal Network | Hanya antar VM, host tidak ikut | Tidak |

**T:** Mode apa supaya host bisa SSH ke VM?  
**J:** Host-only memungkinkan SSH, tetapi tidak punya internet, padahal `apt install` butuh internet. Solusinya dua adapter: NAT (internet) + Host-only (SSH).

- Satu adapter NAT saja bisa dengan port forwarding, tapi lebih ribet
- Host-only terisolasi dari internet luar, jadi eksperimen sensitif tidak bocor keluar
- Hasil di VM: `enp0s3` = NAT, `10.0.2.15/24`; `enp0s8` = Host-only, `192.168.56.107/24` (DHCP)
- Di sisi Windows, adapter Host-only memakai `192.168.56.1`

### 2.4 Konfigurasi VM

- Nama `ubuntu-hardening-lab`, OS Ubuntu (64-bit), ISO `ubuntu-24.04.4-live-server-amd64.iso`
- **Proceed with Unattended Installation** harus di-uncheck. Kalau aktif, VirtualBox menginstal otomatis (termasuk partisi otomatis), padahal kita butuh partisi manual
- Field username/password unattended menjadi abu-abu: itu normal dan menandakan setting benar
- RAM 4096 MB, 2 CPU, EFI tidak dicentang
- Disk VDI 40 GB, **Dynamically allocated** (file membesar sesuai pemakaian). Alternatifnya Fixed size (langsung 40 GB)

### 2.5 Langkah installer (Subiquity)

| Layar | Pilihan | Alasan |
|---|---|---|
| GRUB | Try or Install Ubuntu Server | kernel standar cukup di VM (HWE untuk hardware baru) |
| Language | English | konsisten dengan dokumentasi CIS dan Linux |
| Installer update | Continue without updating | update installer bukan update OS |
| Keyboard | English (US) | default |
| Installation type | Ubuntu Server | baseline standar |
| Network | default DHCP | `10.0.2.x` dan `192.168.56.x` terverifikasi |
| Proxy | kosong | tidak di jaringan berproxy |
| Mirror | default (`id.archive.ubuntu.com`) | mirror Indonesia otomatis, "passed tests" |
| Storage | Custom storage layout | partisi terpisah |
| Profile | nama, hostname, username, password | hostname `ubuntu-hardening-lab` |
| Ubuntu Pro | Skip for now | layanan berbayar, tidak relevan |
| SSH | Install OpenSSH server, password auth aktif | baseline; dimatikan di Fase 3 |
| Featured snaps | tidak memilih apa pun | footprint minimal |

### 2.6 Navigasi installer dan form partisi

- Panah atas/bawah memindah highlight, Enter atau Spasi memilih (termasuk radio button)
- Tab memindah antar field di form
- Form **Add GPT Partition** berisi Size, Format, Mount
- `/tmp` tidak ada di dropdown Mount: pilih **Other** lalu ketik `/tmp`
- Swap: Format diganti `swap`, field Mount hilang otomatis
- Syarat lanjut: ada partisi yang di-mount ke `/` dan boot disk terpilih

**T:** Setelah menekan Done di storage, apakah masih bisa diubah?  
**J:** Selama masih rencana (status "new, to be formatted") masih bisa lewat Reset atau edit partisi. Dialog **Confirm destructive action** adalah titik tanpa kembali: setelah Continue, disk diformat dan instalasi berjalan.

### 2.7 Setelah instalasi

- Pesan `soft lockup - CPU#1 stuck for 40s` sempat muncul saat tahap security updates. Itu warning kernel, bukan error fatal; instalasi tetap selesai
- Login pertama sukses (Ubuntu 24.04.4 LTS, kernel 6.8.0-136-generic, 34 update tersedia)
- **SSH host key** tercetak otomatis oleh cloud-init: identitas kriptografi server untuk memverifikasi bahwa client menyambung ke server yang sama (mencegah man-in-the-middle)
- Verifikasi `ip a`: interface `lo`, `enp0s3`, `enp0s8` semuanya UP dengan IP yang diharapkan
- SSH dari PowerShell: peringatan fingerprint saat koneksi pertama dijawab `yes`
- **Snapshot** `fresh-install-before-hardening` dibuat sebagai baseline

**T:** Snapshot VM itu apa, sama dengan screenshot?  
**J:** Bukan. Snapshot adalah save point seluruh kondisi disk (dan memori) VM, seperti save game. Jika konfigurasi hardening merusak sistem, VM bisa dikembalikan lewat klik kanan snapshot, lalu Restore. Snapshot bisa dibuat saat VM menyala.

```bash
ip a
ssh linuxaufa@192.168.56.107
```

## 3. Troubleshooting Boot dan Virtualization

### 3.1 VM stuck di "Begin: Loading essential drivers"

**Masalah** : Setelah instalasi, boot berikutnya berhenti di `Begin: Loading essential drivers ...` (tahap initramfs) dan berulang. Log kernel menampilkan `soft lockup`, timestamp mencapai 888 detik lebih. VM lain (Kali, endpoint Splunk) normal.  
**Penyebab** : Belum terkonfirmasi. Dugaan: masalah timing/I/O saat boot yang berinteraksi dengan host (Defender, VBS, atau jeda hibernate).  
**Solusi** : Workaround yang konsisten berhasil: di menu GRUB, pada entry "Ubuntu" tekan `e` lalu `F10`.  
**Catatan** : Jangan menulis "sudah diselesaikan" di writeup. Yang terbukti hanya workaround. Perbedaan yang terverifikasi dari VM yang normal: VM ini memakai 6 partisi manual, VM Splunk satu partisi. Tipe alokasi disk VM Splunk tidak diketahui (VirtualBox hanya menampilkan "Normal").

Hipotesis yang sudah diuji:

| Hipotesis | Hasil pengecekan | Status |
|---|---|---|
| ISO masih terpasang | IDE optical sudah Empty | Bukan penyebab |
| Shutdown tidak bersih (fsck) | Shutdown dilakukan dengan `sudo shutdown -h now` | Tidak sesuai fakta |
| Konflik Hyper-V/WSL | VMP, WHP, WSL tidak dicentang, Hyper-V tidak muncul (Windows 11 Home) | Bukan penyebab saat dicek |
| Memory Integrity (VBS/HVCI) | `systeminfo`: VBS Running, HVCI Running; setelah dimatikan dan Windows restart, VM boot | Terkait, akar belum pasti |
| Paravirtualization Hyper-V | Diubah ke Hyper-V, tetap stuck; kedua VM memakai KVM paravirtualization | Bukan penyebab |
| Exclusion Windows Defender | Folder VM ditambahkan; boot sempat normal lalu stuck lagi | Tidak menyelesaikan |
| Jumlah CPU (1 core) | Tidak diuji; VM Kali 6 core lancar | Tidak logis sebagai penyebab tunggal |
| Boot order (Floppy dulu) | Diubah: Hard Disk, Optical, Floppy dinonaktifkan | Hasil tidak dilaporkan |
| Banyak partisi dan disk dynamic | VM Splunk satu partisi, lancar; belum diuji disk Fixed | Dugaan, belum diuji |

Perbandingan dua VM:

| Atribut | ubuntu-hardening-lab | endpoint-linux-splunkk |
|---|---|---|
| RAM | 4096 MB | 8192 MB |
| CPU | 2 | 4 |
| Disk | 40 GB | 50 GB |
| Boot order awal | Floppy, Optical, Hard Disk | Hard Disk, Optical, Floppy |
| Acceleration | Nested Paging, KVM Paravirtualization | sama |
| Partisi | 6 partisi manual | satu partisi |

**T:** Kenapa `e` lalu `F10` di GRUB bisa membuat boot lancar?  
**J:** `e` membuka edit mode yang menampilkan konfigurasi boot entry. `F10` boot dengan konfigurasi itu tanpa menyimpan perubahan. Jika tidak ada yang diubah, parameter kernel identik dengan boot biasa. Yang berbeda diduga **jeda waktu** beberapa detik sebelum initramfs mulai, sehingga proses inisialisasi VirtualBox/Windows sempat selesai. Ini hipotesis; tes pembuktiannya (mendiamkan menu GRUB 15 sampai 20 detik) belum dijalankan.

Opsi permanen yang belum dicoba:

- Disk Fixed size (butuh membuat VM baru)
- Mematikan real-time protection Defender sementara untuk menguji teori scanning
- Mengonfirmasi lewat Event Viewer apa yang mengaktifkan VBS

### 3.2 Pesan soft lockup di console setelah laptop hibernate

**Masalah** : Console menampilkan `watchdog: BUG: soft lockup - CPU#0 stuck for 1309s! [swapper/0:0]`, lalu 3018s, dan `systemd-logind.service: Watchdog timeout (limit 3min)`.  
**Penyebab** : Dugaan: host (laptop) hibernate atau VM di-pause, jam kernel meloncat, lalu watchdog mengira CPU macet puluhan menit. Pengguna mengonfirmasi laptop sempat hibernate dan VM pernah didiamkan lama.  
**Solusi** : Tidak perlu perbaikan. Menekan Enter hanya menggambar ulang prompt, pesan tetap ada di `dmesg`.  
**Catatan** : `swapper/0` adalah idle task, jadi yang "macet" adalah CPU menganggur, bukan program berat. Angka di kurung siku adalah detik sejak boot, bukan jam dinding.

**T:** Kenapa pesan itu muncul di console tapi tidak di terminal SSH?  
**J:** Pesan kernel secara default dicetak ke console. Sesi SSH hanya menerima output command yang dijalankan sendiri. Pesan itu tetap bisa dibaca dari SSH lewat `sudo dmesg | grep -i lockup`. Implikasinya untuk blue team: memantau server hanya lewat SSH membuat pesan kernel dan login console terlewat. Solusi nyata adalah log terpusat (auditd, rsyslog, SIEM).

```bash
sudo dmesg | grep -i lockup
```

- `dmesg`: membaca kernel ring buffer (pesan kernel sejak boot)
- `sudo`: di Ubuntu 24.04 `dmesg` untuk user biasa dibatasi (`kernel.dmesg_restrict`), setting yang baik untuk hardening
- `grep -i lockup`: filter baris berisi "lockup" tanpa peduli besar/kecil huruf

### 3.3 Memory Integrity dan keamanan host

- **Memory Integrity** (HVCI, Hypervisor-protected Code Integrity) mengunci area memori kernel Windows agar hanya kode ter-signed yang berjalan. Ini lapisan pertahanan terhadap rootkit dan driver malware
- Mematikannya bukan hal ekstrem bagi pengguna virtualisasi, tetapi menurunkan satu lapisan proteksi. Windows Defender tetap berjalan dan fitur ini bisa dinyalakan lagi kapan saja (toggle, lalu restart)
- Dampak ke aplikasi: biasanya kecil; game dengan anti-cheat kernel-level dan laptop kerja yang dikelola IT bisa terpengaruh
- **Windows Hypervisor Platform (WHP)** bukan fitur keamanan. Ia API agar software virtualisasi pihak ketiga (VirtualBox 7+) berjalan di atas Hyper-V, sehingga Memory Integrity bisa tetap ON. Konsekuensi: performa VM sedikit lebih lambat
- Status Memory Integrity saat VM kembali stuck tidak dicatat: perlu dicek ulang

**T:** Kenapa masalah ini baru muncul padahal sudah menjalankan hampir 10 VM?  
**J:** Belum terkonfirmasi. Dugaan: VBS aktif setelah Windows Update atau software tertentu (misal anti-cheat), atau beban boot VM ini berbeda (banyak partisi, I/O tinggi) sehingga memicu masalah yang tidak terlihat di VM lain. Konfirmasi opsional lewat Event Viewer, log System, event DeviceGuard atau CodeIntegrity.

```powershell
systeminfo
```

- Cari baris `Virtualization-based security: Status: Running` dan `Services Running: Hypervisor enforced Code Integrity` untuk memastikan VBS aktif
- Host: Windows 11 Home (build 26200), laptop Lenovo, 32 GB RAM

## 4. Fase 2: Filesystem dan Konsep Linux (FHS)

### 4.1 Peta direktori top-level

| Direktori | Fungsi | Relevansi blue team |
|---|---|---|
| `/etc` | konfigurasi sistem (teks) | target utama attacker; dipantau AIDE |
| `/var` | data variabel: log, cache, spool | `/var/log` = bukti investigasi |
| `/tmp` | file sementara, writable semua user | staging payload; kunci dengan `noexec` |
| `/usr` | program dan library | integritas binary |
| `/bin` `/sbin` `/lib` | symlink ke `usr/bin`, `usr/sbin`, `usr/lib` | path symlink bisa menipu script |
| `/opt` | software pihak ketiga | di luar `apt`; perubahan perlu dicek |
| `/proc` | virtual: proses dan status kernel | investigasi proses |
| `/sys` | virtual: perangkat dan driver | info hardware |
| `/dev` | file perangkat | file asing mencurigakan; `/dev/shm` writable |
| `/home` | data user biasa | partisi terpisah |
| `/root` | home user root | di partisi `/` agar root bisa login saat recovery |
| `/boot` | kernel, initrd, GRUB | persistence tingkat rendah |
| `/mnt` `/media` | mount manual / otomatis | media removable |
| `/srv` | data layanan (web, ftp) | kosong tanpa layanan |
| `/run` | data runtime (tmpfs) | hilang saat reboot |
| `/lost+found` | hasil `fsck` | ada di tiap partisi ext4 |

- Referensi resmi: `man hier` dan spesifikasi FHS Linux Foundation (belum dibaca)
- Prinsip: paham **konsep** lebih penting daripada menghafal nama file, karena nama berbeda antar distro

### 4.2 `/home` vs `/root`

**T:** Apa bedanya `/home` dan `/root`?  
**J:** `/home` adalah rumah semua user biasa (`/home/linuxaufa`). `/root` adalah rumah khusus user `root` (UID 0), bukan "grup admin". `/root` sengaja berada di partisi `/` (bukan `/home` yang terpisah): jika `/home` gagal mount saat boot, root tetap punya home dan bisa melakukan recovery.

### 4.3 `/tmp`, sticky bit, `noexec`

**T:** Kenapa `/tmp` jadi target favorit attacker?  
**J:** Karena permission `/tmp` adalah `1777` (`drwxrwxrwt`): semua user bisa menulis dan mengeksekusi. Attacker yang baru mendapat akses terbatas (misal user `www-data`) hampir pasti bisa menulis di `/tmp`, tetapi tidak di `/home/user` (home dibatasi untuk user lain; umumnya 700 atau 750). Koreksi atas jawaban awal: data proses tidak disimpan di `/tmp`, melainkan di RAM; dan akses `/tmp` bukan hanya admin.

**T:** Huruf `t` di akhir `drwxrwxrwt` itu apa?  
**J:** **Sticky bit**. Walaupun semua orang bisa menulis di direktori itu, user hanya boleh menghapus atau me-rename file milik sendiri. Tanpa sticky bit, user A bisa menghapus file user B. `d` di depan adalah tipe file (directory); `t` menggantikan `x` posisi terakhir.

**T:** Apa fungsi `noexec`?  
**J:** Mount option yang membuat kernel menolak menjalankan file apa pun di partisi itu, meski file punya izin `x`. Attacker bisa menulis payload ke `/tmp` tapi mendapat `Permission denied` saat menjalankannya langsung.

**T:** Apakah `noexec` bisa membedakan attacker dari user sah?  
**J:** Tidak. `noexec` buta terhadap identitas dan berlaku sama untuk semua orang. Ia bekerja sebagai pembatasan area berisiko tinggi: `/tmp` memang bukan tempat menjalankan program, jadi dampaknya ke user sah sangat kecil.

**T:** Kalau `noexec` aktif, apakah sticky bit jadi tidak berguna?  
**J:** Tetap berguna. Keduanya mengurus hal berbeda: sticky bit mengatur siapa boleh menghapus file siapa, `noexec` mengatur boleh tidaknya file dijalankan. Mereka saling melengkapi (defense in depth).

- `nosuid`: bit SUID diabaikan. SUID membuat program berjalan dengan hak pemiliknya (misal root), sehingga salinan program berhak root di `/tmp` tidak bisa dipakai menaikkan hak akses
- `nodev`: file perangkat (device file) tidak dianggap device
- Batas `noexec`: `bash /tmp/script.sh` tetap jalan karena bash membaca file sebagai teks, bukan menjalankannya sebagai program. `noexec` itu pelambat, bukan penutup total

### 4.4 `/etc/passwd` vs `/etc/shadow`

```
-rw-r--r-- 1 root root   /etc/passwd
-rw-r----- 1 root shadow /etc/shadow
```

**T:** Kenapa hash password dipindah dari `passwd` ke `shadow`?  
**J:** `/etc/passwd` berpermission 644, jadi **semua user** bisa membacanya. Jika hash ada di sana, user biasa bisa mengambil hash user lain untuk di-crack offline. `/etc/shadow` berpermission 640 (`root:shadow`): hanya root dan grup `shadow` yang bisa membaca. Koreksi: bukan "semua user tanpa izin", tetapi user di luar root/grup `shadow` tidak punya akses sama sekali.

**T:** Kalau `shadow` dikunci, kenapa `passwd` tetap 644?  
**J:** Karena `passwd` berisi metadata non-sensitif yang dibutuhkan banyak command. Di level kernel hanya ada angka UID; menerjemahkan UID 1000 menjadi nama `linuxaufa` (di `ls -l`, `ps`, `id`, `whoami`) dilakukan lewat lookup ke `/etc/passwd`. Login juga membutuhkan home directory dan shell dari file ini. Prinsipnya: pisahkan data berdasarkan sensitivitas, bukan mengunci semuanya.

Format satu baris `/etc/passwd` (7 field, dipisah `:`):

```
linuxaufa:x:1000:1000:<nama>:/home/linuxaufa:/bin/bash
```

- `linuxaufa`: username
- `x`: placeholder password, hash sebenarnya di `/etc/shadow`
- `1000`: UID (user pertama di Ubuntu biasanya 1000)
- `1000`: GID (primary group)
- `<nama>`: field GECOS (nama lengkap, dari "Your name" saat instalasi)
- `/home/linuxaufa`: home directory
- `/bin/bash`: shell default

- `id linuxaufa` menampilkan UID, GID, dan semua grup, termasuk `27(sudo)`
- `getent group sudo` menunjukkan anggota grup: hanya `linuxaufa`
- Audit: user yang seharusnya biasa tetapi ada di grup `sudo` adalah tanda privilege escalation
- Pada `ls -l`, angka `1793` adalah **ukuran file dalam bytes**, bukan UID

### 4.5 `/var/log` dan `auth.log`

**T:** Di mana mencari bukti percobaan login SSH yang gagal?  
**J:** `/var/log/auth.log` di Debian/Ubuntu, `/var/log/secure` di RHEL/CentOS/Fedora. File ini mencatat autentikasi: SSH, `sudo`, PAM.

Anatomi satu baris log:

- Timestamp ISO 8601 dengan `+00:00` berarti **UTC** (WIB = UTC+7). Selalu sadari timezone log saat forensik
- Hostname, nama proses dengan PID (`sshd[1346]`), lalu pesan
- Entry `sudo`: memuat `TTY`, `PWD`, `USER`, dan `COMMAND` persis. Ini audit trail command yang dijalankan sebagai root
- Entry `CRON`: scheduled task otomatis, normal
- Login gagal memunculkan `pam_unix ... authentication failure` (dengan `rhost` IP asal) dan `Failed password for <user> from <IP> port <port> ssh2`

**T:** Berapa jumlah percobaan gagal, dan apakah angkanya akurat?  
**J:** `grep "Failed password"` menghasilkan 2 baris, tapi hanya **1** event asli. Baris lain adalah jejak `sudo` dari command `sudo grep` itu sendiri, karena teks command ikut tercatat dan mengandung "Failed password". Filter ke proses `sshd` memisahkan event asli dari jejak itu.

**T:** Bentuk log mana yang paling berguna mendeteksi penebakan nama user?  
**J:** `Invalid user <nama> from <IP>`, yang muncul saat nama tidak ada di sistem. `Connection reset by invalid user ... [preauth]` berarti koneksi putus sebelum autentikasi selesai (dalam kasus ini pengguna membatalkan karena salah ketik username). Satu baris adalah noise. Serangan terlihat dari **pola**: banyak nama berbeda, IP sama, rentang waktu singkat.

- Log berotasi lewat `logrotate` (`auth.log.1`, dan seterusnya), jadi script yang hanya membaca `auth.log` bisa melewatkan riwayat lama
- Gap tanggal di log (Agustus ke Oktober) berasal dari VM yang mati; dalam forensik gap selalu perlu dipertanyakan

| File | Isi | Dibaca dengan |
|---|---|---|
| `/var/run/utmp` | siapa login sekarang | `who`, `w` |
| `/var/log/wtmp` | riwayat login/logout, reboot | `last` |
| `/var/log/btmp` | riwayat login gagal | `sudo lastb` |

Ketiganya file binary, tidak bisa dibaca dengan `cat`. Hasil `lastb` di VM ini:

| Waktu (UTC) | Entri | Arti |
|---|---|---|
| 1 Agu 12:57 | `linuxauf tty1` | gagal login di console |
| 6 Agu 12:24 | `linuxauf ssh:notty` | tes salah password dari host |
| 15 Okt 12:27 | `UNKNOWN tty1` | gagal di console, nama tidak terbaca |
| 15 Okt 12:28 | `endpoint ssh:notty` | salah ketik username lewat SSH |
| 16 Okt 08:53 | `UNKNOWN tty1` | gagal di console |

- Nama user dipotong 8 karakter; `ssh:notty` berarti login SSH gagal sebelum mendapat terminal
- `lastb` mencatat 5 percobaan, sedangkan `grep "Failed password"` hanya menangkap 1. Satu pola grep bisa melewatkan event; cross-check dengan sumber lain
- Output `last` menunjukkan gap sekitar 70 hari tanpa login atau reboot (6 Agustus 14:31 sampai 15 Oktober 12:26), cocok dengan project yang terbengkalai

**T:** Bagaimana membersihkan history command?  
**J:** `history -c` membersihkan history di memori sesi aktif, `history -w` menulis ulang `~/.bash_history` sehingga kosong. History bash bersifat per user, ditulis saat sesi ditutup, dan bisa dihapus pemiliknya (teknik penghapusan jejak, MITRE T1070.003). Jangan mengetik password di command line karena akan tercatat. Catatan command semua sesi diambil dari log `sudo` di `auth.log` dan dari auditd.

### 4.6 Console, terminal, tty, pts

**T:** Apa bedanya console, terminal, tty, dan pts?  
**J:** Analogi TV:

- **Console** = tombol di badan TV: terminal yang menempel langsung ke mesin (di VM, jendela VirtualBox). Tercatat sebagai `tty1`
- **SSH** = remote: terminal dari jauh, tercatat sebagai `pts/N` dengan IP asal
- **Terminal** = istilah umum untuk antarmuka teks tempat mengetik command (console dan SSH sama-sama terminal)
- **TTY** = penanda jalur: dari mana sebuah command datang

- `tty` mencetak nama terminal yang dipakai (`/dev/tty1` atau `/dev/pts/0`)
- Nomor `pts` dibagikan berurutan; dua sesi dari host menjadi `pts/0` dan `pts/1`. `who` menunjukkan `192.168.56.1` untuk keduanya
- Sesi console tidak punya IP di kolom `FROM` (`-`); sesi SSH punya IP asal
- Cara akses console: klik jendela VirtualBox VM, login, `tty` harus `/dev/tty1`. Lepas fokus keyboard dengan **Right Ctrl**
- Di `w`, sesi `tty1` yang menganggur lama bukti nyata risiko sesi tidak dijaga. Idle timeout dibahas di item E checklist

### 4.7 `/proc`

**T:** Dari mana `ps` dan `top` mengambil data proses?  
**J:** Dari `/proc`, bukan `/tmp`. `/proc` adalah **virtual filesystem**: kernel membuat isinya saat dibaca, tidak tersimpan di disk. Setiap proses punya direktori sesuai PID (`/proc/6092/`). `ps` dan `top` hanya membaca dan merapikan isi file di sana.

| Command | Sumber data |
|---|---|
| `ps`, `top` | `/proc/<PID>/...` |
| `uptime` | `/proc/uptime`, `/proc/loadavg` |
| `free` | `/proc/meminfo` |
| `lscpu` | `/proc/cpuinfo`, `/sys` |

**T:** Kenapa `/proc/cpuinfo` ukurannya 0 padahal ada isinya?  
**J:** Tidak ada data tersimpan. Kernel membangun isinya saat file dibuka, jadi ukuran file yang dilaporkan 0. Analogi: speedometer yang angkanya muncul sesuai kondisi saat itu. Tanggal di `ls -l` juga bukan waktu pembuatan. `/proc` tidak menempati partisi mana pun.

- `echo $$` = PID shell (contoh 6092), cocok dengan `Pid` di `/proc/$$/status`
- Field `status`: `Name` (bash), `State` (S = sleeping, menunggu input), `PPid` (parent process)
- Rantai proses: shell bersarang (6466) → login shell (6448) → proses lain (6447). PID berubah setiap login karena kernel membagikan PID baru ke proses baru
- `/proc/<PID>/exe` adalah symlink ke program yang berjalan (`/usr/bin/bash`)
- `/proc/<PID>/cmdline`: command lengkap. `-bash` (dengan strip) = login shell; `bash` = shell non-login (misal hasil mengetik `bash`). `exit` dari login shell mencetak `logout`
- Blue team: attacker bisa menghapus file malware aslinya sementara proses tetap hidup; `/proc/<PID>/exe` akan menampilkan `(deleted)`

**T:** Apakah `cat /proc/$!/cmdline` menampilkan `sleep 300` atau `sleep300`?  
**J:** `sleep300` (menempel). Argumen di `cmdline` dipisah karakter **NULL** (byte nol, tak terlihat), bukan spasi. `tr '\0' ' '` mengubah NULL menjadi spasi sehingga tampil `sleep 300`. Spasi di `' '` milik `tr`, bukan penyebab pemisah asli.

### 4.8 `/sys` dan `/proc/sys`

- `/proc` = dashboard (keadaan sistem yang berjalan); `/sys` = daftar komponen (perangkat dan driver, tersusun rapi). Keduanya virtual
- `ls /sys` berisi `block`, `bus`, `class`, `dev`, `devices`, `firmware`, `fs`, `hypervisor`, `kernel`, `module`, `power`
- `/sys/class/net/enp0s3/address` = MAC address, cocok dengan `ip a` dan setting VirtualBox
- `/sys/class/net/enp0s3/operstate` = status link (`up`)
- `/sys/block/sda/size` = 83886080 sektor. 83886080 × 512 byte = 40 GiB, cocok dengan disk VM
- `ip`, `lsblk` hanya membaca dan merapikan data dari `/sys`
- **`/proc/sys`** (bukan `/sys`) berisi setting kernel yang bisa diubah. `/proc/sys/net/ipv4/ip_forward` bernilai `0`, jadi item "disable IP forwarding" sudah PASS di kondisi awal
- `sda` = disk pertama, angka = nomor partisi (`sda1` sampai `sda7`)

Hasil `lsblk`:

| Partisi | Ukuran | Mount |
|---|---|---|
| `sda1` | 1M | (spacer GRUB) |
| `sda2` | 15G | `/` |
| `sda3` | 4G | `[SWAP]` |
| `sda4` | 3G | `/tmp` |
| `sda5` | 5G | `/var` |
| `sda6` | 5G | `/var/log` |
| `sda7` | 5G | `/home` |

`sr0` (1024M, tipe rom) adalah drive CD/DVD virtual, bukan disk sungguhan.

### 4.9 Symlink dan merged-usr

- **Symlink** = shortcut: file kecil yang hanya berisi alamat file lain. Di `ls -l` ditandai `l` di depan dan `->` menuju target
- Ukuran symlink adalah panjang teks alamatnya (`pintasan.txt` = 24 karakter), bukan isi target
- `/bin`, `/lib`, `/sbin` di Ubuntu 24.04 adalah symlink (`usr/bin`, `usr/lib`, `usr/sbin`, ukuran 7, 7, 8). Ini **merged-usr**; path lama dipertahankan agar script `#!/bin/bash` tetap jalan
- Link ini **relative symlink** (tanpa `/` awal), dihitung dari lokasi link; `pintasan.txt` memakai alamat absolut
- `/bin.usr-is-merged` adalah direktori asli (`d`); isinya belum dilihat
- `find /bin` hanya mencetak `/bin` (tidak mengikuti symlink), sedangkan `find /bin/` menelusuri isinya karena `/` di akhir memaksa path diperlakukan sebagai direktori
- Risiko: pengecekan seperti `find /bin -perm -0002` mengembalikan kosong dan terbaca PASS padahal tidak memeriksa apa pun (false negative). Solusi: tulis path asli (`/usr/bin`), pakai `/bin/`, atau opsi `-L`
- Attacker juga bisa memakai symlink di direktori writable seperti `/tmp`; Ubuntu punya proteksi `fs.protected_symlinks`

### 4.10 `/etc/fstab` dan mount

**T:** File apa yang menentukan partisi mana dipasang ke folder mana saat boot?  
**J:** `/etc/fstab` (file systems table). Ditulis installer (`curtin`) berdasarkan partisi yang dibuat. `/etc` adalah tempat konfigurasi sistem berbentuk teks: program membaca file di sini, perilakunya diubah dengan mengedit file.

- Enam kolom: device, mount point, tipe, options, dump, pass
- Device ditulis dengan **UUID** karena nama `sda` bisa bergeser jika urutan disk berubah
- Baris swap: mount point `none`, tipe `swap`, opsi `sw`
- Baseline: semua partisi memakai `defaults` (artinya `rw`, `suid`, `dev`, `exec`, `auto`, `nouser`, `async`)
- `findmnt /tmp` menunjukkan kenyataan: `rw,relatime`, tanpa `noexec`
- `fstab` adalah **rencana tertulis**, `findmnt` adalah **keadaan yang berjalan**; keduanya harus sinkron
- Salah ketik di `fstab` bisa membuat VM gagal boot, jadi backup dulu

### 4.11 `/boot`, `/opt`, `/mnt`, `/media`, `/srv`, `/run`

**T:** `/boot` ada di partisi mana dan apa isinya?  
**J:** Di partisi root (`sda2`): `df -h` tidak punya baris mount `/boot`. Isinya:

| File | Fungsi |
|---|---|
| `vmlinuz-6.8.0-136-generic` | kernel Linux |
| `initrd.img-...` | filesystem mini di RAM saat awal boot |
| `config-...`, `System.map-...` | konfigurasi build dan peta alamat fungsi kernel |
| `grub/` | konfigurasi bootloader |
| `vmlinuz`, `initrd.img` (`->`) | symlink ke versi aktif |

- Tulisan "Begin: Loading essential drivers" dicetak oleh tahap `initrd.img`
- `vmlinuz` dan `System.map` berpermission `-rw-------` (dugaan: alamat fungsi kernel berguna untuk exploit)
- `/boot` adalah tempat persistence tingkat rendah; perubahan perlu dicurigai

**T:** Direktori mana yang kosong, dan di mana program hasil `apt` tersimpan?  
**J:** `/opt`, `/mnt`, `/media`, `/srv` semuanya kosong, jadi baseline untuk audit. Koreksi atas dugaan awal: program dari `apt` masuk ke `/usr` (buktinya `/proc/$$/exe -> /usr/bin/bash`), bukan `/opt`. `/opt` untuk software pihak ketiga dengan file sendiri; `/mnt` mount manual; `/media` mount otomatis; `/srv` data layanan.

**T:** Apa itu `tmpfs` dan apa yang terjadi pada isinya saat reboot?  
**J:** Filesystem yang hidup di RAM, bukan disk. Isinya hilang saat reboot. Di `df -h` terlihat `/run`, `/dev/shm`, `/run/lock`, `/run/user/1000`.

- Ukuran 2.0G di `/dev/shm` adalah batas maksimal, RAM terpakai sesuai isinya
- `utmp` tinggal di `/run` (RAM), makanya `who` hanya menampilkan sesi aktif; riwayat ada di `wtmp` (disk)
- Forensik: file attacker di `/dev/shm` lenyap saat reboot. Jangan reboot mesin yang dicurigai sebelum bukti diambil

### 4.12 `/dev/shm` baseline

```
findmnt /dev/shm  ->  rw,nosuid,nodev,inode64
```

- `TARGET` = folder tempat dipasang, `SOURCE` = asal data, `FSTYPE` = jenis filesystem, `OPTIONS` = aturan mount
- `rw` baca-tulis; `nosuid` dan `nodev` sudah aktif bawaan; `inode64` cara penomoran file (tidak terkait keamanan)
- **Tidak ada `noexec`**: bawaan Ubuntu tidak mengunci karena sebagian program memakai `/dev/shm` untuk shared memory. CIS merekomendasikan menambahkannya, tugas Fase 3
- Terbukti: script uji di `/dev/shm` berhasil dijalankan (`bisa jalan`). `/dev/shm` bukan bagian partisi `/tmp`, jadi harus dikunci terpisah

### 4.13 Sintaks Bash yang dipakai

| Simbol | Arti | Contoh |
|---|---|---|
| `\|` | output kiri jadi input command kanan | `echo "halo" \| tr 'a' 'o'` |
| `<` | isi file kanan jadi input command kiri | `tr 'a' 'o' < asli.txt` |
| `>` | output command kiri ditulis ke file kanan (menimpa) | `echo "halo" > asli.txt` |
| `>>` | menambah di akhir file | `echo x >> log.txt` |
| `;` | jalankan berurutan, apa pun hasilnya | `tr ... ; echo` |
| `&&` | jalankan kanan hanya jika kiri sukses | `history -c && history -w` |
| `&` | jalankan di background | `sleep 300 &` |
| `$$` | PID shell sendiri | `/proc/$$/status` |
| `$!` | PID proses background terakhir | `/proc/$!/cmdline` |

**T:** Apakah `<` kebalikan dari `|`?  
**J:** Arah aliran data memang berlawanan (`|` ke kanan, `<` ke kiri), tetapi pasangan kebalikan yang tepat adalah `<` dan `>` (sama-sama berhubungan dengan file). `|` menyambung dua command tanpa file perantara. Cara ingat: lihat arah panah. `>` ke file, `<` dari file, `|` antar command.

- `tr 'dicari' 'pengganti'` mengganti karakter (translate). `tr '\0' ' '` mengganti NULL dengan spasi. Tulisannya backslash nol (`\0`), bukan `/0`
- `tr` tidak menerima nama file langsung, sehingga input dialirkan dengan `<` atau `|`
- Bash menjalankan command berurutan per baris; paste beberapa baris sama dengan mengetik satu per satu
- `~` adalah home directory; alias `ll` = `ls -la`
- `echo -e` menafsirkan escape seperti `\n`; `bash` bersarang dikeluarkan dengan `exit`

```bash
echo "halo" > ~/asli.txt
tr 'a' 'o' < ~/asli.txt
echo "halo" | tr 'a' 'o'
ln -s ~/asli.txt ~/pintasan.txt
ls -l ~/pintasan.txt
sleep 300 &
cat /proc/$!/cmdline
tr '\0' ' ' < /proc/$!/cmdline; echo
kill $!
```

## 5. Fase 3: Hardening Manual (Progress Awal)

### 5.1 Aturan kerja Fase 3

- Urutan mengikuti kategori checklist: A Filesystem, B Services, C Network, D Logging, E Access, F Permissions
- Satu item per giliran: konfigurasi, verifikasi, screenshot "sesudah"
- Screenshot "sebelum" sudah ada dari Fase 2: `fase2/23` (`/tmp`), `fase2/27` dan `fase2/28` (`/dev/shm`)
- Nomor screenshot reset: `docs/screenshots/fase3/00-...`
- Jika konfigurasi salah, snapshot `fresh-install-before-hardening` bisa dipakai kembali

### 5.2 Item A: mount option `/tmp`

Baseline: `rw,relatime` (opsi `defaults` di `fstab`). Target: menambah `nodev`, `nosuid`, `noexec`.

Alasan ketiganya:

- `noexec`: file di `/tmp` tidak bisa dijalankan langsung
- `nosuid`: SUID diabaikan, mencegah salinan program berhak root dipakai menaikkan hak akses
- `nodev`: device file diabaikan; membuat device file butuh root, jadi ini lapisan tambahan jika akun root disalahgunakan
- `/tmp` hanya untuk file sementara, jadi tidak ada alasan sah menjalankan program, SUID, atau device dari sana

**Langkah 1: backup `fstab`**

```bash
sudo cp /etc/fstab /etc/fstab.bak
```

- `cp` menyalin file; `sudo` diperlukan karena menulis di `/etc`
- Alasan: salah ketik di `fstab` bisa membuat VM gagal boot. Mengembalikan: `sudo cp /etc/fstab.bak /etc/fstab`

**Langkah 2: edit baris `/tmp`**

```bash
sudo nano /etc/fstab
```

- Ubah kolom ke-4 saja: dari `defaults` menjadi `defaults,nodev,nosuid,noexec`
- Edit `fstab` (bukan hanya mount manual) karena `fstab` dibaca saat boot; perubahan mount manual hilang setelah reboot
- Opsi baru ditaruh **setelah** `defaults` karena opsi dibaca kiri ke kanan dan yang belakang menimpa; di depan, `defaults` bisa mengembalikan `exec`
- Pakai koma tanpa spasi (spasi memisahkan kolom). Jangan ubah `0 1`, UUID, atau baris lain
- Simpan: `Ctrl+O`, `Enter`, keluar `Ctrl+X`

**Langkah 3: terapkan tanpa reboot**

```bash
sudo mount -o remount /tmp
sudo systemctl daemon-reload
```

- `mount -o remount` menerapkan ulang opsi ke partisi yang sedang terpasang; opsi diambil dari `fstab`
- Mengedit `fstab` tidak mengubah sistem yang sedang berjalan
- `daemon-reload` menyuruh systemd membaca ulang `fstab` (systemd menyimpan salinan di memori); jalankan jika muncul pesan yang menyarankannya

**Langkah 4: verifikasi**

```bash
grep ' /tmp ' /etc/fstab
findmnt /tmp
```

- `grep ' /tmp '` dengan spasi di kiri-kanan agar tidak menangkap path seperti `/var/tmp`. Hasil dua baris: komentar berawalan `#` dan baris aslinya
- Baris tanpa `#` harus memuat `defaults,nodev,nosuid,noexec`
- `findmnt /tmp` menunjukkan kenyataan: OPTIONS harus memuat `nosuid,nodev,noexec`
- Screenshot: `docs/screenshots/fase3/00-fstab-tmp-sesudah-edit`

**Langkah 5: tes `noexec`**

```bash
echo -e '#!/bin/bash\necho bisa jalan' > /tmp/tes.sh
chmod +x /tmp/tes.sh
/tmp/tes.sh
bash /tmp/tes.sh
rm /tmp/tes.sh
```

- Baris 1 membuat script uji yang aman (hanya mencetak satu kalimat) sebagai pembuktian
- Baris 2 memberi izin eksekusi; tanpa ini tes tidak adil karena gagal oleh permission, bukan `noexec`
- Baris 3 harus `Permission denied`; baris 4 tetap `bisa jalan` karena bash membaca file sebagai teks
- Dua cara dites untuk menunjukkan bahwa `noexec` melarang eksekusi langsung, bukan pagar total
- Screenshot: `docs/screenshots/fase3/01-tes-eksekusi-tmp-sesudah`

Status: instruksi langkah 1 sampai 5 sudah diberikan; hasil eksekusi belum dilaporkan pada saat catatan ini dibuat.

### 5.3 Item berikutnya

- Mount option `/dev/shm`: tambah `noexec`, ulangi tes eksekusi (bandingkan dengan `fase2/28`)
- AIDE, lalu kategori B sampai F sesuai checklist
- SSH hardening mencakup idle timeout (terkait sesi `tty1` yang menganggur)
- Item C `ip_forward = 0` sudah PASS pada baseline, cukup didokumentasikan
- Item F permission `/etc/passwd` (644) dan `/etc/shadow` (640) sudah sesuai pada baseline

## 6. Catatan Desain Audit Script (Fase 4)

Kerangka punya 8 function: `check_ssh_root_login`, `check_ssh_password_auth`, `check_unnecessary_services`, `check_ip_forwarding`, `check_passwd_permissions`, `check_shadow_permissions`, `check_password_policy`, `check_auditd_status`. Pola: baca konfigurasi dengan `grep`, bandingkan dengan nilai yang diharapkan, panggil `log_result`.

Pelajaran dari Fase 2 yang harus masuk desain:

- **False positive**: `grep` mentah menghitung jejak `sudo` dari command itu sendiri. Filter ke proses `sshd`; script yang dijalankan dengan `sudo` juga meninggalkan jejak di `auth.log`
- **False negative**: satu pola tidak cukup. Minimal tiga: `Failed password`, `Invalid user`, `authentication failure`, plus cross-check dengan `lastb`
- Deteksi brute-force = hitungan **per IP** (`rhost`) dalam rentang waktu dengan threshold, bukan sekadar ada atau tidaknya satu event
- Log berotasi (`auth.log.1`), timestamp UTC, dan gap tanggal perlu diperhitungkan
- Path symlink (`/bin`) membuat `find` melewatkan isinya; gunakan path asli atau `/`
- Check yang mengembalikan "kosong" harus bisa dibedakan dari "tidak ada yang diperiksa" (pertanyaan terbuka: bagaimana script memastikan pemeriksaannya benar-benar berjalan; belum dibahas)
- Ide check tambahan: deteksi brute-force SSH, anggota grup `sudo`, `noexec` di `/dev/shm`

Fakta baseline yang akan muncul di laporan "before":

| Item | Kondisi awal |
|---|---|
| `/tmp` mount option | `rw,relatime` (tanpa `nodev,nosuid,noexec`) |
| `/dev/shm` | `rw,nosuid,nodev` (tanpa `noexec`) |
| `ip_forward` | `0` (PASS) |
| `/etc/passwd`, `/etc/shadow` | 644 `root:root`, 640 `root:shadow` (PASS) |
| anggota grup `sudo` | hanya `linuxaufa` |
| SSH | password authentication aktif |

## 7. Dokumen Pendukung dan Cara Kerja

### 7.1 Dokumen yang dibuat

- `checklist/cis-benchmark-checklist.md`: 6 kategori (A sampai F), 23 item, item A (partisi) dan F (permission passwd/shadow) sudah dicentang, ditambah tabel progress log
- `project-profile-draft.md`: draft isian template portofolio (24 pertanyaan). Memakai penanda `[SELESAI]`, `[BERJALAN]`, `[RENCANA]`, `[ISI SETELAH RUN]` agar hanya fakta yang dikerjakan yang diklaim. Angka dan temuan audit sengaja tidak dikarang. Perlu revisi periode (jeda Agustus sampai Oktober) dan update tag
- Progress checklist (19 Oktober 2026): 22 item selesai, 3 berjalan, 47 belum dari 72 item (31%)

Pemetaan MITRE ATT&CK yang bersifat konseptual:

| Area di project | Teknik |
|---|---|
| Brute-force SSH di `auth.log` | T1110.001 Password Guessing |
| Akses remote | T1021.004 SSH |
| Audit penyalahgunaan `sudo` | T1548.003 Sudo and Sudo Caching |
| Cron | T1053.003 Cron |
| `noexec` di `/tmp` | T1105 Ingress Tool Transfer (mitigasi) |
| Audit permission file | T1222.002 Linux File and Directory Permissions Modification |
| Penghapusan history | T1070.003 Clear Command History |
| Audit akun lokal dan grup sudo | T1078.003 Local Accounts |

### 7.2 Preferensi cara belajar

- Jawaban singkat dan berpoin, tidak panjang; "detailin" untuk pendalaman
- Command selalu dalam code block agar bisa disalin, dengan penjelasan fungsi dan alasan
- Satu pertanyaan per giliran; boleh menjawab "gabisa" untuk dijelaskan dari dasar
- Screenshot tanpa `.png` agar tidak ganda dengan ekstensi di VS Code
- Nama pengguna lengkap tidak dicantumkan di catatan yang akan dipublikasikan

## 8. Referensi Command dan Flag

### 8.1 Windows (PowerShell)

| Command | Fungsi | Bagian penting |
|---|---|---|
| `mkdir a, b` | membuat banyak folder sekaligus | koma memisah; parent dibuat otomatis |
| `tree /f` | menampilkan struktur folder | `/f` menampilkan file juga |
| `ssh user@ip` | koneksi SSH ke VM | `linuxaufa@192.168.56.107` |
| `systeminfo` | informasi sistem | cari "Virtualization-based security" |

### 8.2 Linux: sistem, jaringan, dan sesi

| Command | Fungsi | Bagian penting |
|---|---|---|
| `ip a` | daftar interface dan IP | `a` = address |
| `sudo` | menjalankan sebagai root | tercatat di `auth.log` |
| `id user` | UID, GID, grup | |
| `getent group nama` | anggota grup | lebih modern dari membaca `/etc/group` |
| `tty` | nama terminal aktif | |
| `who`, `w` | sesi yang sedang login | `w` menambah IDLE dan proses |
| `last` | riwayat login sukses dan reboot | membaca `wtmp` |
| `sudo lastb` | riwayat login gagal | membaca `btmp` |
| `dmesg` | pesan kernel sejak boot | butuh `sudo` di Ubuntu 24.04 |
| `history -c` / `-w` | hapus history memori / tulis ulang file | `-c` clear, `-w` write |
| `exit`, `logout` | keluar dari shell | login shell mencetak `logout` |
| `alias ll='ls -la'` | alias format panjang | |

### 8.3 Linux: file dan teks

| Command | Fungsi | Bagian penting |
|---|---|---|
| `ls -l` | format panjang | permission, owner, ukuran |
| `ls -a` | tampilkan file tersembunyi | nama berawalan titik |
| `ls -d` | direktori itu sendiri, bukan isinya | dipakai untuk melihat symlink `/bin` |
| `cat file` | menampilkan isi file | |
| `head -N`, `tail -N` | N baris pertama / terakhir | `tail -20` dipakai di log |
| `grep "pola" file` | menyaring baris | `-i` abaikan besar/kecil huruf |
| `wc -l` | menghitung baris | `-l` = lines |
| `tr 'a' 'b'` | mengganti karakter | `\0` = NULL |
| `echo -e` | menafsirkan escape | `\n` baris baru |
| `ln -s target link` | membuat symlink | urutan: target dulu |
| `rm file` | menghapus file | |
| `chmod +x file` | memberi izin eksekusi | |
| `find path` | mencari file | `-perm -0002` = world-writable; tidak mengikuti symlink |
| `sleep N &` | diam N detik di background | `&` background |
| `kill $!` | menghentikan proses background terakhir | |

### 8.4 Linux: disk, mount, dan konfigurasi

| Command | Fungsi | Bagian penting |
|---|---|---|
| `lsblk` | daftar disk dan partisi | kolom MOUNTPOINTS |
| `df -h` | pemakaian filesystem | `-h` human-readable |
| `findmnt path` | mount option yang aktif | kolom OPTIONS |
| `cp a b` | menyalin file | dipakai membuat backup `fstab` |
| `nano file` | editor teks | `Ctrl+O` simpan, `Ctrl+X` keluar |
| `mount -o remount path` | menerapkan ulang opsi mount | opsi dari `fstab` |
| `systemctl daemon-reload` | systemd membaca ulang konfigurasi | |

## 9. Koreksi Selama Sesi

| Pernyataan awal | Fakta atau koreksi |
|---|---|
| VM stuck karena shutdown paksa (fsck) | Shutdown dilakukan bersih dengan `sudo shutdown -h now` |
| Konflik Hyper-V/WSL penyebab stuck | Fitur itu tidak aktif; yang aktif VBS/HVCI |
| Paravirtualization VM berbeda dari VM lain | Keduanya KVM Paravirtualization |
| Angka `1793` di `ls -l` adalah UID | Itu ukuran file dalam bytes |
| `Failed password` = 2 percobaan | 1 event asli + 1 jejak `sudo grep` |
| Hanya 1 login gagal | `lastb` mencatat 5; grep satu pola melewatkan sebagian |
| `/tmp` hanya bisa diakses admin | Permission 1777: semua user bisa menulis |
| Proses tersimpan di `/tmp` | Proses ada di RAM; data lewat `/proc` |
| `/opt` paling banyak dipakai | Kosong; `apt` memasang ke `/usr` |
| Console adalah `tty`, terminal adalah `pts` | Keduanya terminal; console = terminal lokal |
| `pts/1` berasal dari Kali | `who` menunjukkan IP host; Kali sesi lain |
| Alias `ll` = `ls -ls` | `ls -la` |
| `tr '/0'` | Karakter NULL ditulis `\0` |
| Sticky bit tidak berguna jika ada `noexec` | Dua fungsi berbeda, keduanya tetap berlaku |
| `<` kebalikan `\|` | Pasangan kebalikan `<` adalah `>` |

## 10. Insight dan Lesson Learned

- Log dan `who` lebih dipercaya daripada ingatan: IP di log menunjukkan sesi sebenarnya
- Satu pola `grep` bisa melewatkan event; selalu cross-check sumber lain (`auth.log`, `lastb`, `wtmp`)
- Event tunggal bukan serangan; deteksi bergantung pada pola dan threshold
- Data dipisah berdasarkan sensitivitas (passwd vs shadow), bukan semuanya dikunci
- Mount option dipasang per partisi; partisi terpisah memberi kontrol granular dan containment
- Defense in depth: sticky bit, `noexec`, `nosuid`, `nodev` mengurus hal berbeda dan saling melengkapi
- `noexec` pelambat, bukan penutup total (`bash script` tetap jalan)
- `/dev/shm` dan `/tmp` sama-sama writable; mengunci satu belum cukup
- Bukti di RAM (`tmpfs`) hilang saat reboot; jangan reboot sebelum mengumpulkan bukti
- `fstab` adalah rencana, `findmnt` adalah kenyataan; selalu cek keduanya
- Backup sebelum mengedit file konfigurasi kritis, dan snapshot VM sebagai jalan kembali
- Pesan kernel muncul di console; pantau lewat log terpusat, bukan hanya sesi SSH
- Troubleshooting jujur: catat apa yang terbukti dan apa yang masih dugaan
- Jeda project dan gap log perlu dicatat; gap log dalam forensik harus dipertanyakan
- Baseline sebelum hardening (nilai default) adalah bukti "before" yang berharga

## 11. Glossary

| Istilah | Arti |
|---|---|
| attack surface | seluruh titik yang bisa diserang |
| FHS | Filesystem Hierarchy Standard, standar struktur direktori Linux |
| mount option | aturan perilaku partisi saat dipasang (`noexec`, `nosuid`, `nodev`) |
| sticky bit | penanda `t` di direktori: hanya pemilik yang boleh menghapus filenya |
| SUID | bit yang membuat program berjalan dengan hak pemiliknya |
| UID / GID | nomor identitas user / grup |
| GECOS | field nama lengkap di `/etc/passwd` |
| symlink | file penunjuk ke file atau direktori lain |
| merged-usr | `/bin`, `/sbin`, `/lib` menjadi symlink ke `/usr` |
| virtual filesystem | isinya dibuat kernel saat dibaca (`/proc`, `/sys`) |
| tmpfs | filesystem di RAM |
| PID / PPid | nomor proses / nomor proses induk |
| console | terminal yang menempel langsung ke mesin |
| tty / pts | nama terminal lokal / terminal pseudo (SSH, emulator) |
| login shell | shell yang dibuat saat login (`-bash`) |
| initramfs / initrd | filesystem mini di RAM saat awal boot |
| GRUB | bootloader |
| soft lockup | peringatan watchdog kernel bahwa CPU tampak tidak bergerak |
| VBS / HVCI | virtualization-based security / hypervisor-protected code integrity |
| WHP | Windows Hypervisor Platform |
| snapshot | save point kondisi VM |
| subiquity | installer teks Ubuntu Server |
| curtin | komponen installer yang menulis `fstab` dan menginstal sistem |
| UTC | zona waktu acuan log (WIB = UTC+7) |
| preauth | tahap sebelum autentikasi selesai |
| brute force | menebak kredensial berulang kali |
| username enumeration | menebak nama user yang valid |
| false positive / negative | alarm palsu / event nyata yang lolos |
| defense in depth | pertahanan berlapis |
| baseline | kondisi awal sebelum hardening |
| AIDE | tool filesystem integrity checking |
| CIS Benchmark | panduan konfigurasi aman |

## 12. Lampiran: Daftar Screenshot

Fase 1 (`docs/screenshots/fase1/`):

```
00 struktur-folder-project
01 vm-name-dan-iso
02 unattended-installation-disabled
03 hardware-ram-cpu
04 virtual-hard-disk-40gb
05 network-adapter1-nat
06 network-adapter2-hostonly
07 pemilihan-bahasa
08 installer-update-available
09 keyboard-configuration
10 installation-type
11 network-configuration-2-adapter
12 proxy-configuration
13 archive-mirror-configuration
14 guided-storage-custom-layout-selected
15 storage-config-setelah-root-dibuat
16 add-partition-root-contoh
17 storage-configuration-final-summary
18 confirm-destructive-action
19 profile-configuration-kosong
20 upgrade-ubuntu-pro
21 ssh-configuration
22 featured-server-snaps
23 installation-complete
24 first-login-berhasil
25 ip-a-verifikasi-network
26 ssh-dari-windows-berhasil
27 take-snapshot-dialog
28 snapshot-berhasil-dibuat
```

Fase 2 (`docs/screenshots/fase2/`):

```
00 permission-passwd-vs-shadow
01 cek-uid-gid-linuxaufa
02 cek-anggota-grup-sudo
03 clear-history
04 auth-log-tail-20
05 ssh-percobaan-gagal-powershell
06 auth-log-bukti-percobaan-gagal
07 hitung-dan-lihat-failed-password
08 tty-dan-who-dua-sesi
09 last-dan-lastb
10 grep-endpoint-invalid-user
11 akses-console-tty1
12 dmesg-soft-lockup-via-ssh
13 proc-ls-dan-pid-shell
14 proc-status-dan-cpuinfo
15 proc-exe-dan-cmdline
16 pid-bash-bersarang-dan-tr
17 proc-cmdline-pemisah-null
18 latihan-symlink
19 bin-lib-sbin-symlink
20 find-symlink-trailing-slash
21 sys-class-net-dan-block
22 lsblk-partisi-fase1
23 fstab-dan-findmnt-tmp
24 isi-boot
25 direktori-kosong-dan-df-h
26 isi-run
27 findmnt-dev-shm
28 tes-eksekusi-dev-shm-before
29 latihan-redirect-dan-pipe
```

Fase 3 (`docs/screenshots/fase3/`, menunggu hasil):

```
00 fstab-tmp-sesudah-edit
01 tes-eksekusi-tmp-sesudah
```

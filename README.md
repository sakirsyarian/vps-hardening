# 🔐 Panduan Hardening VPS untuk Pemula

### CentOS · Ubuntu/Debian · Arch · openSUSE + Windows | Lindungi VPS dari Bot Brute Force

---

## Pendahuluan

Ketika kamu menyewa VPS baru, server tersebut langsung terekspos ke internet. Dalam hitungan menit, bot-bot otomatis sudah mulai mencoba login menggunakan kombinasi username dan password secara berulang-ulang (brute force attack).

Panduan ini akan membantumu mengamankan VPS dari ancaman tersebut, mulai dari awal hingga selesai, untuk:

- VPS dengan sistem operasi **CentOS / RHEL / AlmaLinux / Rocky Linux**
- VPS dengan sistem operasi **Ubuntu / Debian**
- VPS dengan sistem operasi **Arch Linux / Manjaro**
- VPS dengan sistem operasi **openSUSE Leap / Tumbleweed**
- Laptop/PC dengan sistem operasi **Windows**

### Konvensi Panduan Ini

Setiap langkah yang berbeda antar distro akan ditampilkan dalam **tab per distro** seperti ini:

> 🔴 **CentOS/RHEL** | 🟠 **Ubuntu/Debian** | 🔵 **Arch Linux** | 🟢 **openSUSE**

Langkah yang sama untuk semua distro ditampilkan tanpa label.

### Urutan Langkah

| Urutan | Langkah                     | Tujuan                               |
| ------ | --------------------------- | ------------------------------------ |
| 1      | Update sistem               | Patch keamanan terbaru               |
| 2      | Buat user non-root          | Hindari pakai akun root langsung     |
| 3      | Generate SSH Key di Windows | Siapkan metode login yang aman       |
| 4      | Copy SSH Key ke VPS         | Izinkan laptop kamu masuk ke VPS     |
| 5      | Harden konfigurasi SSH      | Matikan login password & root        |
| 6      | Konfigurasi Firewall        | Buka port baru, tutup port 22        |
| 7      | Install sshguard / fail2ban | Auto-ban IP yang mencoba brute force |
| 8      | Auto Security Updates       | Server selalu update otomatis        |

> ⚠️ **PENTING:** Ikuti urutan langkah ini dengan benar! Jangan loncat-loncat. Kalau salah urutan, kamu bisa terkunci keluar dari VPS sendiri.

---

## Step 1 — Update Sistem

Langkah pertama sebelum apapun adalah memastikan semua paket sistem sudah diupdate ke versi terbaru, termasuk patch keamanan.

🔴 **CentOS / RHEL / AlmaLinux / Rocky Linux:**
```bash
dnf update -y
```

🟠 **Ubuntu / Debian:**
```bash
apt update && apt upgrade -y
```

🔵 **Arch Linux / Manjaro:**
```bash
pacman -Syu
```

🟢 **openSUSE Leap / Tumbleweed:**
```bash
zypper refresh && zypper update -y
```

> 💡 Proses ini mungkin butuh beberapa menit tergantung koneksi internet VPS kamu.

---

## Step 2 — Buat User Non-Root

Berbahaya jika kamu selalu login sebagai root. Kalau akun root berhasil dibobol, penyerang punya kendali penuh atas server. Solusinya: buat user biasa yang punya akses sudo (admin terbatas).

🔴 **CentOS / RHEL / AlmaLinux / Rocky Linux:**
```bash
useradd -m namauser
passwd namauser
usermod -aG wheel namauser   # grup 'wheel' = akses sudo di RHEL-based
```

🟠 **Ubuntu / Debian:**
```bash
adduser namauser             # interaktif, langsung set password
usermod -aG sudo namauser    # grup 'sudo' di Debian-based
```

🔵 **Arch Linux / Manjaro:**
```bash
useradd -m -G wheel namauser
passwd namauser
# Aktifkan grup wheel di sudoers:
EDITOR=nano visudo
# Uncomment baris: %wheel ALL=(ALL:ALL) ALL
```

🟢 **openSUSE Leap / Tumbleweed:**
```bash
useradd -m namauser
passwd namauser
usermod -aG wheel namauser   # openSUSE menggunakan grup 'wheel'
```

Example:

```bash
- namauser: openclaw
```

> 💡 Ganti `namauser` dengan nama yang kamu inginkan.

---

## Step 3 — Generate SSH Key di Windows

SSH Key adalah metode login yang jauh lebih aman dibanding password biasa. Cara kerjanya seperti gembok dan kunci — public key dipasang di VPS, private key disimpan di laptop kamu.

> ✅ Langkah ini **sama untuk semua distro** karena dilakukan di Windows, bukan di VPS.

### Buka PowerShell dan jalankan:

```powershell
ssh-keygen -t ed25519 -C "label-sesukamu"

# Contoh:
ssh-keygen -t ed25519 -C "laptop-windows-gw"
```

**Penjelasan flag:**

- `-t ed25519` → algoritma enkripsi terbaik saat ini (lebih cepat & aman dari RSA)
- `-C "label"` → komentar/label saja, tidak wajib, berguna untuk identifikasi

### Jika sudah ada file key dengan nama yang sama, gunakan flag `-f`:

```powershell
ssh-keygen -t ed25519 -C "vps-openclaw" -f "$env:USERPROFILE\.ssh\id_ed25519_openclaw"
```

> ⚠️ Jangan gunakan shortcut `~` pada flag `-f` di Windows karena tidak selalu berfungsi. Gunakan `$env:USERPROFILE` sebagai gantinya.

### Tentang Passphrase

Saat generate key, kamu akan diminta mengisi passphrase. Ini adalah password untuk melindungi file private key di laptop kamu sendiri (bukan password VPS).

|                 | Tanpa Passphrase                                              | Dengan Passphrase                   |
| --------------- | ------------------------------------------------------------- | ----------------------------------- |
| **Keamanan**    | Siapapun yang punya akses ke laptopmu bisa langsung pakai key | Key terlindungi, butuh PIN tambahan |
| **Kenyamanan**  | Login langsung tanpa diminta apapun                           | Diminta passphrase tiap koneksi     |
| **Rekomendasi** | Development / belajar                                         | VPS production / data penting       |

### Hasil generate key — 2 file akan terbuat:

```
C:\Users\NamaKamu\.ssh\
├── id_ed25519_openclaw        ← private key (JANGAN dibagikan ke siapapun!)
└── id_ed25519_openclaw.pub    ← public key (boleh dikopi ke server)
```

---

## Step 4 — Copy SSH Key ke VPS

Sekarang kita perlu menaruh public key kamu ke VPS supaya server mengenali dan mengizinkan laptop kamu untuk login.

> ✅ Perintah ini **sama untuk semua distro**, dijalankan dari PowerShell Windows.

### Jalankan dari PowerShell Windows:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519_openclaw.pub | ssh namauser@ip-vps "mkdir -p /home/namauser/.ssh && cat >> /home/namauser/.ssh/authorized_keys && chown -R namauser:namauser /home/namauser/.ssh && chmod 700 /home/namauser/.ssh && chmod 600 /home/namauser/.ssh/authorized_keys"
```

Example:

```bash
- namauser: openclaw
- ip-vps: 43.134.111.176
```

> 💡 File `authorized_keys` di VPS bisa berisi banyak public key (1 per baris). Jadi kalau kamu punya banyak device (laptop, PC, HP), tinggal tambah key masing-masing di file yang sama.

### Verifikasi — coba login pakai SSH Key (sebelum matikan password auth):

```bash
ssh -i ~/.ssh/id_ed25519_openclaw namauser@ip-vps
```

> ⚠️ Pastikan kamu **BERHASIL** login dengan SSH key sebelum lanjut ke Step 5! Kalau gagal, jangan matikan password authentication dulu.

---

## Step 5 — Harden Konfigurasi SSH

Sekarang kita hardening file konfigurasi SSH di VPS untuk menutup celah-celah yang umum dieksploitasi.

> ✅ File konfigurasi SSH berada di path yang **sama untuk semua distro**: `/etc/ssh/sshd_config`

### Buka file konfigurasi SSH:

```bash
vim /etc/ssh/sshd_config
```

### Ubah/tambahkan baris berikut:

```
Port 2222                    # Ganti dari default 22 (pilih angka 1024-65535)
PermitRootLogin no           # Larang login langsung sebagai root
MaxAuthTries 5               # Maks 5 percobaan sebelum disconnect
PasswordAuthentication no    # Matikan login pakai password
```

> ⚠️ **Jangan restart SSH dulu setelah ini!** Kita perlu buka port 2222 di firewall terlebih dahulu, baru restart SSH. Kalau restart sekarang, kamu bisa terkunci keluar.

---

## Step 6 — Konfigurasi Firewall

Setiap distro menggunakan firewall yang berbeda-beda. Di sini kita akan membuka port SSH baru dan menutup port 22.

---

### 🔴 CentOS / RHEL / AlmaLinux / Rocky Linux — `firewalld`

```bash
# Install dan aktifkan firewalld
dnf install firewalld -y
systemctl start firewalld
systemctl enable firewalld

# Buka port SSH baru
firewall-cmd --permanent --add-port=2222/tcp

# Tutup port 22 (hapus service ssh)
firewall-cmd --permanent --remove-service=ssh

# Apply perubahan
firewall-cmd --reload

# Cek hasilnya
firewall-cmd --list-all
```

---

### 🟠 Ubuntu / Debian — `ufw`

UFW (Uncomplicated Firewall) adalah firewall default di Ubuntu.

```bash
# Install ufw jika belum ada
apt install ufw -y

# Izinkan port SSH baru SEBELUM mengaktifkan firewall
ufw allow 2222/tcp

# Hapus izin port 22
ufw delete allow ssh
# atau: ufw delete allow 22/tcp

# Aktifkan firewall (konfirmasi dengan 'y')
ufw enable

# Cek hasilnya
ufw status verbose
```

> ⚠️ Di Ubuntu, **aktifkan UFW setelah** menambahkan rule port 2222. Jika diaktifkan sebelumnya tanpa rule apapun, kamu langsung terkunci keluar!

---

### 🔵 Arch Linux / Manjaro — `nftables` atau `ufw`

Arch tidak menyertakan firewall secara default. Pilihan termudah adalah menggunakan UFW:

```bash
# Install ufw
pacman -S ufw

# Aktifkan service
systemctl enable --now ufw

# Izinkan port 2222
ufw allow 2222/tcp

# Hapus izin default SSH
ufw delete allow ssh

# Aktifkan firewall
ufw enable

# Cek hasilnya
ufw status verbose
```

Alternatif menggunakan `nftables` (lebih native di Arch):

```bash
pacman -S nftables
systemctl enable --now nftables

# Edit konfigurasi nftables
nano /etc/nftables.conf
```

Tambahkan rule berikut ke dalam `chain input`:
```
tcp dport 2222 accept   # izinkan port SSH baru
# tcp dport 22 accept   # hapus/comment baris ini jika ada
```

```bash
# Reload konfigurasi
nft -f /etc/nftables.conf
```

---

### 🟢 openSUSE Leap / Tumbleweed — `firewalld`

openSUSE juga menggunakan `firewalld`, sama seperti CentOS.

```bash
# Pastikan firewalld aktif
systemctl start firewalld
systemctl enable firewalld

# Buka port SSH baru
firewall-cmd --permanent --add-port=2222/tcp

# Tutup port 22
firewall-cmd --permanent --remove-service=ssh

# Apply perubahan
firewall-cmd --reload

# Cek hasilnya
firewall-cmd --list-all
```

> 💡 openSUSE juga memiliki **YaST Firewall** (GUI) yang bisa diakses via `yast2 firewall` jika kamu lebih familiar dengan antarmuka grafis.

---

### Sekarang restart SSH (semua distro):

```bash
systemctl restart sshd
```

> 💡 Di Debian/Ubuntu, nama service-nya bisa `ssh` atau `sshd`:
> ```bash
> systemctl restart ssh   # Ubuntu/Debian
> systemctl restart sshd  # CentOS, Arch, openSUSE
> ```

### WAJIB — Test koneksi dari terminal baru sebelum tutup sesi lama:

```bash
ssh -p 2222 -i ~/.ssh/id_ed25519_openclaw namauser@ip-vps
```

> ✅ Buka terminal/PowerShell **BARU** di Windows dan test konek dengan perintah di atas. Jangan tutup sesi lama sampai dipastikan berhasil!

---

## Step 7 — Install & Konfigurasi Tools Anti Brute Force

### Pilihan Tool

| Tool         | Kelebihan                                          | Cocok untuk                     |
| ------------ | -------------------------------------------------- | ------------------------------- |
| **sshguard** | Ringan, mudah dikonfigurasi, ban bertingkat        | Semua distro, setup cepat       |
| **fail2ban** | Lebih fleksibel, bisa protect banyak service       | Server yang banyak service      |

---

### sshguard

`sshguard` secara otomatis memblokir IP address yang terlalu banyak gagal login.

#### 🔴 CentOS / RHEL / AlmaLinux / Rocky Linux:
```bash
sudo dnf install sshguard -y
sudo systemctl enable --now sshguard
```

#### 🟠 Ubuntu / Debian:
```bash
sudo apt install sshguard -y
sudo systemctl enable --now sshguard
```

#### 🔵 Arch Linux / Manjaro:
```bash
sudo pacman -S sshguard
sudo systemctl enable --now sshguard
```

#### 🟢 openSUSE Leap / Tumbleweed:
```bash
sudo zypper install sshguard -y
sudo systemctl enable --now sshguard
```

#### Konfigurasi sshguard (opsional, semua distro):

```bash
sudo nano /etc/sshguard/sshguard.conf
```

Ubah nilai berikut sesuai kebutuhan (nilai default):

```
THRESHOLD=30        # ban setelah 3 percobaan gagal
BLOCK_TIME=120      # ban pertama (2 menit), naik 2x lipat tiap ban berikutnya
DETECTION_TIME=1800 # window waktu pemantauan (30 menit)
```

Sistem ban bertingkat sshguard:

```
Ban ke-1 → 2 menit
Ban ke-2 → 4 menit
Ban ke-3 → 8 menit
Ban ke-4 → 16 menit
... dst
```

Restart setelah edit konfigurasi:

```bash
sudo systemctl restart sshguard
```

#### Cek status & IP yang di-ban (semua distro):

```bash
# Cek status service
sudo systemctl status sshguard

# Monitor log real-time
sudo journalctl -u sshguard -f
```

Cek IP yang di-ban (sesuaikan dengan firewall distro kamu):

```bash
# CentOS / openSUSE (firewalld):
sudo firewall-cmd --ipset=sshguard4 --get-entries

# Ubuntu / Arch (ufw/iptables):
sudo iptables -L sshguard -n -v
```

---

### fail2ban (Alternatif)

`fail2ban` lebih powerful dari sshguard karena bisa memproteksi banyak service sekaligus (SSH, Nginx, Apache, dll).

#### 🔴 CentOS / RHEL / AlmaLinux / Rocky Linux:
```bash
sudo dnf install epel-release -y   # fail2ban ada di repo EPEL
sudo dnf install fail2ban -y
sudo systemctl enable --now fail2ban
```

#### 🟠 Ubuntu / Debian:
```bash
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
```

#### 🔵 Arch Linux / Manjaro:
```bash
sudo pacman -S fail2ban
sudo systemctl enable --now fail2ban
```

#### 🟢 openSUSE Leap / Tumbleweed:
```bash
sudo zypper install fail2ban -y
sudo systemctl enable --now fail2ban
```

#### Konfigurasi dasar fail2ban (semua distro):

Selalu buat file `.local` — jangan edit `.conf` langsung agar tidak tertimpa saat update.

```bash
sudo nano /etc/fail2ban/jail.local
```

Isi dengan konfigurasi berikut:

```ini
[DEFAULT]
bantime  = 3600    # ban 1 jam
findtime = 600     # window pemantauan 10 menit
maxretry = 5       # max 5 percobaan gagal

[sshd]
enabled = true
port    = 2222     # sesuaikan dengan port SSH kamu
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```

Restart setelah konfigurasi:

```bash
sudo systemctl restart fail2ban
```

Cek status dan IP yang di-ban:

```bash
# Status jail SSH
sudo fail2ban-client status sshd

# Lihat IP yang di-ban
sudo fail2ban-client get sshd banlist

# Unban IP tertentu (jika perlu)
sudo fail2ban-client set sshd unbanip 1.2.3.4
```

---

## Step 8 — Auto Security Updates (Opsional)

Supaya server selalu mendapat patch keamanan terbaru secara otomatis.

#### 🔴 CentOS / RHEL / AlmaLinux / Rocky Linux:
```bash
dnf install dnf-automatic -y
vim /etc/dnf/automatic.conf

# Ubah baris ini:
apply_updates = yes

# Aktifkan timer otomatis
systemctl enable --now dnf-automatic.timer
```

#### 🟠 Ubuntu / Debian:
```bash
apt install unattended-upgrades -y

# Konfigurasi
dpkg-reconfigure --priority=low unattended-upgrades
# atau edit langsung:
nano /etc/apt/apt.conf.d/50unattended-upgrades

# Aktifkan auto-update
nano /etc/apt/apt.conf.d/20auto-upgrades
```

Isi file `20auto-upgrades`:
```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

#### 🔵 Arch Linux / Manjaro:
```bash
# Arch tidak merekomendasikan auto-update penuh karena bisa breaking changes.
# Gunakan cron job untuk update periodik dengan konfirmasi:
pacman -S cronie
systemctl enable --now cronie

# Tambahkan ke crontab (weekly update dengan log):
crontab -e
# 0 3 * * 0 pacman -Syu --noconfirm >> /var/log/pacman-auto.log 2>&1
```

> ⚠️ Di Arch, auto-update penuh berisiko karena Arch adalah rolling release. Pertimbangkan untuk memonitor update secara manual atau menggunakan `paccache` untuk membersihkan cache lama.

#### 🟢 openSUSE Leap / Tumbleweed:
```bash
# Install zypper-automatic (tersedia di openSUSE)
zypper install zypper-aptdaemon -y

# Atau gunakan YaST Online Update (YOU):
yast2 online_update_configuration

# Aktifkan auto-update via systemd timer
systemctl enable --now yast2-online-update.timer
```

---

## Bonus — SSH Config untuk Koneksi Lebih Praktis

Daripada mengetik perintah panjang setiap kali konek, kamu bisa buat shortcut di file SSH config.

### Buat/edit file config di Windows:

```powershell
notepad $env:USERPROFILE\.ssh\config
```

### Isi dengan konfigurasi VPS kamu:

```
Host openclaw
    HostName ip-vps
    User namauser
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_openclaw
```

Example:

```bash
- namauser: openclaw
- ip-vps: 43.134.111.176
```

### Setelah itu cukup ketik:

```powershell
ssh openclaw
```

---

## Monitoring & Troubleshooting

### Cek percobaan login gagal:

🔴 **CentOS / RHEL / AlmaLinux / Rocky Linux:**
```bash
# Log semua aktivitas SSH
sudo tail -f /var/log/secure
sudo grep "Failed password" /var/log/secure
sudo grep "Invalid user" /var/log/secure
```

🟠 **Ubuntu / Debian:**
```bash
# Ubuntu menggunakan journald, bukan /var/log/secure
sudo journalctl -u ssh -f
sudo grep "Failed password" /var/log/auth.log
sudo grep "Invalid user" /var/log/auth.log
```

🔵 **Arch Linux:**
```bash
sudo journalctl -u sshd -f
sudo journalctl -u sshd | grep "Failed"
```

🟢 **openSUSE:**
```bash
sudo journalctl -u sshd -f
sudo grep "Failed password" /var/log/messages
```

---

## Checklist Final

Gunakan checklist ini untuk memastikan semua langkah sudah selesai:

|     | Langkah                                            | Verifikasi                                              |
| --- | -------------------------------------------------- | ------------------------------------------------------- |
| ☐   | Update sistem                                      | Tidak ada error saat update                             |
| ☐   | Buat user non-root                                 | Bisa login sebagai user baru                            |
| ☐   | Generate SSH Key di Windows                        | File `id_ed25519` dan `.pub` ada di `.ssh` folder       |
| ☐   | Copy public key ke VPS                             | Berhasil login tanpa password                           |
| ☐   | Edit `sshd_config` (port, disable root & password) | File tersimpan                                          |
| ☐   | Buka port 2222 di firewall                         | Perintah cek firewall menampilkan port 2222             |
| ☐   | Tutup port 22                                      | Perintah cek firewall tidak menampilkan port/service 22 |
| ☐   | Restart sshd                                       | Service berjalan normal                                 |
| ☐   | Test koneksi port 2222 dari terminal baru          | Berhasil masuk ke VPS                                   |
| ☐   | Install & konfigurasi sshguard / fail2ban          | Service berjalan, log aktif                             |
| ☐   | Setup SSH Config di Windows (opsional)             | `ssh vps-ku` berfungsi                                  |

---

## Quick Reference — Perintah Penting

### Koneksi & SSH

| Kebutuhan                        | Perintah                                                |
| -------------------------------- | ------------------------------------------------------- |
| Konek ke VPS                     | `ssh -p 2222 namauser@ip-vps`                           |
| Konek via SSH config             | `ssh vps-ku`                                            |
| SSH Tunnel (akses port tertutup) | `ssh -N -L 3000:localhost:3000 -p 2222 namauser@ip-vps` |
| Restart SSH                      | `sudo systemctl restart sshd`                           |

### Firewall per Distro

| Distro              | Cek rule                          | Restart firewall                        |
| ------------------- | --------------------------------- | --------------------------------------- |
| CentOS / openSUSE   | `sudo firewall-cmd --list-all`    | `sudo systemctl restart firewalld`      |
| Ubuntu / Debian     | `sudo ufw status verbose`         | `sudo ufw reload`                       |
| Arch (ufw)          | `sudo ufw status verbose`         | `sudo ufw reload`                       |
| Arch (nftables)     | `sudo nft list ruleset`           | `sudo systemctl restart nftables`       |

### Anti Brute Force

| Kebutuhan                       | sshguard                                              | fail2ban                                |
| ------------------------------- | ----------------------------------------------------- | --------------------------------------- |
| Lihat IP yang di-ban            | `sudo firewall-cmd --ipset=sshguard4 --get-entries`   | `sudo fail2ban-client get sshd banlist` |
| Monitor log                     | `sudo journalctl -u sshguard -f`                      | `sudo fail2ban-client status sshd`      |
| Restart service                 | `sudo systemctl restart sshguard`                     | `sudo systemctl restart fail2ban`       |

### Update Sistem

| Distro              | Update manual                          |
| ------------------- | -------------------------------------- |
| CentOS / RHEL       | `sudo dnf update -y`                   |
| Ubuntu / Debian     | `sudo apt update && sudo apt upgrade -y` |
| Arch Linux          | `sudo pacman -Syu`                     |
| openSUSE            | `sudo zypper update -y`                |

---

## Perbandingan Ringkas: Firewall & Package Manager

| Distro                  | Package Manager | Firewall Default | Grup Sudo  | SSH Service Name |
| ----------------------- | --------------- | ---------------- | ---------- | ---------------- |
| CentOS / RHEL / AlmaLinux / Rocky | `dnf`  | `firewalld`      | `wheel`    | `sshd`           |
| Ubuntu                  | `apt`           | `ufw`            | `sudo`     | `ssh`            |
| Debian                  | `apt`           | `nftables`/`ufw` | `sudo`     | `ssh`            |
| Arch Linux / Manjaro    | `pacman`        | Tidak ada (manual) | `wheel`  | `sshd`           |
| openSUSE Leap / Tumbleweed | `zypper`     | `firewalld`      | `wheel`    | `sshd`           |

---

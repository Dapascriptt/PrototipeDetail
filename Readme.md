# Panduan Lengkap Platform Auto-Deployment Multi-Project

---

## 1. Gambaran Besar Sistem

Platform ini terdiri dari tiga komponen utama yang saling terhubung:

**`deploy.yml`** → file konfigurasi di repositori GitHub yang mengatur otomatisasi CI/CD

**`index.php`** → file penerima di server STB yang memproses deployment

**Cloudflare Tunnel** → jembatan agar aplikasi di server lokal bisa diakses internet

Ketika mahasiswa push kode ke GitHub, ketiga komponen ini bekerja bersama secara otomatis hingga aplikasi bisa diakses publik melalui domain.

### Alur Lengkap dari Push sampai Live

```
Mahasiswa push kode ke GitHub
         ↓
[Detect & Validate]
  ✓ Generate slug nama repo
  ✓ Deteksi Laravel / PHP Native
  ✓ PHP Syntax Check — kalau error, stop di sini
         ↓
[Package & Deploy]
  ✓ Buat ZIP (exclude .env, vendor, storage)
  ✓ Kirim ZIP ke index.php via HTTP POST
         ↓
[index.php di server STB]
  ✓ Autentikasi request dari GitHub Actions
  ✓ Load/buat metadata project (port, SSH user)
  ✓ Provisioning database MySQL otomatis
  ✓ Buat SSH user + .bash_profile + sudoers
  ✓ Ekstrak ZIP → rsync ke direktori project
  ✓ Auto-generate .env dengan config DB
  ✓ Atur permission 3-layer
  ✓ Tulis Nginx config → validasi → reload (+ buat symlink sites-enabled)
  ✓ Tulis Systemd service → enable
  ✓ Start/restart service aplikasi
  ✓ Update Cloudflare DNS & Tunnel via API
  ✓ Kirim response ADPL:xxx ke GitHub
         ↓
[Notification]
  ✓ Tampilkan Job Summary berisi:
    - Domain aplikasi
    - Kredensial SSH
    - Status deployment
    - Panduan langkah selanjutnya
         ↓
Mahasiswa SSH ke server menggunakan
kredensial dari Job Summary
         ↓
composer install → migrate → restart-app
         ↓
Aplikasi LIVE dan bisa diakses publik
melalui domain Cloudflare Tunnel
```

---

## 2. Cloudflare Tunnel

### Apa itu Cloudflare Tunnel?

Cloudflare Tunnel adalah layanan yang menghubungkan server lokal (STB di jaringan rumah/kampus) ke internet secara aman **tanpa perlu IP publik dan tanpa membuka port router**. Server lokal "memanggil keluar" ke Cloudflare, bukan sebaliknya, sehingga server tidak terekspos langsung ke internet.

### Cara Membuat Tunnel

```bash
# Buat tunnel baru, output berupa ID tunnel
cloudflared tunnel create nameserver

# Edit file konfigurasi tunnel
sudo nano /root/.cloudflared/config.yml
```

### Isi `config.yml`

File ini berisi kredensial untuk terhubung ke tunnel dan konfigurasi routing domain. Isinya kurang lebih:

```yaml
tunnel: <ID-tunnel-kamu>
credentials-file: /root/.cloudflared/<ID-tunnel>.json

ingress:
  - hostname: project-a.akhzafachrozy.my.id
    service: http://localhost:8001
  - hostname: project-b.akhzafachrozy.my.id
    service: http://localhost:8002
  - service: http_status:404
```

**Ingress** adalah daftar aturan routing yang memberitahu Cloudflare Tunnel ke mana harus meneruskan request yang masuk. Setiap baris berisi `hostname` (domain yang dituju pengguna) dan `service` (port lokal tempat aplikasi berjalan di server). Baris terakhir tanpa `hostname` adalah fallback — request yang tidak cocok dikembalikan 404.

Pada platform ini, ingress ditambahkan otomatis oleh `index.php` via Cloudflare API setiap kali ada project baru dideploy, sehingga tidak perlu edit manual.

### Cara Akses dan Edit Config

```bash
cd /root/.cloudflared
cat config.yml    # lihat isi
nano config.yml   # edit isi
```

### Hubungan dengan index.php

Ketika deploy pertama, `index.php` otomatis memanggil Cloudflare API untuk:
1. Membuat DNS CNAME record baru → `namaproject.akhzafachrozy.my.id`
2. Menambahkan ingress rule baru di tunnel → arahkan domain ke `localhost:port`

Sehingga setelah deployment selesai, domain langsung aktif tanpa konfigurasi manual.

---

## 3. Deploy.yml — Workflow GitHub Actions

### Konsep Dasar

`deploy.yml` mengimplementasikan konsep CI/CD. Penelitian terdahulu umumnya menyebut tahapan ini sebagai *build, testing, dan deploy*. Pada platform ini, konsep tersebut diadaptasi menjadi tiga tahapan yang lebih spesifik sesuai kebutuhan, yaitu **Detect & Validate**, **Package & Deploy**, dan **Notification**.

### Trigger — Kapan Workflow Jalan?

```yaml
on:
  push:
    branches:
      - main
      - master
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

Workflow otomatis jalan saat ada push ke branch `main` atau `master`. File dokumentasi seperti `.md` diabaikan agar deployment tidak terpicu hanya karena edit README. Ada juga trigger manual lewat UI GitHub (`workflow_dispatch`) dengan pilihan mode `full` atau `update`.

### Concurrency

```yaml
concurrency:
  group: deploy-${{ github.ref_name }}
  cancel-in-progress: false
```

Hanya satu deployment yang boleh berjalan pada satu waktu untuk branch yang sama. Deployment baru menunggu sampai yang sedang berjalan selesai, tidak dibatalkan.

---

### Job 1: Detect & Validate

**Generate Repository Slug**

```bash
SLUG=$(echo "${{ github.event.repository.name }}" \
  | tr '[:upper:]' '[:lower:]' \
  | sed 's/[^a-z0-9-]/-/g' \
  | sed 's/--*/-/g' \
  | sed 's/^-//;s/-$//')
```

Mengubah nama repo menjadi format aman untuk subdomain dan direktori. `Proyek_TA_Daffa` → `proyek-ta-daffa`. Prosesnya: huruf kapital → kecil, karakter non-alfanumerik → tanda hubung, tanda hubung ganda → satu, hapus tanda hubung di awal/akhir.

**Detect Framework**

```bash
if [[ -f "artisan" && -f "composer.json" ]]; then
  echo "app_type=laravel"
else
  echo "app_type=php-native"
fi
```

Mendeteksi apakah project Laravel atau PHP Native berdasarkan keberadaan file `artisan` dan `composer.json`. Hasil deteksi disimpan sebagai output yang bisa digunakan job berikutnya.

**Set Deploy Mode**

```bash
if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
  echo "deploy_mode=${{ inputs.mode }}"
else
  echo "deploy_mode=full"
fi
```

Jika workflow dipicu manual lewat UI GitHub, gunakan mode yang dipilih pengguna. Jika dipicu otomatis oleh push, selalu gunakan mode `full`.

**PHP Syntax Check**

```bash
find . -name "*.php" ! -path "./vendor/*" -exec php -l {} \;
```

Memeriksa semua file PHP apakah ada syntax error. Kalau ada error, workflow berhenti di sini dan tidak lanjut ke deployment. Inilah inti dari **Continuous Integration** — kode yang salah tidak boleh sampai ke server.

---

### Job 2: Package & Deploy

**Create Deployment Package**

```bash
find . \
  -not -path "./.git/*" \
  -not -path "./.github/*" \
  -not -path "./node_modules/*" \
  -not -path "./vendor/*" \
  -not -name ".env" \
  -not -name "deploy.zip" \
  -not -name "*.log" \
  | zip deploy.zip -@ -q
```

Mengemas project ke ZIP dengan mengecualikan file yang tidak perlu dikirim ke server. `vendor/` dan `node_modules/` tidak dikirim karena bisa diinstall ulang di server. `.env` tidak dikirim karena berisi konfigurasi yang berbeda antara lokal dan server.

**Send Deployment Package**

```bash
RESPONSE=$(curl -s -X POST \
  -H "User-Agent: GitHub-Actions-AutoDeploy" \
  -F "key=autodeploy-classroom" \
  -F "repo=$REPO_SLUG" \
  -F "mode=$DEPLOY_MODE" \
  -F "file=@deploy.zip" \
  "$DEPLOY_URL")
```

Mengirim ZIP ke `index.php` di server STB menggunakan HTTP POST. Parameter `key` digunakan untuk autentikasi, `repo` untuk nama project, `mode` untuk menentukan full atau update deployment.

**Parse & Verifikasi Response**

```bash
if echo "$RESPONSE" | grep -q "AUTODEPLOY_START"; then
  echo "status=success"
else
  echo "status=failed"
  exit 1
fi
```

Mengecek apakah server berhasil memproses. Jika respons mengandung `AUTODEPLOY_START`, berarti sukses. Kemudian informasi seperti domain, port, SSH user dan password di-parse dari respons untuk ditampilkan di summary.

**Parse Deployment Results**

```bash
echo "P_DOMAIN=$(echo "$RAW_RESPONSE" | grep '^ADPL:domain=' | cut -d'=' -f2-)"
echo "P_SSH_USER=$(echo "$RAW_RESPONSE" | grep '^ADPL:ssh_user=' | cut -d'=' -f2-)"
echo "P_SSH_PASS=$(echo "$RAW_RESPONSE" | grep '^ADPL:ssh_pass=' | cut -d'=' -f2-)"
```

Mengambil informasi spesifik dari respons server dengan format `ADPL:key=value`. Diambil domain, port, SSH user, SSH password, SSH command, nama service, status web, dan status server.

---

### Job 3: Notification

```yaml
notify:
  needs: [detect, deploy]
  if: always()
```

Selalu berjalan meskipun job sebelumnya gagal (`if: always()`). Menampilkan status akhir deployment termasuk domain, kredensial SSH, dan panduan langkah selanjutnya di Job Summary GitHub Actions.

---

## 4. index.php — Deployment Receiver

### Fungsi Utama

`index.php` adalah komponen sisi server yang menerima ZIP dari GitHub Actions dan secara otomatis membangun seluruh infrastruktur untuk setiap project.

---

### Autentikasi

```php
if ($key === 'autodeploy-classroom') {
    $isGithub = str_contains($ua, 'GitHub') || str_contains($ua, 'actions');
    if ($validRepo && $isGithub) {
        $allowed = true;
    }
}
if (!$allowed) {
    abort(403, 'Unauthorized');
}
```

Ada dua jalur autentikasi. Jalur pertama menggunakan API key rahasia dengan `hash_equals()` yang aman dari timing attack. Jalur kedua menggunakan key khusus `autodeploy-classroom` yang hanya diizinkan jika request berasal dari GitHub Actions (dicek dari User-Agent header). Request dari luar GitHub Actions akan ditolak 403.

---

### Metadata Persistence

```php
$metaFile = "$metaDir/$repo.json";
$meta = loadMeta($metaFile);
```

Setiap project punya file JSON yang menyimpan konfigurasi persistennya: port, SSH user, SSH password, nama database, dan waktu deploy terakhir. Ini memastikan pada deploy berikutnya semua konfigurasi tetap sama dan konsisten.

---

### Port Management

```php
do {
    $port = rand($portMin, $portMax); // 8000-8999
} while (in_array($port, $usedPorts, true));
```

Setiap project mendapat port unik yang tidak bentrok dengan project lain. Port ini disimpan di metadata sehingga tidak berubah setiap deploy. Inilah salah satu mekanisme isolasi di konsep multi-project.

---

### Database Auto Provisioning

```php
$dbSql = "
CREATE DATABASE IF NOT EXISTS `$dbName`
CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER IF NOT EXISTS '$dbUser'@'localhost' IDENTIFIED BY '$dbPass';
GRANT ALL PRIVILEGES ON `$dbName`.* TO '$dbUser'@'localhost';
FLUSH PRIVILEGES;
";
```

Database MySQL dibuat otomatis dengan nama yang sama dengan repo (tanda hubung diganti underscore). User database juga dibuat otomatis dengan akses penuh ke database tersebut. Mahasiswa tidak perlu buat database manual.

---

### SSH User Setup

```php
if (!posix_getpwnam($sshUser)) {
    shell_exec("useradd -M -d $target -s /bin/bash $sshUser");
}
shell_exec("usermod -aG www $sshUser");

$sshPass = $repo . '<3' . $digits; // contoh: mhika-frozen<3042
shell_exec("echo '$sshUser:$sshPass' | chpasswd");
```

User Linux dibuat dengan nama sama dengan repo, home directory diarahkan ke folder project, ditambahkan ke group `www` agar bisa baca-tulis file web server. Password di-generate otomatis dan ditampilkan di Job Summary.

`.bash_profile` yang dibuat otomatis berisi alias berguna seperti `php`, `artisan`, `composer`, `restart-app`, `fix-perm`, dan `log-app`. Saat mahasiswa login SSH, langsung masuk ke direktori project dan muncul banner info berisi domain dan perintah yang tersedia.

---

### Mekanisme Rsync dengan Exclude

Mekanisme *rsync* dengan *exclude* adalah proses sinkronisasi file aplikasi dari paket *deployment* ke direktori server secara selektif, di mana file-file tertentu dikecualikan agar konfigurasi dan data yang telah ada pada server tidak tertimpa oleh proses *deployment* berikutnya.

```php
$rsyncExcludes = [
    '.env',           // konfigurasi sensitif server
    'storage/',       // file upload user
    'public/storage',
    'vendor/',        // dependensi PHP
    'node_modules/',  // dependensi JS
    'database/database.sqlite',
    '.git/',
    '.bash_profile',  // konfigurasi SSH user
    '.bashrc',
];

$rsyncExclude = implode(' ', array_map(
    fn($e) => '--exclude=' . escapeshellarg($e),
    $rsyncExcludes
));

$rsyncCmd = "rsync -rlptD --checksum $rsyncExclude"
          . " --include='.*'"
          . " " . escapeshellarg(rtrim($src, '/') . '/')
          . " " . escapeshellarg(rtrim($target, '/') . '/')
          . " 2>&1";

$rsyncOut = shell_exec($rsyncCmd);
logMsg($repo, "rsync selesai. Output:\n" . trim($rsyncOut ?: '(tidak ada output)'));
```

Opsi `--checksum` memastikan hanya file yang benar-benar berubah isinya yang disalin, bukan berdasarkan waktu modifikasi — lebih akurat dan efisien.

---

### Deteksi Framework & Auto .env

```php
$composerJsonPath = "$target/composer.json";
$isLaravel = false;

if (file_exists("$target/artisan") && file_exists($composerJsonPath)) {
    $composerContent = file_get_contents($composerJsonPath);
    if (str_contains($composerContent, 'laravel/framework')) {
        $isLaravel = true;
    }
}

$appType = $isLaravel ? 'laravel' : 'php-native';
```

Deteksi dua lapis: cek file `artisan` dulu, konfirmasi dengan baca isi `composer.json`. Jika Laravel, `.env` dibuat otomatis dari `.env.example` dengan konfigurasi database yang sudah diprovisioning. Handle juga kasus baris yang dikomentari dengan `#`.

---

### Permission 3-Layer

#### Dasar: Cara Baca Angka Permission Linux

Setiap angka permission terdiri dari 3 digit yang mewakili hak akses untuk tiga pihak:

```
Digit 1 → Owner (pemilik file)
Digit 2 → Group (kelompok)
Digit 3 → Others (orang lain)
```

Setiap digit merupakan penjumlahan dari nilai berikut:

```
4 = Read    (baca)
2 = Write   (tulis)
1 = Execute (jalankan)
0 = Tidak ada akses

7 = 4+2+1 = Read + Write + Execute
6 = 4+2   = Read + Write
5 = 4+1   = Read + Execute
4 = 4     = Read saja
```

#### Implementasi di Platform Ini

**Layer 1 — Direktori Root Project: `sshUser:sshUser 755`**
```
Owner  (sshUser) → 7 = Read + Write + Execute
Group  (sshUser) → 5 = Read + Execute
Others           → 5 = Read + Execute
```
Direktori root project dimiliki oleh SSH user. Semua orang bisa masuk ke direktori ini, tapi hanya SSH user yang bisa mengubah isinya.

**Layer 2 — Konten Project: `sshUser:www 775/664`**

Folder (775):
```
Owner (sshUser) → 7 = Read + Write + Execute
Group (www)     → 7 = Read + Write + Execute
Others          → 5 = Read + Execute
```

File (664):
```
Owner (sshUser) → 6 = Read + Write
Group (www)     → 6 = Read + Write
Others          → 4 = Read saja
```

**Layer 3 — Storage & Cache: `sshUser:www 775`**
```
Owner (sshUser) → 7 = Read + Write + Execute
Group (www)     → 7 = Read + Write + Execute
Others          → 5 = Read + Execute
```
Direktori `storage/` dan `bootstrap/cache/` butuh permission tulis oleh web server karena Laravel menyimpan file log, cache view, dan session di sini. Tanpa permission ini Laravel akan error.

**Khusus .env: `sshUser:www 640`**
```
Owner (sshUser) → 6 = Read + Write
Group (www)     → 4 = Read saja
Others          → 0 = Tidak ada akses sama sekali
```
File `.env` berisi password database dan APP_KEY yang sangat sensitif. Web server hanya perlu membacanya, tidak perlu menulis. Orang lain tidak boleh mengakses sama sekali.

#### Ringkasan Permission

| Target | Permission | Artinya |
|---|---|---|
| Direktori root | 755 | SSH user bisa ubah, semua bisa masuk |
| Folder project | 775 | SSH user & web server bisa ubah |
| File project | 664 | SSH user & web server bisa tulis, others hanya baca |
| storage/ & cache/ | 775 | Web server harus bisa tulis untuk Laravel |
| .env | 640 | Hanya SSH user & web server yang bisa baca, others diblokir total |

---

### Provisioning Nginx dan Systemd per Project

*Provisioning* Nginx dan *Systemd* per *project* adalah proses pembangunan infrastruktur server secara otomatis untuk setiap *project* yang didaftarkan pada platform, meliputi pembuatan konfigurasi Nginx sebagai pengatur lalu lintas akses menuju aplikasi yang sesuai dan pembuatan file *systemd service* sebagai pengelola *lifecycle* aplikasi, sehingga setiap *project* memiliki konfigurasi layanan yang terpisah dan independen dalam satu server yang sama.

#### Nginx + Symlink sites-enabled

```php
$nginxConf     = "$nginxAvail/$repo.conf";
$nginxLink     = "$nginxEnabled/$repo.conf";
$servicePath   = "$systemd/autodeploy-$repo.service";
$serviceName   = "autodeploy-$repo.service";
$isFirstDeploy = !file_exists($nginxConf) || empty($meta);

file_put_contents($nginxConf, $nginxCommon . $nginxBody);
logMsg($repo, "Nginx config ditulis ($appType): $nginxConf");

shell_exec("ln -sf " . escapeshellarg($nginxConf) . " " . escapeshellarg($nginxLink) . " 2>&1");
logMsg($repo, "Nginx symlink: $nginxLink");

$nginxTest = shell_exec('nginx -t 2>&1');
if (str_contains($nginxTest, 'successful')) {
    shell_exec('systemctl reload nginx 2>&1');
    logMsg($repo, "Nginx reload OK -> $sub -> :$port");
}
```

**Kenapa pakai symlink?** Nginx membaca konfigurasi dari dua folder:
- `sites-available` → tempat menyimpan semua config (semua project ada di sini)
- `sites-enabled` → config yang aktif digunakan Nginx

Symlink berfungsi sebagai "jembatan" — shortcut dari `sites-enabled` yang mengarah ke file di `sites-available`. Sebelum direload, konfigurasi divalidasi dulu dengan `nginx -t`. Jika ada error, file config dihapus dan deployment dibatalkan.

#### Systemd Service

```php
file_put_contents($servicePath, $svcContent);
shell_exec('systemctl daemon-reload');
shell_exec("systemctl enable " . escapeshellarg($serviceName) . " 2>&1");
logMsg($repo, "Systemd service ditulis & dienable: $serviceName");
```

**Systemd** adalah sistem manajemen layanan pada Linux yang bertugas menjalankan, menghentikan, dan memantau proses aplikasi yang berjalan di server. Pada platform ini, *systemd* digunakan untuk memastikan setiap aplikasi mahasiswa yang telah di-*deploy* selalu berjalan secara otomatis, termasuk menghidupkan kembali aplikasi apabila berhenti karena kesalahan atau server *restart*:

```php
$svcContent .= "Restart=on-failure\n";      // hidupkan lagi kalau crash
$svcContent .= "RestartSec=5s\n";           // tunggu 5 detik sebelum restart
$svcContent .= "ProtectSystem=strict\n";    // keamanan: batasi akses sistem
$svcContent .= "NoNewPrivileges=true\n";    // tidak bisa eskalasi privilege
```

Untuk Laravel dijalankan dengan `php artisan serve`, untuk PHP Native dengan `php -S`.

**Kenapa perlu Systemd?** Tanpa systemd, kalau server restart, aplikasi mati dan tidak ada yang menghidupkan kembali. Dengan systemd service, setiap project punya "penjaga" sendiri yang menjalankan aplikasi otomatis saat booting dan menghidupkan kembali jika crash.

> **Analogi:** Symlink itu seperti daftar menu restoran yang aktif. Systemd itu seperti koki yang ditugaskan khusus untuk setiap menu — kalau koki jatuh sakit, langsung diganti otomatis.

---

### Cloudflare API Integration

```php
// Buat DNS CNAME
cfRequest('POST', ".../dns_records", $cfHeaders, json_encode([
    'type'    => 'CNAME',
    'name'    => $repo,
    'content' => $tunnelCname,
    'proxied' => true,
]));

// Update tunnel ingress
cfRequest('PUT', ".../cfd_tunnel/$tunnelId/configurations", $cfHeaders,
    json_encode(['config' => ['ingress' => $ingressFiltered]])
);
```

Pada deploy pertama, DNS dan tunnel ingress dikonfigurasi otomatis via API. Pada deploy berikutnya, sistem verifikasi apakah ingress sudah ada — kalau sudah ada tidak diubah, kalau hilang diperbaiki otomatis.

---

### Output Response

Response yang dikirim ke GitHub Actions menggunakan format terstruktur:

```
AUTODEPLOY_START
ADPL:domain=https://namaproject.akhzafachrozy.my.id
ADPL:ssh_user=namaproject
ADPL:ssh_pass=namaproject<3042
ADPL:port=8042
ADPL:srv_status=waiting-setup
AUTODEPLOY_END
```

Format `ADPL:key=value` ini yang di-parse oleh step `Parse Deployment Results` di `deploy.yml` untuk ditampilkan sebagai tabel di Job Summary.

---

## 5. Keterkaitan dengan Penelitian Terdahulu

Penelitian terdahulu umumnya menyebut tahapan CI/CD sebagai *build, testing, dan deploy*. Faqih dkk. (2024) menunjukkan bahwa implementasi CI/CD di GitHub Actions pada kenyataannya tidak terbatas pada tahapan baku tersebut — bahkan ditemukan tools yang tidak masuk taksonomi standar namun digunakan secara nyata di dunia industri.

Thota (2020) juga menyatakan bahwa workflow GitHub Actions dapat dikustomisasi sesuai kebutuhan spesifik proyek (*"Programming workflows from GitHub Actions enable developers to customize the execution processes according to individual project specifications"*).

Hal ini mendukung pendekatan penelitian ini yang mengadaptasi tahapan tersebut menjadi **Detect & Validate**, **Package & Deploy**, dan **Notification** sesuai kebutuhan spesifik platform auto-deployment multi-project.

---

## 6. Tips Praktis: Perbaikan kalau db:seed Gagal

Kalau seeder gagal karena butuh package seperti Faker yang tidak tersedia di production, jalankan perintah ini secara berurutan:

```bash
composer install --optimize-autoloader
php artisan db:seed
composer install --no-dev --optimize-autoloader
restart-app
```

**Kenapa dua kali `composer install`?**

1. `composer install --optimize-autoloader` → install **semua** dependensi termasuk `require-dev` seperti Faker agar seeder bisa jalan
2. `php artisan db:seed` → jalankan seeder yang membutuhkan Faker
3. `composer install --no-dev --optimize-autoloader` → install ulang **tanpa** dev dependensi agar server production bersih
4. `restart-app` → restart service agar perubahan aktif

---

## 7. Referensi Jurnal untuk Sidang

**Faqih, A.R., dkk. (2024)**
*Empirical Analysis of CI/CD Tools Usage in GitHub Actions Workflows*
Journal of Informatics and Web Engineering, Vol. 3 No. 2

Relevan di: halaman 258 (tabel tools "Etc" yang tidak masuk taksonomi standar) dan halaman 259 (RQ3: tools GitHub Actions sangat adaptif terhadap kebutuhan proyek).

---

**Thota, R.C. (2020)**
*CI/CD Pipeline Optimization: Enhancing Deployment Speed and Reliability with AI and Github Actions*
IJIRMPS, Vol. 8 Issue 2

Relevan di: halaman 3 dan 5 (*"Workflows are defined through YAML files in GitHub Actions to deliver customization and flexibility so teams can customize their CI/CD pipelines for their unique requirements"*).

---

**Jawaban saat penguji tanya kenapa tahapan berbeda dari build-test-deploy:**

> *"Penelitian terdahulu seperti Faqih dkk. (2024) dan Thota (2020) menunjukkan bahwa workflow GitHub Actions sangat adaptif dan dapat dikustomisasi sesuai kebutuhan spesifik proyek. Pada penelitian ini, tahapan build-test-deploy diadaptasi menjadi Detect & Validate, Package & Deploy, dan Notification yang lebih sesuai dengan kebutuhan platform auto-deployment multi-project. Buktinya mahasiswa cukup push ke GitHub, sistem yang otomatis menangani semuanya."*

---

## 8. Perintah Terminal Linux yang Sering Dipakai

### Apa itu Service?

Service adalah program yang berjalan di background server secara terus-menerus. Di platform ini, setiap project punya service sendiri dengan nama `autodeploy-[repo].service`, contoh: `autodeploy-web-ranking.service`. Service inilah yang menjalankan aplikasi Laravel kamu di server.

---

### systemctl — Kelola Service

`systemctl` adalah perintah untuk mengelola service di Linux menggunakan Systemd.

```bash
# Cek status service — apakah aktif, mati, atau error
systemctl status autodeploy-web-ranking.service

# Start service (jalankan pertama kali)
systemctl start autodeploy-web-ranking.service

# Stop service (matikan)
systemctl stop autodeploy-web-ranking.service

# Restart service (matikan lalu jalankan ulang)
systemctl restart autodeploy-web-ranking.service

# Reload semua service setelah ada perubahan file .service
systemctl daemon-reload

# Cek apakah service aktif (output: active / inactive)
systemctl is-active autodeploy-web-ranking.service
```

> Di platform ini, kamu bisa pakai alias `restart-app` dan `status-app` yang sudah diset otomatis di `.bash_profile` — tidak perlu ketik nama service panjang.

---

### journalctl — Lihat Log Service

`journalctl` digunakan untuk membaca log dari systemd service, berguna saat aplikasi tidak mau jalan dan perlu tahu penyebabnya.

```bash
# Lihat 50 baris log terakhir dari service
journalctl -u autodeploy-web-ranking.service -n 50 --no-pager

# Lihat log secara live (update otomatis)
journalctl -u autodeploy-web-ranking.service -f

# Lihat log sejak server terakhir reboot
journalctl -u autodeploy-web-ranking.service -b
```

> Alias `log-app` di `.bash_profile` sudah menjalankan perintah ini secara otomatis.

---

### cat — Tampilkan Isi File

`cat` digunakan untuk menampilkan isi file langsung di terminal tanpa membuka editor.

```bash
# Lihat isi file .env
cat .env

# Lihat isi file konfigurasi Nginx project
cat /etc/nginx/sites-available/web-ranking.conf

# Lihat isi metadata project (port, SSH user, dll)
cat /var/lib/autodeploy/web-ranking.json

# Lihat log deployment
cat /var/log/autodeploy/web-ranking.log
```

---

### nano — Edit File di Terminal

`nano` adalah editor teks sederhana di terminal. Digunakan untuk mengedit file konfigurasi langsung di server.

```bash
# Edit file .env
nano .env

# Edit config tunnel Cloudflare
nano /root/.cloudflared/config.yml
```

**Shortcut penting saat di dalam nano:**
```
Ctrl + S  → Simpan file
Ctrl + X  → Keluar dari nano
Ctrl + W  → Cari teks
Ctrl + K  → Hapus satu baris
```

---

### ls — Lihat Isi Direktori

```bash
# Lihat isi folder saat ini
ls

# Lihat detail (ukuran, permission, tanggal)
ls -la

# Lihat isi folder tertentu
ls -la /www/wwwroot/hosting/web-ranking/

# Lihat isi folder storage
ls -la storage/logs/
```

---

### cd — Pindah Folder

```bash
# Masuk ke folder project
cd /www/wwwroot/hosting/web-ranking/

# Kembali ke folder sebelumnya
cd ..

# Kembali ke home directory
cd ~
```

---

### pwd — Cek Posisi Folder Saat Ini

```bash
pwd
# Output contoh: /www/wwwroot/hosting/web-ranking
```

---

### curl — Test Akses HTTP

`curl` digunakan untuk menguji apakah domain/aplikasi bisa diakses dan mengembalikan respons yang benar.

```bash
# Cek apakah domain bisa diakses (lihat HTTP status)
curl -I https://web-ranking.akhzafachrozy.my.id

# Cek respons localhost (sebelum domain aktif)
curl -I http://127.0.0.1:8042

# Cek dengan output detail waktu respons
curl -o /dev/null -s -w "%{http_code} - %{time_total}s\n" https://web-ranking.akhzafachrozy.my.id
```

**Kode HTTP yang perlu diketahui:**
```
200 → OK, aplikasi berjalan normal
302 → Redirect (biasanya ke /login, masih normal)
404 → Halaman tidak ditemukan
500 → Error di aplikasi (cek log)
502 → Service mati atau belum jalan
503 → Setup in progress (halaman waiting Laravel)
```

---

### free — Cek Penggunaan RAM

```bash
# Lihat penggunaan RAM dan swap dalam format mudah dibaca
free -h
```

Output contoh:
```
              total   used    free    shared  buff/cache  available
Mem:          1.8Gi   1.0Gi   497Mi   12Mi    312Mi       756Mi
Swap:          15Gi   1.7Gi   13Gi
```

---

### df — Cek Penggunaan Storage

```bash
# Lihat penggunaan disk semua partisi
df -h

# Lihat penggunaan disk folder tertentu saja
du -sh /www/wwwroot/hosting/web-ranking/
```

---

### tail — Lihat Log secara Live

```bash
# Lihat 50 baris terakhir log aplikasi
tail -n 50 /var/log/autodeploy/web-ranking-app.log

# Lihat log secara live (update otomatis saat ada log baru)
tail -f /var/log/autodeploy/web-ranking-app.log

# Alias yang sudah tersedia:
log-app   # sama dengan tail -f log aplikasi kamu
```

---

### php artisan — Perintah Laravel

```bash
# Generate APP_KEY (wajib setelah deploy pertama)
php artisan key:generate

# Jalankan migrasi database
php artisan migrate --force

# Jalankan seeder
php artisan db:seed

# Buat symlink storage
php artisan storage:link

# Clear cache
php artisan cache:clear
php artisan config:clear
php artisan view:clear
php artisan route:clear

# Cek status migrasi
php artisan migrate:status
```

---

### composer — Kelola Dependensi PHP

```bash
# Install semua dependensi (termasuk dev, untuk seeding)
composer install --optimize-autoloader

# Install tanpa dev dependensi (untuk production)
composer install --no-dev --optimize-autoloader

# Update dependensi
composer update
```

---

### Alias Siap Pakai di Platform Ini

Setelah login SSH, alias berikut sudah tersedia tanpa perlu ketik perintah panjang:

```bash
restart-app    # sudo systemctl restart autodeploy-[repo].service
stop-app       # sudo systemctl stop autodeploy-[repo].service
status-app     # sudo systemctl status autodeploy-[repo].service
log-app        # tail -f /var/log/autodeploy/[repo]-app.log
fix-perm       # sudo fix-perm-[repo] (perbaiki permission semua file)
php            # /www/server/php/83/bin/php
artisan        # php artisan
composer       # /usr/local/bin/composer
```

---

### Urutan Perintah Setelah Deploy Pertama

Ini urutan yang benar setelah deploy pertama berhasil dan kamu sudah SSH ke server:

```bash
# 1. Install dependensi
composer install --no-dev --optimize-autoloader

# 2. Generate APP_KEY
php artisan key:generate

# 3. Jalankan migrasi
php artisan migrate --force

# 4. Buat symlink storage (agar file upload bisa diakses publik)
php artisan storage:link

# 5. Fix permission
fix-perm

# 6. Aktifkan aplikasi
restart-app

# 7. Verifikasi
status-app
curl -I http://127.0.0.1:[port]
```

---

### Urutan Perintah Kalau db:seed Gagal

```bash
# Install dulu termasuk dev dependency (Faker perlu ini)
composer install --optimize-autoloader

# Jalankan seeder
php artisan db:seed

# Install ulang tanpa dev (bersihkan production)
composer install --no-dev --optimize-autoloader

# Restart service
restart-app
```

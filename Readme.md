# Panduan Lengkap Platform Auto-Deployment Multi-Project

---

## 1. Gambaran Besar Sistem

Platform ini terdiri dari tiga komponen utama yang saling terhubung:

**`deploy.yml`** → file konfigurasi di repositori GitHub yang mengatur otomatisasi CI/CD

**`index.php`** → file penerima di server STB yang memproses deployment

**Cloudflare Tunnel** → jembatan agar aplikasi di server lokal bisa diakses internet

Ketika mahasiswa push kode ke GitHub, ketiga komponen ini bekerja bersama secara otomatis hingga aplikasi bisa diakses publik melalui domain.

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

Setiap project yang dideploy akan otomatis ditambahkan ke ingress ini oleh `index.php` melalui Cloudflare API — jadi tidak perlu edit manual setiap kali ada project baru &
Ingress adalah daftar aturan routing yang memberitahu Cloudflare Tunnel ke mana harus meneruskan request yang masuk. Setiap baris berisi hostname (domain yang dituju pengguna) dan service (port lokal tempat aplikasi berjalan di server). Baris terakhir tanpa hostname adalah fallback — request yang tidak cocok dikembalikan 404. Pada platform ini, ingress ditambahkan otomatis oleh index.php via Cloudflare API setiap kali ada project baru dideploy, sehingga tidak perlu edit manual.

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
  | sed 's/[^a-z0-9-]/-/g')
```
Mengubah nama repo menjadi format aman untuk subdomain dan direktori. `Proyek_TA_Daffa` → `proyek-ta-daffa`.

**Detect Framework**
```bash
if [[ -f "artisan" && -f "composer.json" ]]; then
  echo "app_type=laravel"
else
  echo "app_type=php-native"
fi
```
Mendeteksi apakah project Laravel atau PHP Native berdasarkan keberadaan file `artisan` dan `composer.json`.

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
  -not -path "./node_modules/*" \
  -not -path "./vendor/*" \
  -not -name ".env" \
  | zip deploy.zip -@ -q
```
Mengemas project ke ZIP dengan mengecualikan file yang tidak perlu dikirim ke server. `vendor/` dan `node_modules/` tidak dikirim karena bisa diinstall ulang di server. `.env` tidak dikirim karena berisi konfigurasi yang berbeda antara lokal dan server.

**Send Deployment Package**
```bash
RESPONSE=$(curl -s -X POST \
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
  exit 1
fi
```
Mengecek apakah server berhasil memproses. Jika respons mengandung `AUTODEPLOY_START`, berarti sukses. Kemudian informasi seperti domain, port, SSH user dan password di-parse dari respons untuk ditampilkan di summary.

---

### Job 3: Notification
```yaml
notify:
  needs: [detect, deploy]
  if: always()
```
Selalu berjalan meskipun job sebelumnya gagal. Menampilkan status akhir deployment termasuk domain, kredensial SSH, dan panduan langkah selanjutnya di Job Summary GitHub Actions.

---

## 4. index.php — Deployment Receiver

### Fungsi Utama
`index.php` adalah komponen sisi server yang menerima ZIP dari GitHub Actions dan secara otomatis membangun seluruh infrastruktur untuk setiap project.

### Autentikasi
```php
if ($key === 'autodeploy-classroom') {
    $isGithub = str_contains($ua, 'GitHub') || str_contains($ua, 'actions');
    if ($validRepo && $isGithub) {
        $allowed = true;
    }
}
```
Ada dua jalur autentikasi. Jalur pertama menggunakan API key rahasia. Jalur kedua menggunakan key khusus `autodeploy-classroom` yang hanya diizinkan jika request berasal dari GitHub Actions (dicek dari User-Agent header). Request dari luar GitHub Actions akan ditolak 403.

### Metadata Persistence
```php
$metaFile = "$metaDir/$repo.json";
$meta = loadMeta($metaFile);
```
Setiap project punya file JSON yang menyimpan konfigurasi persistennya: port, SSH user, SSH password, nama database. Ini memastikan pada deploy berikutnya semua konfigurasi tetap sama dan konsisten.

### Port Management
```php
do {
    $port = rand($portMin, $portMax); // 8000-8999
} while (in_array($port, $usedPorts, true));
```
Setiap project mendapat port unik yang tidak bentrok dengan project lain. Port ini disimpan di metadata sehingga tidak berubah setiap deploy. Inilah salah satu mekanisme isolasi di konsep multi-project.

### Database Auto Provisioning
```php
$dbSql = "
CREATE DATABASE IF NOT EXISTS `$dbName`;
CREATE USER IF NOT EXISTS '$dbUser'@'localhost' IDENTIFIED BY '$dbPass';
GRANT ALL PRIVILEGES ON `$dbName`.* TO '$dbUser'@'localhost';
";
```
Database MySQL dibuat otomatis dengan nama yang sama dengan repo. Mahasiswa tidak perlu buat database manual — saat deploy pertama semuanya sudah siap.

### SSH User Setup
```php
shell_exec("useradd -M -d $target -s /bin/bash $sshUser");
shell_exec("usermod -aG www $sshUser");
$sshPass = $repo . '<3' . $digits; // contoh: mhika-frozen<3042
shell_exec("echo '$sshUser:$sshPass' | chpasswd");
```
User Linux dibuat dengan nama sama dengan repo, home directory diarahkan ke folder project, ditambahkan ke group `www` agar bisa baca-tulis file web server. Password di-generate otomatis dan ditampilkan di Job Summary.

`.bash_profile` yang dibuat otomatis berisi alias berguna seperti `php`, `artisan`, `composer`, `restart-app`, `fix-perm`, dan `log-app`. Saat mahasiswa login SSH, langsung masuk ke direktori project dan muncul banner info berisi domain dan perintah yang tersedia.

### Mekanisme Rsync dengan Exclude
Mekanisme *rsync* dengan *exclude* adalah proses sinkronisasi file aplikasi dari paket *deployment* ke direktori server secara selektif, di mana file-file tertentu seperti `.env`, `storage/`, dan `vendor/` dikecualikan dari proses sinkronisasi agar konfigurasi dan data yang telah ada pada server tidak tertimpa oleh proses *deployment* berikutnya.

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
$rsyncCmd = "rsync -rlptD --checksum $rsyncExclude ...";
```
Opsi `--checksum` memastikan hanya file yang benar-benar berubah isinya yang disalin, bukan berdasarkan waktu modifikasi — lebih akurat dan efisien.

### Deteksi Framework & Auto .env
```php
if (file_exists("$target/artisan") && file_exists($composerJsonPath)) {
    if (str_contains($composerContent, 'laravel/framework')) {
        $isLaravel = true;
    }
}
```
Deteksi dua lapis: cek file `artisan` dulu, konfirmasi dengan baca isi `composer.json`. Jika Laravel, `.env` dibuat otomatis dari `.env.example` dengan konfigurasi database yang sudah diprovisioning termasuk handle baris yang dikomentari dengan `#`.

# Permission 3-Layer

## Dasar: Cara Baca Angka Permission Linux

Setiap angka permission terdiri dari 3 digit. Masing-masing digit mewakili hak akses untuk tiga pihak:

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
```

Contoh:
```
7 = 4+2+1 = Read + Write + Execute
6 = 4+2   = Read + Write
5 = 4+1   = Read + Execute
4 = 4     = Read saja
0 = 0     = Tidak ada akses sama sekali
```

---

## Implementasi di Platform Ini

### Layer 1 — Direktori Root Project: `sshUser:sshUser 755`
```
Owner  (sshUser) → 7 = Read + Write + Execute
Group  (sshUser) → 5 = Read + Execute
Others           → 5 = Read + Execute
```
Direktori root project dimiliki oleh SSH user. Semua orang bisa masuk ke direktori ini, tapi hanya SSH user yang bisa mengubah isinya.

---

### Layer 2 — Konten Project: `sshUser:www 775/664`

**Folder (775):**
```
Owner (sshUser) → 7 = Read + Write + Execute
Group (www)     → 7 = Read + Write + Execute
Others          → 5 = Read + Execute
```
Folder bisa dibaca dan ditulis oleh SSH user maupun web server (`www`). Orang lain hanya bisa masuk ke folder, tidak bisa mengubah isinya.

**File (664):**
```
Owner (sshUser) → 6 = Read + Write
Group (www)     → 6 = Read + Write
Others          → 4 = Read saja
```
File bisa dibaca dan ditulis oleh SSH user maupun web server. Orang lain hanya bisa membaca.

---

### Layer 3 — Storage & Cache: `sshUser:www 775`
```
Owner (sshUser) → 7 = Read + Write + Execute
Group (www)     → 7 = Read + Write + Execute
Others          → 5 = Read + Execute
```
Direktori `storage/` dan `bootstrap/cache/` butuh permission tulis oleh web server karena Laravel menyimpan file log, cache view, dan session di sini. Tanpa permission ini Laravel akan error.

---

### Khusus .env: `sshUser:www 640`
```
Owner (sshUser) → 6 = Read + Write
Group (www)     → 4 = Read saja
Others          → 0 = Tidak ada akses sama sekali
```
File `.env` berisi password database dan APP_KEY yang sangat sensitif. Web server hanya perlu membacanya, tidak perlu menulis. Orang lain tidak boleh mengakses sama sekali.

---

## Ringkasan

| Target | Permission | Artinya |
|---|---|---|
| Direktori root | 755 | SSH user bisa ubah, semua bisa masuk |
| Folder project | 775 | SSH user & web server bisa ubah |
| File project | 664 | SSH user & web server bisa tulis, others hanya baca |
| storage/ & cache/ | 775 | Web server harus bisa tulis untuk Laravel |
| .env | 640 | Hanya SSH user & web server yang bisa baca, others diblokir total |
### Provisioning Nginx dan Systemd per Project
*Provisioning* Nginx dan *Systemd* per *project* adalah proses pembangunan infrastruktur server secara otomatis untuk setiap *project* yang didaftarkan pada platform, meliputi pembuatan konfigurasi Nginx sebagai pengatur lalu lintas akses menuju aplikasi yang sesuai dan pembuatan file *systemd service* sebagai pengelola *lifecycle* aplikasi, sehingga setiap *project* memiliki konfigurasi layanan yang terpisah dan independen dalam satu server yang sama.

**Nginx** dikonfigurasi sebagai reverse proxy — meneruskan request dari domain publik ke port lokal tempat aplikasi berjalan:
```php
$nginxConf = "$nginxAvail/$repo.conf";
file_put_contents($nginxConf, $nginxCommon . $nginxBody);
shell_exec("ln -sf $nginxConf $nginxLink");
shell_exec('nginx -t && systemctl reload nginx');
```
Sebelum direload, konfigurasi divalidasi dulu dengan `nginx -t`. Jika ada error, file config dihapus dan deployment dibatalkan.

**Systemd** adalah sistem manajemen layanan pada Linux yang bertugas menjalankan, menghentikan, dan memantau proses aplikasi yang berjalan di server. Pada platform ini, *systemd* digunakan untuk memastikan setiap aplikasi mahasiswa yang telah di-*deploy* selalu berjalan secara otomatis, termasuk menghidupkan kembali aplikasi apabila berhenti karena kesalahan atau server *restart*:
```php
$svcContent .= "Restart=on-failure\n";      // hidupkan lagi kalau crash
$svcContent .= "RestartSec=5s\n";           // tunggu 5 detik sebelum restart
$svcContent .= "ProtectSystem=strict\n";    // keamanan: batasi akses sistem
$svcContent .= "NoNewPrivileges=true\n";    // tidak bisa eskalasi privilege
```
Untuk Laravel dijalankan dengan `php artisan serve`, untuk PHP Native dengan `php -S`.

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

## 5. Tips Praktis: Perbaikan kalau db:seed Gagal

Kalau seeder gagal karena butuh package seperti Faker yang tidak tersedia di production, jalankan perintah ini secara berurutan:

```bash
composer install --optimize-autoloader
php artisan db:seed
composer install --no-dev --optimize-autoloader
restart-app
```

**Kenapa dua kali `composer install`?**

`composer install --optimize-autoloader` → install **semua** dependensi termasuk `require-dev` seperti Faker agar seeder bisa jalan.

`php artisan db:seed` → jalankan seeder yang membutuhkan Faker.

`composer install --no-dev --optimize-autoloader` → install ulang **tanpa** dev dependensi agar server production bersih — tidak ada package development yang tidak diperlukan di server.

`restart-app` → restart service agar perubahan aktif.

---

## 6. Alur Lengkap dari Push sampai Live

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
  ✓ Tulis Nginx config → validasi → reload
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

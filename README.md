# ☁️ Proyek Akhir: Cloud Full-Stack Deployment & GitOps Pipeline

Dokumentasi komprehensif untuk proses kontainerisasi, otomatisasi pipeline GitOps CI/CD, pengamanan infrastruktur dengan enkripsi SSL otomatis, hingga implementasi tumpukan pemantauan (*monitoring stack*) untuk aplikasi **Wibe Studio** pada lingkungan AWS EC2.

---

## 🚀 Arsitektur Sistem & Alur Kerja Deployment

```
[ Developer ] -- Push --> [ GitHub Main Branch ]
                                 │
                       ┌─────────┴─────────┐
                       ▼                   ▼
            [ GitHub Actions CI ]   [ GitHub Actions CD ]
               ├── Lint & Build        ├── SSH ke AWS EC2
               ├── Trivy Security Scan └── Eksekusi deploy-bluegreen.sh
               └── Push ke Docker Hub               │
                                           ┌────────┴────────┐
                                           ▼                 ▼
                                    [ Blue Container ] [ Green Container ]
                                     (Port 8081)        (Port 8082)
                                           ▲                 ▲
                                           └────────┬────────┘
                                                    │
                                             [ Nginx Reverse Proxy ] <─── [ Monitoring Stack ]
                                                    ▲                      (Prometheus & Grafana)
                                                    │
                                             [ HTTPS Traffic ]

```

### Keunggulan Utama Proyek

* **Zero Downtime (Blue-Green):** Pengalihan lalu lintas (*traffic*) otomatis memastikan tidak ada gangguan atau putusnya koneksi pada sisi pengguna saat pembaruan aplikasi diluncurkan.
* **Security First (Trivy Scan):** Inspeksi kerentanan keamanan pada tingkatan kode (*pre-flight image scan*) dilakukan sebelum image didorong ke Docker Hub.
* **Mekanisme Rollback Otomatis:** Sistem akan menghapus container pementasan (*staging*) baru dan mengisolasi trafik pada container lama jika uji kesehatan (*health check*) gagal.
* **Observabilitas Terpusat:** Monitoring metrik server (CPU, RAM, Storage, Load) secara real-time via Grafana dan Prometheus.

---

## 🛠️ Tumpukan Teknologi (Tech Stack)

| Kategori | Teknologi |
| --- | --- |
| **Runtime / Build** | Node.js 22 (Alpine) |
| **Web Server / Edge Proxy** | Nginx (Alpine / Ubuntu Native) |
| **Kontainerisasi** | Docker & Docker Compose |
| **Platform CI/CD** | GitHub Actions |
| **Cloud Infrastructure** | AWS EC2 (Ubuntu 24.04 LTS, tipe t3.micro) |
| **Keamanan & SSL** | Reverse Proxy Nginx, Let's Encrypt TLS (Certbot), Aquasecurity Trivy |
| **Monitoring & Visualisasi** | Prometheus, Grafana, Node Exporter, Portainer CE |

---

## 📋 Prasyarat Sistem (Prerequisites)

* ✅ Akun GitHub & Repositori Aktif.
* ✅ Akun Docker Hub & Personal Access Token (PAT).
* ✅ Akun AWS & Key Pair (`.pem`) yang valid.
* ✅ Domain Publik aktif yang sudah diarahkan ke IP Publik EC2 (`A Record`).
* ✅ Git, Docker, dan Node.js 22 terpasang di komputer lokal untuk pengujian awal.

---

## 📖 Panduan Implementasi Langkah demi Langkah

### 1. Kloning Repositori & Pengembangan Lokal

Unduh kode sumber aplikasi ke lingkungan lokal dan lakukan verifikasi fungsionalitas:

```bash
git clone https://github.com/codebucks27/wibe-studio.git
cd wibe-studio

# Pasang dependensi menggunakan flag legacy peer resolution
npm install --legacy-peer-deps

# Jalankan server pengembangan lokal
npm run dev

```

*Pastikan aplikasi berjalan normal pada port lokal yang ditentukan sebelum masuk ke tahap kontainerisasi.*

### 2. Kontainerisasi dengan Dockerfile Multi-Stage

Buat file bernama `Dockerfile` di direktori utama proyek untuk mengoptimalkan ukuran image produksi:

```dockerfile
# ==========================================
# Tahap 1: Build & Kompilasi
# ==========================================
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install --legacy-peer-deps

COPY . .
RUN npm run build

# ==========================================
# Tahap 2: Lingkungan Eksekusi Produksi
# ==========================================
FROM nginx:alpine

# Salin hasil build statis dari tahap builder ke direktori Nginx
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]

```

#### Build dan uji kontainer Docker secara lokal:

```bash
# Membuat image Docker lokal
docker build -t wibe-studio .

# Menjalankan kontainer dengan memetakan port internal ke port lokal 8080
docker run -d -p 8080:80 wibe-studio

```

*Akses `http://localhost:8080` untuk memastikan container berfungsi dengan baik.*

### 3. Konfigurasi Standar Docker Compose Lokal

Buat file `docker-compose.yml` di komputer lokal Anda:

```yaml
version: '3.8'

services:
  wibe-studio:
    image: devrama404/wibe-studio:latest
    container_name: wibe-studio
    ports:
      - "8080:80"
    restart: unless-stopped

```

Jalankan layanan di latar belakang (*background*):

```bash
docker compose up -d

```

### 4. Sinkronisasi Kode Sumber & Registri Image

Inisialisasi Git dan lakukan unggah (*push*) awal baik ke GitHub maupun Docker Hub:

```bash
# Push ke repositori kontrol versi (GitHub)
git init
git add .
git commit -m "feat: inisialisasi konfigurasi multi-stage docker, compose, dan ci-cd"
git branch -M main
git remote add origin https://github.com/devrama404/wibe-studio.git
git push -u origin main

# Autentikasi dan push ke repositori image (Docker Hub)
docker login
docker tag wibe-studio devrama404/wibe-studio:latest
docker push devrama404/wibe-studio:latest

```

---

## ☁️ Pengaturan Server & Infrastruktur (AWS EC2)

### 5. Inisialisasi Server & Pemasangan Docker Engine

1. Jalankan sebuah *Instance* AWS EC2 (`Ubuntu Server 24.04 LTS`, tipe `t3.micro`).
2. Konfigurasikan **Security Group** EC2 dengan membuka port akses: `80` (HTTP), `443` (HTTPS), `9443` (Portainer), `3001` (Grafana), `9090` (Prometheus), serta port pengujian (`8081`, `8082`).
3. Masuk ke server via SSH dan pasang Docker Engine resmi:

```bash
chmod 400 key.pem
ssh -i key.pem ubuntu@<EC2_PUBLIC_IP>

# Pasang dependensi penandatangan paket dan repositori resmi Docker
sudo apt update && sudo apt install -y ca-certificates curl git
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Berikan hak akses mengeksekusi perintah Docker tanpa sudo ke user ubuntu
sudo usermod -aG docker ubuntu
sudo systemctl enable --now docker

```

*Disarankan untuk keluar (log out) dari terminal SSH dan masuk kembali agar hak akses grup baru aktif sepenuhnya.*

### 6. Pemasangan Nginx & Otomatisasi SSL TLS (Certbot)

Gunakan Nginx bawaan host server sebagai gerbang utama (*Reverse Proxy*):

```bash
sudo apt update && sudo apt install -y nginx certbot python3-certbot-nginx
sudo systemctl enable --now nginx

```

Hapus konfigurasi bawaan dan buat berkas konfigurasi situs baru di `/etc/nginx/sites-available/wibestudio.devrama.my.id`:

```nginx
server {
    listen 80;
    server_name wibestudio.devrama.my.id;

    location / {
        proxy_pass http://127.0.0.1:8081; # Port default mengarah ke Target Blue aktif awal
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

Tautkan konfigurasi situs agar aktif, lalu validasi integritas struktur file Nginx:

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/wibestudio.devrama.my.id /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx

```

#### Amankan domain publik Anda dengan sertifikat SSL Let's Encrypt:

```bash
sudo certbot --nginx -d wibestudio.devrama.my.id

```
*Ikuti petunjuk di layar dan pilih opsi **Redirect (2)** untuk memaksakan pengalihan otomatis lalu lintas HTTP ke HTTPS.*

### Link : [wibestudio.devrama.my.id](url)
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e4fes6jldtzw53j0lujy.png)



---

## 🔒 Otomatisasi Pipeline CI/CD (GitHub Actions)

### 7. Pengaturan GitHub Secrets

Navigasikan repositori online Anda ke **Settings ➔ Secrets and variables ➔ Actions**, kemudian daftarkan variabel sensitif berikut:

| Nama Secret Key | Kegunaan Nilai Rahasia | Contoh Nilai |
| --- | --- | --- |
| `DOCKERHUB_USERNAME` | Identitas pengguna akun Docker Hub | `devrama404` |
| `DOCKERHUB_TOKEN` | Token akses personal keamanan Docker Hub | `dckr_pat_xxxxxxxxxxxxx` |
| `EC2_HOST` | Alamat IP Publik Host Server AWS EC2 | `54.xx.xx.xx` |
| `EC2_USER` | Identitas pengguna sistem default OS | `ubuntu` |
| `SSH_PRIVATE_KEY` | Salinan utuh isi file kunci privat `.pem` | `-----BEGIN OPENSSH PRIVATE KEY-----...` |

### 8. Penyusunan File Workflow Pipeline

Buat berkas otomatisasi pipeline pada repositori lokal Anda di direktori `.github/workflows/deploy.yml`:

```yaml
name: CI-CD Production Pipeline

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Kode Sumber
        uses: actions/checkout@v4

      - name: Inisialisasi Runtime Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22

      - name: Memasang Dependensi Paket Aplikasi
        run: npm install --legacy-peer-deps

      - name: Menjalankan Kompilasi Aplikasi (Build)
        run: npm run build

      - name: Autentikasi Akses Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Membangun Image Docker Produksi
        run: |
          docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/wibe-studio:latest .

      - name: Pemindaian Kerentanan Keamanan Menggunakan Trivy Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: '${{ secrets.DOCKERHUB_USERNAME }}/wibe-studio:latest'
          format: 'table'
          exit-code: '0' # Atur ke '1' jika ingin mematikan pipeline secara paksa saat terdeteksi celah bahaya
          severity: 'HIGH,CRITICAL'

      - name: Mempublikasikan Image ke Registri Docker Hub
        run: |
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/wibe-studio:latest

      - name: Eksekusi Perintah Deployment Jarak Jauh (SSH)
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            ~/deployment/deploy-bluegreen.sh ${{ secrets.DOCKERHUB_USERNAME }}/wibe-studio:latest

```

---

## 🟢 Mesin Otomatisasi Blue-Green Deployment

### 9. Implementasi Shell Script di Server EC2

Masuk kembali ke dalam server AWS EC2 Anda menggunakan SSH, lalu bentuk folder penyimpanan skrip rilis:

```bash
mkdir -p ~/deployment
nano ~/deployment/deploy-bluegreen.sh

```

Tempelkan struktur kode shell dinamis berikut:

```bash
#!/bin/bash
set -e

IMAGE=$1
NGINX_CONF="/etc/nginx/sites-available/wibestudio.devrama.my.id"

echo "========================================="
echo "🔄 Memulai Siklus Rilis Aplikasi: $IMAGE"
echo "========================================="

echo "📥 Menarik manifes image terbaru dari repositori..."
docker pull $IMAGE

# 1. Menjalankan infrastruktur kontainer target Green untuk pementasan aplikasi
echo "🟢 Menyebarkan Lingkungan Staging Green (Port 8082)..."
docker rm -f wibe-green || true
docker run -d --name wibe-green -p 8082:80 --restart unless-stopped $IMAGE

echo "⏳ Mengistirahatkan sistem selama 15 detik untuk proses pemanasan aplikasi..."
sleep 15

# 2. Melakukan evaluasi kelaikan kontainer via pengujian kesehatan lokal
echo "🔍 Menguji Integrasi Kesehatan Sistem (Health Check)..."
if curl -f http://localhost:8082 > /dev/null 2>&1; then
    echo "✅ Pemeriksaan kesehatan sukses! Mengalihkan alur lalu lintas web ke lingkungan Green."
    
    # Memanipulasi port upstream Nginx menggunakan perintah sed
    sudo sed -i 's/127.0.0.1:8081/127.0.0.1:8082/g' $NGINX_CONF
    sudo nginx -t
    sudo systemctl reload nginx

    # 3. Memperbarui versi kontainer Blue utama dengan versi yang telah divalidasi
    echo "🔵 Memperbarui Lingkungan Utama Blue (Port 8081)..."
    docker rm -f wibe-blue || true
    docker run -d --name wibe-blue -p 8081:80 --restart unless-stopped $IMAGE
    
    echo "⏳ Mengistirahatkan sistem selama 10 detik untuk proses pemanasan Blue..."
    sleep 10

    # 4. Mengembalikan gerbang lalu lintas utama internet secara aman ke port 8081
    echo "🔄 Menyelaraskan kembali rute trafik web produksi secara permanen ke Blue..."
    sudo sed -i 's/127.0.0.1:8082/127.0.0.1:8081/g' $NGINX_CONF
    sudo nginx -t
    sudo systemctl reload nginx

    # Membersihkan container transient staging agar tidak memakan sumber daya host
    echo "🧹 Menghapus sisa container pementasan Green..."
    docker rm -f wibe-green
    echo "🎉 Siklus Rilis Sukses Penuh Tanpa Downtime Terdeteksi!"
else
    echo "❌ KESALAHAN FATAL: Uji Kesehatan Gagal Terpenuhi pada Staging Port 8082."
    echo "⚠️ Memulai sistem mitigasi kegagalan otomatis (Automated Rollback)..."
    docker rm -f wibe-green || true
    echo "🚨 Rollback Berhasil. Alur trafik pengguna aman pada container lama."
    exit 1
fi

```

Berikan hak izin akses eksekusi agar skrip dapat dipanggil oleh pipeline GitHub Actions:

```bash
chmod +x ~/deployment/deploy-bluegreen.sh

```

---

## 📊 Manajemen Kontainer & Monitoring Stack

### 10. Akses Cepat Dasbor Monitoring Portainer (Opsional)

Untuk kemudahan administrasi kontainer secara visual, Anda dapat mengaktifkan Portainer di server EC2 Anda:

```bash
docker volume create portainer_data
docker run -d --name portainer --restart=always -p 9000:9000 -p 9443:9443 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest

```

*Akses kontrol visual via browser di alamat: `https://<EC2_PUBLIC_IP>:9443`.*

### 11. Implementasi Prometheus & Grafana Monitoring Stack

Masuk ke server EC2 Anda, buat direktori monitoring terpisah, lalu konfigurasikan metrik peninjau performa server:

```bash
mkdir -p ~/monitoring && cd ~/monitoring
nano docker-compose.yml

```

Isi berkas `docker-compose.yml` pemantauan berikut:

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    ports:
      - "9090:9090"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin # Silakan ganti kata sandi default ini demi keamanan
    volumes:
      - grafana_data:/var/lib/grafana
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: unless-stopped

volumes:
  grafana_data:

```

Selanjutnya, buat file penentu target penarikan metrik di `prometheus.yml`:

```bash
nano prometheus.yml

```

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

```

Jalankan seluruh tumpukan sistem pemantauan server:

```bash
docker compose up -d

```

#### Langkah Menghubungkan Visualisasi Data:

1. Buka tautan dasbor **Grafana** di web browser: `http://<EC2_PUBLIC_IP>:3001` (Gunakan kredensial awal: `admin` / `admin`).
2. Masuk ke menu **Connections** ➔ **Data Sources** ➔ Pilih **Prometheus**.
3. Isi parameter URL dengan nama service internal Docker Compose: `http://prometheus:9090`, lalu klik tombol **Save & Test**.
4. Anda kini dapat membuat Visualisasi Panel menggunakan beberapa fungsi PromQL standar berikut:
* **Penggunaan Kapasitas CPU:** `node_cpu_seconds_total`
* **Sisa Ketersediaan Memori RAM:** `node_memory_MemAvailable_bytes`
* **Beban Muatan Kerja Server:** `node_load1`



---

## 📁 Struktur Hirarki Direktori Proyek

```
📦 wibe-studio (repo-root)
├── 📂 .github/workflows/
│   └── deploy.yml          # Pipeline Otomatisasi GitOps CI/CD
├── 📄 Dockerfile           # Instruksi Dockerisasi Multi-Stage Build
├── 📂 monitoring/
│   ├── docker-compose.yml  # Struktur Tumpukan Monitoring (Grafana/Prometheus)
│   └── prometheus.yml      # Konfigurasi Target Scrape Prometheus
├── 📂 src/                 # Kode Sumber Utama Aplikasi Web
└── 📄 README.md            # File Dokumentasi Proyek Ini

```

---

## ⚠️ Simulasi Pengujian & Solusi Masalah (*Troubleshooting*)

* **Pencegatan Kerentanan (Trivy Scan):** Jika di kemudian hari tim pengembang tidak sengaja memasukkan library yang cacat keamanan, Trivy secara otomatis memunculkan peringatan pada riwayat logs GitHub Actions. Ubah pengaturan parameter variabel `exit-code: '0'` menjadi `'1'` untuk mematikan alur kerja otomatisasi secara paksa apabila ditemukan risiko berlabel *High* atau *Critical*.
* **Simulasi Kegagalan Skenario Rollback:** Untuk menguji fungsionalitas ketahanan sistem pemulihan script, ganti baris fungsi deteksi kesehatan (*health check curl*) di server EC2 Anda ke port acak yang sengaja tidak dialokasikan:
```bash
# Mengubah uji kecocokan ke target port palsu di skrip Anda
curl -f http://localhost:9999

```


Saat komit kode baru didorong, pipeline akan memicu kegagalan uji kesehatan secara sengaja. Skrip akan langsung mengaktifkan logika penyelamatan, mematikan kontainer rusak, dan memelihara trafik web pengguna pada kontainer lama tanpa memicu terjadinya *downtime* sistem.
* **Verifikasi Kunci SSH:** Apabila koneksi CD terhambat, pastikan variabel rahasia `SSH_PRIVATE_KEY` di GitHub Secrets memuat enkapsulasi teks blok utuh tanpa ada spasi atau baris yang terpotong, termasuk baris penanda pembuka `-----BEGIN OPENSSH PRIVATE KEY-----` dan penutup `-----END OPENSSH PRIVATE KEY-----`.

# ☁️ Final Project: Cloud Full-Stack Deployment

Proyek ini mendokumentasikan proses deployment aplikasi web (Node.js + Nginx) ke AWS EC2 menggunakan Docker, CI/CD GitHub Actions, Reverse Proxy dengan SSL otomatis (Let's Encrypt), serta monitoring stack (Prometheus & Grafana).

---

## 🛠 Tech Stack
| Kategori        | Teknologi                          |
|-----------------|------------------------------------|
| Runtime/Build   | Node.js 22 (Alpine)                |
| Web Server      | Nginx (Alpine)                     |
| Container       | Docker & Docker Compose            |
| CI/CD           | GitHub Actions                     |
| Cloud           | AWS EC2 (Ubuntu 24.04 LTS, t3.micro) |
| Security/Proxy  | Nginx Reverse Proxy, Let's Encrypt |
| Monitoring      | Prometheus, Grafana, Node Exporter |

---

## 📋 Prerequisites
- ✅ Akun GitHub & Repository
- ✅ Akun DockerHub & Access Token
- ✅ Akun AWS & Key Pair (`.pem`)
- ✅ Domain yang sudah diarahkan ke IP EC2 (`A Record`)
- ✅ Docker, Git, dan Node.js 22 terinstall di mesin lokal

---

## 🚀 Local Development & Testing
1. **Clone repository**
   ```bash
   git clone https://github.com/codebucks27/wibe-studio.git
   cd repo
   ```
2. **Build & Run Docker container**
   ```bash
   docker build -t wibe-studio .
   docker run -d -p 8080:80 wibe-studio
   ```
3. **Akses aplikasi**  
   Buka browser: `http://localhost:8080`

---

## 📦 Registry & Version Control
### 🔹 Push ke DockerHub
```bash
docker login
docker tag wibe-studio devrama404/wibe-studio:latest
docker push devrama404/wibe-studio:latest
```

### 🔹 Push ke GitHub
```bash
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/devrama404/wibe-studio.git
git push -u origin main
```

---

## ☁️ AWS EC2 Setup
1. **Launch Instance**  
   OS: `Ubuntu Server 24.04 LTS` | Tipe: `t3.micro` | Key Pair: Buat & simpan `.pem`
2. **SSH ke Server**
   ```bash
   chmod 400 key.pem
   ssh -i key.pem ubuntu@<EC2_PUBLIC_IP>
   ```
3. **Install Docker & Git**
   ```bash
   sudo apt update && sudo apt install -y ca-certificates curl git
   sudo install -m 0755 -d /etc/apt/keyrings
   sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
   sudo chmod a+r /etc/apt/keyrings/docker.asc
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   sudo apt update
   sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   sudo systemctl enable --now docker
   ```

---

## 🔄 CI/CD Pipeline (GitHub Actions)
### 1. Setup Secrets
Masuk ke `GitHub Repo → Settings → Secrets and variables → Actions` dan tambahkan:
| Secret Name          | Value Contoh               |
|----------------------|----------------------------|
| `DOCKERHUB_USERNAME` | `devrama404`               |
| `DOCKERHUB_TOKEN`    | `dckr_pat_xxxxxxxxxxxxx`   |
| `EC2_HOST`           | `<IP_EC2>`                 |
| `EC2_USER`           | `ubuntu`                   |
| `SSH_PRIVATE_KEY`    | Isi lengkap `key.pem` Anda |

### 2. Workflow
Buat file `.github/workflows/deploy.yml`. Pipeline akan menjalankan:  
`Checkout → Setup Node 22 → Install Deps → Build App → Docker Login → Build & Push Image → SSH Deploy to EC2 (port 80)`

*(Konfigurasi lengkap YAML tersedia di dokumentasi proyek)*

---

## 🔒 Reverse Proxy & SSL Setup
1. **Install Nginx & Certbot**
   ```bash
   sudo apt update && sudo apt install -y nginx certbot python3-certbot-nginx
   sudo systemctl enable --now nginx
   ```
2. **Konfigurasi Reverse Proxy**
   ```bash
   sudo rm /etc/nginx/sites-enabled/default
   sudo nano /etc/nginx/sites-available/wibestudio.devrama.my.id
   ```
   Isi konfigurasi:
   ```nginx
   server {
       listen 80;
       server_name wibestudio.devrama.my.id;

       location / {
           proxy_pass http://127.0.0.1:8080;
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
   ```bash
   sudo ln -s /etc/nginx/sites-available/wibestudio.devrama.my.id /etc/nginx/sites-enabled/
   sudo nginx -t && sudo systemctl restart nginx
   ```
3. **Aktifkan SSL (Let's Encrypt)**
   ```bash
   sudo certbot --nginx -d wibestudio.devrama.my.id
   ```
   ✅ Ikuti prompt, pilih `Redirect (2)` untuk mengalihkan HTTP ke HTTPS secara otomatis.

---

## 📊 Monitoring Stack
1. **Buat Folder & File Konfigurasi**
   ```bash
   mkdir monitoring && cd monitoring
   nano docker-compose.yml
   nano prometheus.yml
   ```
2. **Jalankan Stack**
   ```bash
   docker compose up -d
   ```
3. **Akses Dashboard**
   - 📈 **Grafana:** `http://<EC2_IP>:3001` | Default: `admin` / `admin`
   - 🕵️ **Prometheus:** `http://<EC2_IP>:9090`
4. **Hubungkan Grafana ke Prometheus**  
   `Settings → Data Sources → Add Prometheus → URL: http://prometheus:9090 → Save & Test`
5. **Buat Panel Dashboard (Contoh PromQL)**
   - CPU: `node_cpu_seconds_total`
   - RAM: `node_memory_MemAvailable_bytes`
   - Load: `node_load1`

---

## 📁 Project Structure
```
📦 repo-root
├── 📂 .github/workflows/
│   └── deploy.yml          # CI/CD Pipeline
├── 📄 Dockerfile           # Node.js Builder → Nginx
├── 📂 monitoring/
│   ├── docker-compose.yml  # Monitoring Stack
│   └── prometheus.yml      # Prometheus Config
└── 📂 src/                 # Source Code Aplikasi
```

---

## ⚠️ Troubleshooting & Catatan
- 🔐 Pastikan **Security Group EC2** membuka port: `80`, `443`, `8080`, `3001`, `9090`.
- 🔄 Port `proxy_pass` di Nginx (`8080`) harus sesuai dengan port mapping container lokal. Di EC2, CI/CD langsung map ke port `80`.
- 🐳 Token DockerHub bisa dibuat di: DockerHub → Settings → Security → Access Tokens.
- 🔑 `SSH_PRIVATE_KEY` di GitHub Secrets harus berisi seluruh isi file `.pem` (termasuk `-----BEGIN...` dan `-----END...`).

---

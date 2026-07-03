# Panduan Lengkap: MariaDB Galera Cluster + Monitoring (Prometheus + Grafana)

## Arsitektur

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   node1     │  │   node2     │  │   node3     │
│ MariaDB     │  │ MariaDB     │  │ MariaDB     │
│ + Galera    │◄─┤ + Galera    │◄─┤ + Galera    │
│ + exporter  │  │ + exporter  │  │ + exporter  │
│ :3306 :9104 │  │ :3306 :9104 │  │ :3306 :9104 │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                         │ scrape :9104
                         ▼
              ┌─────────────────────┐
              │  server-monitoring  │
              │  Prometheus  :9090  │
              │  Alertmanager :9093 │
              │  Grafana     :3000  │
              └─────────────────────┘
```

**4 server dibutuhkan:**
| Server | IP contoh | Peran |
|---|---|---|
| node1 | 192.168.1.10 | Galera node 1 (bootstrap) |
| node2 | 192.168.1.11 | Galera node 2 |
| node3 | 192.168.1.12 | Galera node 3 |
| monitoring | 192.168.1.20 | Prometheus + Alertmanager + Grafana |

Ganti semua IP di panduan ini sesuai environment kamu.

---

## BAGIAN 1 — Install MariaDB Galera Cluster

### 1.1 Jalankan di SEMUA node (node1, node2, node3)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y mariadb-server mariadb-client galera-4
mariadb --version
```

Buka firewall:
```bash
sudo ufw allow 3306/tcp   # MySQL client
sudo ufw allow 4567/tcp   # Replikasi Galera
sudo ufw allow 4567/udp
sudo ufw allow 4568/tcp   # IST (Incremental State Transfer)
sudo ufw allow 4444/tcp   # SST (State Snapshot Transfer)
sudo ufw reload
```

Amankan instalasi (set root password dll):
```bash
sudo mysql_secure_installation
```

Stop MariaDB dulu sebelum konfigurasi Galera:
```bash
sudo systemctl stop mariadb
```

### 1.2 Konfigurasi Galera (beda per node)

Edit `sudo nano /etc/mysql/mariadb.conf.d/60-galera.cnf`

**Template dasar** (isi sama di ketiga node, kecuali 2 baris terakhir):

```ini
[galera]
wsrep_on = ON
wsrep_provider = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name = "my_galera_cluster"
wsrep_cluster_address = "gcomm://192.168.1.10,192.168.1.11,192.168.1.12"
wsrep_sst_method = rsync

binlog_format = ROW
default_storage_engine = InnoDB
innodb_autoinc_lock_mode = 2
bind-address = 0.0.0.0

# --- UBAH SESUAI NODE MASING-MASING ---
wsrep_node_address = "192.168.1.10"
wsrep_node_name = "node1"
```

- Di **node2**: `wsrep_node_address = "192.168.1.11"`, `wsrep_node_name = "node2"`
- Di **node3**: `wsrep_node_address = "192.168.1.12"`, `wsrep_node_name = "node3"`

### 1.3 Bootstrap cluster

**Hanya di node1** (node pertama yang menyalakan cluster):
```bash
sudo galera_new_cluster
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
# Harus: 1
```

**Di node2, lalu node3** (satu per satu, tunggu join sebelum lanjut ke node berikutnya):
```bash
sudo systemctl start mariadb
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

Setelah semua join, cek dari node mana saja:
```sql
SHOW STATUS LIKE 'wsrep_cluster_size';        -- 3
SHOW STATUS LIKE 'wsrep_ready';               -- ON
SHOW STATUS LIKE 'wsrep_local_state_comment'; -- Synced
```

Supaya mudah restart di masa depan, jalankan `systemctl enable mariadb` di ketiga node (tapi ingat: saat restart SEMUA node bersamaan, node pertama yang naik harus di-bootstrap ulang dengan `galera_new_cluster`, bukan `systemctl start` biasa).

```bash
sudo systemctl enable mariadb
```

---

## BAGIAN 2 — mysqld_exporter (install di SEMUA node Galera)

### 2.1 Buat user monitoring

Jalankan **sekali saja** di node mana pun (otomatis replikasi ke semua node karena cluster sudah aktif):

```sql
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'GantiPasswordIni123!' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SLAVE MONITOR ON *.* TO 'exporter'@'localhost';
FLUSH PRIVILEGES;
```

### 2.2 Install exporter (di SETIAP node)

```bash
cd /tmp
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.16.0/mysqld_exporter-0.16.0.linux-amd64.tar.gz
tar xvf mysqld_exporter-0.16.0.linux-amd64.tar.gz
sudo mv mysqld_exporter-0.16.0.linux-amd64/mysqld_exporter /usr/local/bin/
```

```bash
sudo mkdir -p /etc/mysqld_exporter
sudo tee /etc/mysqld_exporter/.my.cnf > /dev/null <<EOF
[client]
user=exporter
password=GantiPasswordIni123!
EOF
sudo chmod 600 /etc/mysqld_exporter/.my.cnf
```

```bash
sudo tee /etc/systemd/system/mysqld_exporter.service > /dev/null <<EOF
[Unit]
Description=Prometheus MySQLd Exporter
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/mysqld_exporter \
  --config.my-cnf=/etc/mysqld_exporter/.my.cnf \
  --collect.info_schema.innodb_metrics \
  --collect.global_status \
  --collect.global_variables \
  --web.listen-address=:9104
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now mysqld_exporter
```

Buka firewall (izinkan hanya dari server monitoring):
```bash
sudo ufw allow from 192.168.1.20 to any port 9104
```

Verifikasi: `curl localhost:9104/metrics | grep wsrep_cluster_size`

---

## BAGIAN 3 — Prometheus (di server monitoring)

```bash
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v3.0.1/prometheus-3.0.1.linux-amd64.tar.gz
tar xvf prometheus-3.0.1.linux-amd64.tar.gz
sudo mv prometheus-3.0.1.linux-amd64 /opt/prometheus
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir -p /var/lib/prometheus /opt/prometheus/rules
sudo chown -R prometheus:prometheus /opt/prometheus /var/lib/prometheus
```

### 3.1 Config utama — `sudo nano /opt/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']

rule_files:
  - "rules/galera_alerts.yml"

scrape_configs:
  - job_name: 'galera_cluster'
    static_configs:
      - targets:
        - '192.168.1.10:9104'
        - '192.168.1.11:9104'
        - '192.168.1.12:9104'
        labels:
          cluster: 'my_galera_cluster'

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### 3.2 Systemd service

```bash
sudo tee /etc/systemd/system/prometheus.service > /dev/null <<EOF
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
ExecStart=/opt/prometheus/prometheus \
  --config.file=/opt/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.listen-address=:9090
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
```

Cek `http://192.168.1.20:9090/targets` — ketiga node harus **UP**.

---

## BAGIAN 4 — Alertmanager (notifikasi)

```bash
cd /tmp
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
tar xvf alertmanager-0.27.0.linux-amd64.tar.gz
sudo mv alertmanager-0.27.0.linux-amd64 /opt/alertmanager
sudo useradd --no-create-home --shell /bin/false alertmanager
sudo mkdir -p /var/lib/alertmanager
sudo chown -R alertmanager:alertmanager /opt/alertmanager /var/lib/alertmanager
```

### 4.1 Config — contoh kirim ke Telegram (ganti sesuai channel notifikasi kamu; bisa juga email/Slack/WhatsApp API)

`sudo nano /opt/alertmanager/alertmanager.yml`:

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'default'
  group_wait: 10s
  group_interval: 1m
  repeat_interval: 1h

receivers:
  - name: 'default'
    # Contoh webhook generik — ganti dengan URL notifikasi kamu
    webhook_configs:
      - url: 'http://localhost:5001/'
        send_resolved: true
```

> Kalau mau notifikasi ke Telegram/Slack/email spesifik, kasih tahu saya platform yang dipakai — saya siapkan config receiver-nya.

### 4.2 Systemd service

```bash
sudo tee /etc/systemd/system/alertmanager.service > /dev/null <<EOF
[Unit]
Description=Alertmanager
After=network.target

[Service]
User=alertmanager
ExecStart=/opt/alertmanager/alertmanager \
  --config.file=/opt/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager \
  --web.listen-address=:9093
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now alertmanager
```

### 4.3 Rule alert Galera — `sudo nano /opt/prometheus/rules/galera_alerts.yml`

```yaml
groups:
  - name: galera_cluster_alerts
    rules:
      - alert: GaleraClusterSizeTurun
        expr: mysql_global_status_wsrep_cluster_size < 3
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Jumlah node cluster Galera turun di {{ $labels.instance }}"
          description: "wsrep_cluster_size = {{ $value }}, seharusnya 3"

      - alert: GaleraNodeTidakSiap
        expr: mysql_global_status_wsrep_ready == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.instance }} tidak wsrep_ready"

      - alert: GaleraNodeTidakSynced
        expr: mysql_global_status_wsrep_local_state != 4
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.instance }} bukan status Synced"
          description: "wsrep_local_state = {{ $value }} (4 = Synced)"

      - alert: GaleraFlowControlPaused
        expr: mysql_global_status_wsrep_flow_control_paused > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Replikasi Galera di {{ $labels.instance }} mengalami flow control tinggi"
          description: "Kemungkinan node lambat/overload, nilai = {{ $value }}"

      - alert: MySQLDown
        expr: mysql_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MySQL/MariaDB di {{ $labels.instance }} down"

      - alert: ExporterDown
        expr: up{job="galera_cluster"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "mysqld_exporter di {{ $labels.instance }} tidak terjangkau"
```

Restart Prometheus supaya rule terbaca:
```bash
sudo chown -R prometheus:prometheus /opt/prometheus/rules
sudo systemctl restart prometheus
```

Cek rule aktif: `http://192.168.1.20:9090/rules`
Cek Alertmanager: `http://192.168.1.20:9093`

---

## BAGIAN 5 — Grafana

```bash
sudo apt install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install -y grafana
sudo systemctl enable --now grafana-server
```

Akses `http://192.168.1.20:3000` (login awal: `admin` / `admin`, langsung diminta ganti password).

### 5.1 Tambah data source

Connections → Data sources → Add data source → **Prometheus** → URL: `http://localhost:9090` → Save & test

### 5.2 Import dashboard siap pakai

Dashboards → New → Import, lalu masukkan salah satu ID dari grafana.com/dashboards:
- **7362** — MySQL Overview
- **14057** — MySQL / Galera Cluster Overview

Pilih data source Prometheus yang baru dibuat → Import.

---

## Checklist verifikasi akhir

```bash
# Di tiap node Galera
systemctl status mariadb mysqld_exporter
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"

# Di server monitoring
systemctl status prometheus alertmanager grafana-server
curl -s localhost:9090/api/v1/targets | grep health
```

- [ ] `wsrep_cluster_size` = 3 di semua node
- [ ] `wsrep_ready` = ON di semua node
- [ ] Semua target Prometheus berstatus UP (`/targets`)
- [ ] Rule alert muncul di `/rules` tanpa error
- [ ] Dashboard Grafana menampilkan data dari ketiga node
- [ ] Test alert (misal stop mariadb di 1 node) → notifikasi muncul di Alertmanager

## Catatan keamanan

- Ganti semua password default (`GantiPasswordIni123!`, admin Grafana) sebelum production.
- Batasi akses port 9090/9093/9104/3000 hanya dari IP internal/VPN, jangan expose ke internet publik.
- Pertimbangkan TLS untuk Grafana kalau diakses lewat jaringan publik.

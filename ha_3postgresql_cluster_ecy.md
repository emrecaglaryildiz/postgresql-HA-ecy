# PostgreSQL Patroni Cluster Kurulum Kılavuzu
### Ubuntu 24.04 LTS — 3 Node HA Yapılandırması

---

## İçindekiler

1. [Mimari ve Bileşenler](#1-mimari-ve-bileşenler)
2. [Ön Gereksinimler](#2-ön-gereksinimler)
3. [Tüm Node'larda Sistem Hazırlığı](#3-tüm-nodelardasystem-hazırlığı)
4. [etcd Cluster Kurulumu](#4-etcd-cluster-kurulumu)
5. [PostgreSQL 16 Kurulumu](#5-postgresql-16-kurulumu)
6. [Patroni Kurulumu ve Yapılandırması](#6-patroni-kurulumu-ve-yapılandırması)
7. [HAProxy Kurulumu](#7-haproxy-kurulumu)
8. [Keepalived — Sanal IP (VIP)](#8-keepalived--sanal-ip-vip)
9. [Cluster'ı Başlatma ve Doğrulama](#9-clusterı-başlatma-ve-doğrulama)
10. [Failover Testi](#10-failover-testi)
11. [Patronictl Referansı](#11-patronictl-referansı)
12. [Sorun Giderme](#12-sorun-giderme)

---

## 1. Mimari ve Bileşenler

### Bileşenler

| Bileşen | Görev |
|---|---|
| **PostgreSQL 16** | Veritabanı motoru |
| **Patroni** | PostgreSQL HA yönetimi, otomatik failover |
| **etcd** | Dağıtık key-value store (Patroni DCS backend) |
| **HAProxy** | Read/Write yönlendirme (5432 → Primary, 5433 → Replica) |
| **Keepalived** | Sanal IP (VIP) — aktif HAProxy node'una bağlar |

### Node Planı

```
┌─────────────────────────────────────────────────────────────┐
│                  VIP: 192.168.10.10                         │
│              (Keepalived — HAProxy üstünde)                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
  ┌──────────┐   ┌──────────┐   ┌──────────┐
  │  node1   │   │  node2   │   │  node3   │
  │.10.11    │   │.10.12    │   │.10.13    │
  │          │   │          │   │          │
  │ etcd     │   │ etcd     │   │ etcd     │
  │ Patroni  │   │ Patroni  │   │ Patroni  │
  │ PG 16    │   │ PG 16    │   │ PG 16    │
  │ HAProxy  │   │ HAProxy  │   │ HAProxy  │
  │Keepalived│   │Keepalived│   │Keepalived│
  └──────────┘   └──────────┘   └──────────┘
  [PRIMARY]      [REPLICA]      [REPLICA]
```

### IP / Hostname Tablosu

| Hostname | IP Adresi | Rol |
|---|---|---|
| pg-node1 | 192.168.10.11 | etcd + Patroni + PG + HAProxy |
| pg-node2 | 192.168.10.12 | etcd + Patroni + PG + HAProxy |
| pg-node3 | 192.168.10.13 | etcd + Patroni + PG + HAProxy |
| — | 192.168.10.10 | Keepalived VIP (Sanal IP) |

> **Not:** IP adreslerini kendi ortamınıza göre değiştirin. Kılavuz boyunca bu değerler kullanılmaktadır.

### Port Planı

| Port | Servis | Açıklama |
|---|---|---|
| 2379 | etcd client | Patroni → etcd bağlantısı |
| 2380 | etcd peer | etcd node'ları arası peer iletişimi |
| 5432 | PostgreSQL | Doğrudan PG bağlantısı |
| 8008 | Patroni REST API | Sağlık kontrolü ve yönetim |
| 5000 | HAProxy R/W | Primary node'a yönlendirir |
| 5001 | HAProxy R/O | Replica node'lara yönlendirir |
| 7000 | HAProxy Stats | Web istatistik arayüzü |
| 112 | Keepalived VRRP | Node'lar arası VRRP iletişimi |

---

## 2. Ön Gereksinimler

### Donanım Minimumu (her node için)

- CPU: 2 vCPU
- RAM: 4 GB
- Disk: 40 GB (OS) + ayrı veri diski önerilir
- Network: Tüm node'lar birbirini görmeli

### Erişim Gereksinimleri

- Her 3 node'a `root` veya `sudo` yetkili kullanıcı ile SSH erişimi
- İnternet erişimi (paket kurulumu için) veya yerel APT mirror

---

## 3. Tüm Node'larda Sistem Hazırlığı

> **ÖNEMLİ:** Bu bölümdeki komutlar **her 3 node'da** (`pg-node1`, `pg-node2`, `pg-node3`) çalıştırılmalıdır.

### 3.1 Sistem Güncellemesi

```bash
apt update && apt upgrade -y
```

### 3.2 /etc/hosts Dosyasını Düzenleme

Her 3 node'da `/etc/hosts` dosyasına aşağıdaki satırları ekleyin:

```bash
cat >> /etc/hosts << 'EOF'
192.168.10.11   pg-node1
192.168.10.12   pg-node2
192.168.10.13   pg-node3
EOF
```

Doğrulama:
```bash
ping -c 2 pg-node1
ping -c 2 pg-node2
ping -c 2 pg-node3
```

### 3.3 Hostname Ayarı

**pg-node1'de:**
```bash
hostnamectl set-hostname pg-node1
```

**pg-node2'de:**
```bash
hostnamectl set-hostname pg-node2
```

**pg-node3'de:**
```bash
hostnamectl set-hostname pg-node3
```

### 3.4 Zaman Senkronizasyonu

Patroni ve etcd için tüm node'larda saat senkronizasyonu kritiktir.

```bash
apt install -y chrony
systemctl enable chrony
systemctl start chrony
chronyc tracking
```

### 3.5 Firewall Kuralları

```bash
# UFW aktif değilse geç, aktifse aşağıdaki kuralları uygula
ufw allow 22/tcp
ufw allow 2379/tcp    # etcd client
ufw allow 2380/tcp    # etcd peer
ufw allow 5432/tcp    # PostgreSQL
ufw allow 8008/tcp    # Patroni REST API
ufw allow 5000/tcp    # HAProxy R/W
ufw allow 5001/tcp    # HAProxy R/O
ufw allow 7000/tcp    # HAProxy Stats
ufw allow 112         # Keepalived VRRP
ufw reload
```

### 3.6 Gerekli Paketler

```bash
apt install -y \
  python3 \
  python3-pip \
  python3-venv \
  python3-psycopg2 \
  wget \
  curl \
  jq \
  net-tools \
  haproxy \
  keepalived
```

---

## 4. etcd Cluster Kurulumu

> **ÖNEMLİ:** Bu bölüm **her 3 node'da** çalıştırılır; ancak konfigürasyon içeriği her node için farklıdır.

### 4.1 etcd Kurulumu

```bash
apt install -y etcd-server etcd-client
```

Servisin durmasını sağlayın (yapılandırmadan önce):

```bash
systemctl stop etcd
systemctl disable etcd
```

### 4.2 etcd Veri Dizini Oluşturma

```bash
mkdir -p /var/lib/etcd
chown etcd:etcd /var/lib/etcd
chmod 700 /var/lib/etcd
```

### 4.3 etcd Konfigürasyonu — pg-node1

**SADECE pg-node1'de** çalıştırın:

```bash
cat > /etc/default/etcd << 'EOF'
ETCD_NAME="pg-node1"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.10.11:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.10.11:2379,http://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.10.11:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.10.11:2379"
ETCD_INITIAL_CLUSTER="pg-node1=http://192.168.10.11:2380,pg-node2=http://192.168.10.12:2380,pg-node3=http://192.168.10.13:2380"
ETCD_INITIAL_CLUSTER_TOKEN="patroni-etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_HEARTBEAT_INTERVAL="1000"
ETCD_ELECTION_TIMEOUT="5000"
ETCD_ENABLE_V2="true"
EOF
```

### 4.4 etcd Konfigürasyonu — pg-node2

**SADECE pg-node2'de** çalıştırın:

```bash
cat > /etc/default/etcd << 'EOF'
ETCD_NAME="pg-node2"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.10.12:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.10.12:2379,http://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.10.12:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.10.12:2379"
ETCD_INITIAL_CLUSTER="pg-node1=http://192.168.10.11:2380,pg-node2=http://192.168.10.12:2380,pg-node3=http://192.168.10.13:2380"
ETCD_INITIAL_CLUSTER_TOKEN="patroni-etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_HEARTBEAT_INTERVAL="1000"
ETCD_ELECTION_TIMEOUT="5000"
ETCD_ENABLE_V2="true"
EOF
```

### 4.5 etcd Konfigürasyonu — pg-node3

**SADECE pg-node3'de** çalıştırın:

```bash
cat > /etc/default/etcd << 'EOF'
ETCD_NAME="pg-node3"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.10.13:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.10.13:2379,http://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.10.13:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.10.13:2379"
ETCD_INITIAL_CLUSTER="pg-node1=http://192.168.10.11:2380,pg-node2=http://192.168.10.12:2380,pg-node3=http://192.168.10.13:2380"
ETCD_INITIAL_CLUSTER_TOKEN="patroni-etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_HEARTBEAT_INTERVAL="1000"
ETCD_ELECTION_TIMEOUT="5000"
ETCD_ENABLE_V2="true"
EOF
```

### 4.6 etcd systemd Service Dosyasını Güncelleme

Ubuntu 24.04'te etcd servis dosyası ortam değişkenlerini okumayabilir. Aşağıdaki override'ı **her 3 node'da** uygulayın:

```bash
mkdir -p /etc/systemd/system/etcd.service.d/
cat > /etc/systemd/system/etcd.service.d/override.conf << 'EOF'
[Service]
EnvironmentFile=/etc/default/etcd
ExecStart=
ExecStart=/usr/bin/etcd
Restart=always
RestartSec=5
EOF

systemctl daemon-reload
```

### 4.7 etcd'yi Başlatma

**Önce pg-node1, sonra pg-node2, sonra pg-node3** sırasıyla:

```bash
systemctl enable etcd
systemctl start etcd
```

Her node'da başlattıktan sonra cluster durumunu kontrol edin:

```bash
etcdctl --endpoints=http://127.0.0.1:2379 member list
```

Beklenen çıktı (3 node listelenmeli):
```
xxxxxxxxxxxxxxxx: name=pg-node1 peerURLs=http://192.168.10.11:2380 clientURLs=http://192.168.10.11:2379 isLeader=true
xxxxxxxxxxxxxxxx: name=pg-node2 peerURLs=http://192.168.10.12:2380 clientURLs=http://192.168.10.12:2379 isLeader=false
xxxxxxxxxxxxxxxx: name=pg-node3 peerURLs=http://192.168.10.13:2380 clientURLs=http://192.168.10.13:2379 isLeader=false
```

Cluster sağlık kontrolü:
```bash
etcdctl --endpoints=http://192.168.10.11:2379,http://192.168.10.12:2379,http://192.168.10.13:2379 endpoint health
```

---

## 5. PostgreSQL 16 Kurulumu

> **ÖNEMLİ:** Bu bölüm **her 3 node'da** çalıştırılır.

### 5.1 PGDG Deposunu Ekleme

```bash
apt install -y curl ca-certificates gnupg
install -d /usr/share/postgresql-common/pgdg
curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail \
  https://www.postgresql.org/media/keys/ACCC4CF8.asc
sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
  https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
  > /etc/apt/sources.list.d/pgdg.list'
apt update
```

### 5.2 PostgreSQL 16 Kurulumu

```bash
apt install -y postgresql-16 postgresql-client-16
```

### 5.3 PostgreSQL Servisini Durdurma

Patroni PostgreSQL'i kendi yönetecektir. Mevcut instance'ı durdurun ve devre dışı bırakın:

```bash
systemctl stop postgresql
systemctl disable postgresql
```

### 5.4 Varsayılan Veri Dizinini Temizleme

Patroni kendi cluster'ını oluşturacaktır. Var olan veri dizinini temizleyin:

```bash
rm -rf /var/lib/postgresql/16/main
mkdir -p /var/lib/postgresql/16/main
chown postgres:postgres /var/lib/postgresql/16/main
chmod 700 /var/lib/postgresql/16/main
```

---

## 6. Patroni Kurulumu ve Yapılandırması

> **ÖNEMLİ:** Bu bölüm **her 3 node'da** çalıştırılır; ancak patroni.yml içeriği her node için farklıdır.

### 6.1 Patroni Kurulumu (pip ile)

```bash
apt install -y python3-pip python3-dev libpq-dev
pip3 install patroni[etcd3] --break-system-packages
```

Kurulumu doğrulayın:
```bash
patroni --version
```

### 6.2 Patroni Konfigürasyon Dizini

```bash
mkdir -p /etc/patroni
```

### 6.3 Patroni Konfigürasyonu — pg-node1

**SADECE pg-node1'de** aşağıdaki dosyayı oluşturun:

```bash
cat > /etc/patroni/patroni.yml << 'EOF'
scope: pg-cluster
namespace: /db/
name: pg-node1

restapi:
  listen: 192.168.10.11:8008
  connect_address: 192.168.10.11:8008

etcd3:
  hosts:
    - 192.168.10.11:2379
    - 192.168.10.12:2379
    - 192.168.10.13:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_keep_size: 1024
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        archive_mode: "off"
        shared_preload_libraries: "pg_stat_statements"
        log_destination: "stderr"
        logging_collector: "on"
        log_directory: "/var/log/postgresql"
        log_filename: "postgresql-%Y-%m-%d_%H%M%S.log"
        log_rotation_age: "1d"
        log_rotation_size: "100MB"
        log_min_duration_statement: 1000
        log_checkpoints: "on"
        log_connections: "on"
        log_disconnections: "on"
        log_lock_waits: "on"
        log_temp_files: 0

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replicator 192.168.10.0/24 md5
    - host replication replicator 127.0.0.1/32 trust
    - host all all 0.0.0.0/0 md5
    - local all all trust

  users:
    admin:
      password: "StrongAdminPass123!"
      options:
        - createrole
        - createdb
    replicator:
      password: "StrongReplPass123!"
      options:
        - replication

postgresql:
  listen: "192.168.10.11,127.0.0.1:5432"
  connect_address: "192.168.10.11:5432"
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: "StrongReplPass123!"
    superuser:
      username: postgres
      password: "StrongPostgresPass123!"
    rewind:
      username: rewind_user
      password: "StrongRewindPass123!"
  parameters:
    unix_socket_directories: "/var/run/postgresql"
    max_connections: 200
    shared_buffers: "512MB"
    effective_cache_size: "1536MB"
    maintenance_work_mem: "128MB"
    checkpoint_completion_target: 0.9
    wal_buffers: "16MB"
    default_statistics_target: 100
    random_page_cost: 1.1
    effective_io_concurrency: 200

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
EOF
```

### 6.4 Patroni Konfigürasyonu — pg-node2

**SADECE pg-node2'de** aşağıdaki dosyayı oluşturun:

```bash
cat > /etc/patroni/patroni.yml << 'EOF'
scope: pg-cluster
namespace: /db/
name: pg-node2

restapi:
  listen: 192.168.10.12:8008
  connect_address: 192.168.10.12:8008

etcd3:
  hosts:
    - 192.168.10.11:2379
    - 192.168.10.12:2379
    - 192.168.10.13:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_keep_size: 1024
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        archive_mode: "off"
        shared_preload_libraries: "pg_stat_statements"
        log_destination: "stderr"
        logging_collector: "on"
        log_directory: "/var/log/postgresql"
        log_filename: "postgresql-%Y-%m-%d_%H%M%S.log"
        log_rotation_age: "1d"
        log_rotation_size: "100MB"
        log_min_duration_statement: 1000
        log_checkpoints: "on"
        log_connections: "on"
        log_disconnections: "on"
        log_lock_waits: "on"
        log_temp_files: 0

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replicator 192.168.10.0/24 md5
    - host replication replicator 127.0.0.1/32 trust
    - host all all 0.0.0.0/0 md5
    - local all all trust

  users:
    admin:
      password: "StrongAdminPass123!"
      options:
        - createrole
        - createdb
    replicator:
      password: "StrongReplPass123!"
      options:
        - replication

postgresql:
  listen: "192.168.10.12,127.0.0.1:5432"
  connect_address: "192.168.10.12:5432"
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: "StrongReplPass123!"
    superuser:
      username: postgres
      password: "StrongPostgresPass123!"
    rewind:
      username: rewind_user
      password: "StrongRewindPass123!"
  parameters:
    unix_socket_directories: "/var/run/postgresql"
    max_connections: 200
    shared_buffers: "512MB"
    effective_cache_size: "1536MB"
    maintenance_work_mem: "128MB"
    checkpoint_completion_target: 0.9
    wal_buffers: "16MB"
    default_statistics_target: 100
    random_page_cost: 1.1
    effective_io_concurrency: 200

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
EOF
```

### 6.5 Patroni Konfigürasyonu — pg-node3

**SADECE pg-node3'de** aşağıdaki dosyayı oluşturun:

```bash
cat > /etc/patroni/patroni.yml << 'EOF'
scope: pg-cluster
namespace: /db/
name: pg-node3

restapi:
  listen: 192.168.10.13:8008
  connect_address: 192.168.10.13:8008

etcd3:
  hosts:
    - 192.168.10.11:2379
    - 192.168.10.12:2379
    - 192.168.10.13:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_keep_size: 1024
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        archive_mode: "off"
        shared_preload_libraries: "pg_stat_statements"
        log_destination: "stderr"
        logging_collector: "on"
        log_directory: "/var/log/postgresql"
        log_filename: "postgresql-%Y-%m-%d_%H%M%S.log"
        log_rotation_age: "1d"
        log_rotation_size: "100MB"
        log_min_duration_statement: 1000
        log_checkpoints: "on"
        log_connections: "on"
        log_disconnections: "on"
        log_lock_waits: "on"
        log_temp_files: 0

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replicator 192.168.10.0/24 md5
    - host replication replicator 127.0.0.1/32 trust
    - host all all 0.0.0.0/0 md5
    - local all all trust

  users:
    admin:
      password: "StrongAdminPass123!"
      options:
        - createrole
        - createdb
    replicator:
      password: "StrongReplPass123!"
      options:
        - replication

postgresql:
  listen: "192.168.10.13,127.0.0.1:5432"
  connect_address: "192.168.10.13:5432"
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: "StrongReplPass123!"
    superuser:
      username: postgres
      password: "StrongPostgresPass123!"
    rewind:
      username: rewind_user
      password: "StrongRewindPass123!"
  parameters:
    unix_socket_directories: "/var/run/postgresql"
    max_connections: 200
    shared_buffers: "512MB"
    effective_cache_size: "1536MB"
    maintenance_work_mem: "128MB"
    checkpoint_completion_target: 0.9
    wal_buffers: "16MB"
    default_statistics_target: 100
    random_page_cost: 1.1
    effective_io_concurrency: 200

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
EOF
```

### 6.6 Log Dizinini Oluşturma

**Her 3 node'da:**

```bash
mkdir -p /var/log/postgresql
chown postgres:postgres /var/log/postgresql
```

### 6.7 Patroni systemd Servis Dosyası

**Her 3 node'da** aşağıdaki servis dosyasını oluşturun:

```bash
cat > /etc/systemd/system/patroni.service << 'EOF'
[Unit]
Description=Patroni — High Availability PostgreSQL Cluster
Documentation=https://patroni.readthedocs.io/en/latest/
After=network.target etcd.service
Wants=etcd.service

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=on-failure
RestartSec=5
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=patroni

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
```

### 6.8 Patroni Konfigürasyon Dosyasının İzinleri

```bash
chown postgres:postgres /etc/patroni/patroni.yml
chmod 640 /etc/patroni/patroni.yml
```

---

## 7. HAProxy Kurulumu

> **ÖNEMLİ:** Bu bölüm **her 3 node'da** çalıştırılır. Konfigürasyon içeriği tüm node'larda aynıdır.

HAProxy, Patroni'nin REST API sağlık endpoint'ini (`/primary`, `/replica`) kullanarak hangi node'un Primary, hangisinin Replica olduğunu dinamik olarak belirler.

### 7.1 HAProxy Konfigürasyonu

**Her 3 node'da** aynı konfigürasyonu uygulayın:

```bash
cat > /etc/haproxy/haproxy.cfg << 'EOF'
global
    maxconn 200
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    mode tcp
    log global
    retries 2
    timeout connect 5s
    timeout client 1m
    timeout server 1m
    option tcplog

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /haproxy
    stats refresh 5s
    stats show-node
    stats show-legends

# Primary — Yazma trafiği (port 5000)
listen pg_primary
    bind *:5000
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-node1 192.168.10.11:5432 check port 8008
    server pg-node2 192.168.10.12:5432 check port 8008
    server pg-node3 192.168.10.13:5432 check port 8008

# Replica — Okuma trafiği (port 5001) — Round-robin
listen pg_replicas
    bind *:5001
    balance roundrobin
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-node1 192.168.10.11:5432 check port 8008
    server pg-node2 192.168.10.12:5432 check port 8008
    server pg-node3 192.168.10.13:5432 check port 8008
EOF
```

### 7.2 HAProxy Başlatma

```bash
systemctl enable haproxy
systemctl start haproxy
systemctl status haproxy
```

---

## 8. Keepalived — Sanal IP (VIP)

> Keepalived, aktif HAProxy node'una bir Sanal IP (VIP: `192.168.10.10`) bağlar. Bir node çöktüğünde VIP otomatik olarak başka bir node'a geçer.

### 8.1 pg-node1 Keepalived Konfigürasyonu

**SADECE pg-node1'de** (`MASTER` — en yüksek öncelik):

```bash
cat > /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
   router_id pg-node1
   script_user root
   enable_script_security
}

vrrp_script chk_haproxy {
    script "/usr/bin/pgrep haproxy"
    interval 2
    weight 2
    fall 2
    rise 1
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass PgCluster2024!
    }
    virtual_ipaddress {
        192.168.10.10/24
    }
    track_script {
        chk_haproxy
    }
}
EOF
```

### 8.2 pg-node2 Keepalived Konfigürasyonu

**SADECE pg-node2'de** (`BACKUP` — orta öncelik):

```bash
cat > /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
   router_id pg-node2
   script_user root
   enable_script_security
}

vrrp_script chk_haproxy {
    script "/usr/bin/pgrep haproxy"
    interval 2
    weight 2
    fall 2
    rise 1
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass PgCluster2024!
    }
    virtual_ipaddress {
        192.168.10.10/24
    }
    track_script {
        chk_haproxy
    }
}
EOF
```

### 8.3 pg-node3 Keepalived Konfigürasyonu

**SADECE pg-node3'de** (`BACKUP` — en düşük öncelik):

```bash
cat > /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
   router_id pg-node3
   script_user root
   enable_script_security
}

vrrp_script chk_haproxy {
    script "/usr/bin/pgrep haproxy"
    interval 2
    weight 2
    fall 2
    rise 1
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 99
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass PgCluster2024!
    }
    virtual_ipaddress {
        192.168.10.10/24
    }
    track_script {
        chk_haproxy
    }
}
EOF
```

> **Not:** `interface eth0` değerini kendi network interface adınızla değiştirin. Kontrol etmek için: `ip link show`

### 8.4 Keepalived Başlatma

**Her 3 node'da:**

```bash
systemctl enable keepalived
systemctl start keepalived
systemctl status keepalived
```

VIP'in pg-node1'e atandığını kontrol edin:

```bash
# pg-node1'de çalıştırın
ip addr show eth0 | grep 192.168.10.10
```

---

## 9. Cluster'ı Başlatma ve Doğrulama

### 9.1 Patroni'yi Başlatma

**Önce pg-node1'de** başlatın (ilk bootstrap için Primary olacak):

```bash
# pg-node1'de:
systemctl enable patroni
systemctl start patroni
```

Başlatma loglarını izleyin:

```bash
journalctl -u patroni -f
```

Beklenen log çıktısı (kısaltılmış):
```
patroni[xxxxx]: INFO: No PostgreSQL configuration file found. Using default
patroni[xxxxx]: INFO: Initialize a new cluster
patroni[xxxxx]: INFO: postmaster started, pid=xxxxx
patroni[xxxxx]: INFO: promoted self to leader via round-robin
patroni[xxxxx]: INFO: Lock owner: pg-node1; I am the leader with the lock
```

pg-node1 başarıyla Primary olduktan sonra **pg-node2 ve pg-node3'ü** başlatın:

```bash
# pg-node2'de:
systemctl enable patroni
systemctl start patroni

# pg-node3'de:
systemctl enable patroni
systemctl start patroni
```

### 9.2 Cluster Durumunu Kontrol Etme

Herhangi bir node'da:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Beklenen çıktı:
```
+ Cluster: pg-cluster (xxxxxxxxxxxxxxxx) +---------+----+-----------+
| Member   | Host            | Role    | State   | TL | Lag in MB |
+----------+-----------------+---------+---------+----+-----------+
| pg-node1 | 192.168.10.11:5432 | Leader  | running |  1 |           |
| pg-node2 | 192.168.10.12:5432 | Replica | running |  1 |         0 |
| pg-node3 | 192.168.10.13:5432 | Replica | running |  1 |         0 |
+----------+-----------------+---------+---------+----+-----------+
```

### 9.3 Patroni REST API Kontrolü

```bash
# Primary kontrolü
curl http://192.168.10.11:8008/primary
# Beklenen: HTTP 200

# Replica kontrolü
curl http://192.168.10.12:8008/replica
# Beklenen: HTTP 200

# Cluster bilgisi
curl -s http://192.168.10.11:8008/cluster | jq .
```

### 9.4 Replikasyon Durumu

Primary node'da (pg-node1):

```bash
psql -U postgres -h 127.0.0.1 -c "SELECT client_addr, state, sync_state, sent_lsn, write_lsn, flush_lsn, replay_lsn FROM pg_stat_replication;"
```

### 9.5 HAProxy Üzerinden Bağlantı Testi

```bash
# Primary'ye yaz bağlantısı (port 5000)
psql -U postgres -h 192.168.10.10 -p 5000 -c "SELECT pg_is_in_recovery(), inet_server_addr();"
# Beklenen: f | 192.168.10.11 (false = primary)

# Replica okuma bağlantısı (port 5001)
psql -U postgres -h 192.168.10.10 -p 5001 -c "SELECT pg_is_in_recovery(), inet_server_addr();"
# Beklenen: t | 192.168.10.12 veya 192.168.10.13 (true = replica)
```

### 9.6 HAProxy İstatistik Sayfası

Tarayıcıdan açın:
```
http://192.168.10.10:7000/haproxy
```

---

## 10. Failover Testi

### 10.1 Manuel Switchover (Planlı Geçiş)

Servisi kesintisiz Primary'yi değiştirmek için `switchover` kullanın:

```bash
# Herhangi bir node'da çalıştırın
patronictl -c /etc/patroni/patroni.yml switchover pg-cluster

# Veya hedef node belirleyerek:
patronictl -c /etc/patroni/patroni.yml switchover pg-cluster --master pg-node1 --candidate pg-node2 --force
```

Sonucu kontrol edin:
```bash
patronictl -c /etc/patroni/patroni.yml list
```

### 10.2 Zorla Failover (Acil Geçiş)

Primary node tamamen erişilemez olduğunda:

```bash
patronictl -c /etc/patroni/patroni.yml failover pg-cluster --master pg-node1 --candidate pg-node2 --force
```

### 10.3 Node'u Cluster'dan Çıkarma ve Ekleme

```bash
# Node'u durdur
systemctl stop patroni   # ilgili node'da

# Cluster durumu — "stopped" görünecek
patronictl -c /etc/patroni/patroni.yml list

# Node'u yeniden başlat — otomatik olarak replica olarak katılır
systemctl start patroni  # ilgili node'da
```

### 10.4 Replikasyon Lag Simülasyonu

```bash
# Primary'de ağır yazma yükü
psql -U postgres -h 192.168.10.10 -p 5000 -c "
  CREATE TABLE IF NOT EXISTS test_lag AS SELECT generate_series(1,1000000) AS id;
"

# Replica'larda lag kontrolü
patronictl -c /etc/patroni/patroni.yml list
```

---

## 11. Patronictl Referansı

| Komut | Açıklama |
|---|---|
| `patronictl -c /etc/patroni/patroni.yml list` | Cluster üyelerini listele |
| `patronictl -c /etc/patroni/patroni.yml topology` | Topoloji görünümü |
| `patronictl -c /etc/patroni/patroni.yml history` | Failover geçmişi |
| `patronictl -c /etc/patroni/patroni.yml switchover pg-cluster` | Planlı Primary geçişi |
| `patronictl -c /etc/patroni/patroni.yml failover pg-cluster` | Zorla failover |
| `patronictl -c /etc/patroni/patroni.yml restart pg-cluster` | Tüm cluster'ı yeniden başlat |
| `patronictl -c /etc/patroni/patroni.yml restart pg-cluster pg-node2` | Belirli node'u yeniden başlat |
| `patronictl -c /etc/patroni/patroni.yml reload pg-cluster` | Konfigürasyonu yeniden yükle |
| `patronictl -c /etc/patroni/patroni.yml pause pg-cluster` | Otomatik failover'ı dondur |
| `patronictl -c /etc/patroni/patroni.yml resume pg-cluster` | Otomatik failover'ı devam ettir |
| `patronictl -c /etc/patroni/patroni.yml edit-config` | DCS'teki konfigürasyonu düzenle |
| `patronictl -c /etc/patroni/patroni.yml show-config` | DCS'teki konfigürasyonu göster |
| `patronictl -c /etc/patroni/patroni.yml reinit pg-cluster pg-node2` | Node'u sıfırla ve yeniden başlat |

---

## 12. Sorun Giderme

### Patroni başlamıyor

```bash
# Detaylı log
journalctl -u patroni -n 100 --no-pager

# YAML sözdizimi kontrolü
python3 -c "import yaml; yaml.safe_load(open('/etc/patroni/patroni.yml'))" && echo "YAML OK"

# etcd erişimi kontrolü
curl http://127.0.0.1:2379/health

# postgresql bin yolu kontrolü
ls -la /usr/lib/postgresql/16/bin/postgres
```

### etcd cluster sağlıksız görünüyor

```bash
# Tüm endpoint'lerin durumu
etcdctl --endpoints=http://192.168.10.11:2379,http://192.168.10.12:2379,http://192.168.10.13:2379 endpoint health

# etcd üyelerini listele
etcdctl --endpoints=http://127.0.0.1:2379 member list -w table

# etcd logları
journalctl -u etcd -n 50 --no-pager
```

### Replica lag yüksek

```bash
# Primary'de replikasyon durumu
psql -U postgres -h 127.0.0.1 -c "SELECT * FROM pg_stat_replication;"

# Replica'da WAL alıcısı durumu
psql -U postgres -h 127.0.0.1 -c "SELECT * FROM pg_stat_wal_receiver;"

# Patroni lag bilgisi
patronictl -c /etc/patroni/patroni.yml list
```

### pg_rewind hatası

Failover sonrası eski Primary cluster'a katılamıyorsa:

```bash
# Eski Primary'de (artık replica olacak node'da)
systemctl stop patroni

# Veri dizinini temizle ve Patroni'nin yeniden oluşturmasına izin ver
rm -rf /var/lib/postgresql/16/main/*

systemctl start patroni
# Patroni otomatik olarak basebackup yapacak
```

### HAProxy backend down görünüyor

```bash
# Patroni REST API'yi manuel kontrol et
curl -v http://192.168.10.11:8008/primary
curl -v http://192.168.10.12:8008/replica

# HAProxy servis logları
journalctl -u haproxy -n 30 --no-pager

# HAProxy konfigürasyon syntax kontrolü
haproxy -c -f /etc/haproxy/haproxy.cfg
```

### Keepalived VIP atanmıyor

```bash
# VRRP durumu
journalctl -u keepalived -n 30 --no-pager

# Network interface adını kontrol et
ip link show
# keepalived.conf içindeki "interface" değeriyle eşleşmeli

# VIP adresi kontrolü
ip addr show | grep 192.168.10.10
```

### Tüm Servislerin Özet Durum Kontrolü

Herhangi bir node'da:

```bash
echo "=== etcd ===" && systemctl is-active etcd
echo "=== Patroni ===" && systemctl is-active patroni
echo "=== HAProxy ===" && systemctl is-active haproxy
echo "=== Keepalived ===" && systemctl is-active keepalived
echo "=== Cluster ===" && patronictl -c /etc/patroni/patroni.yml list
```

---

## Önemli Güvenlik Notları

> Bu kılavuzdaki şifreler (`StrongAdminPass123!` vb.) **örnek değerlerdir**.
> Üretim ortamına geçmeden önce tüm şifreleri güçlü, benzersiz değerlerle değiştirin.
> Şifreleri Vault, AWS Secrets Manager veya benzeri bir secrets yönetim aracında saklayın.
> Üretimde etcd için TLS sertifikaları kullanılması **şiddetle önerilir**.

---

*Kılavuz Sürümü: 1.0 — PostgreSQL 16 + Patroni 3.x + etcd 3.x + Ubuntu 24.04 LTS*

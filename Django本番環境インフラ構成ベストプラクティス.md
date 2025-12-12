# Django本番環境インフラ構成ベストプラクティス

## 目次
1. [推奨インフラ構成](#1-推奨インフラ構成)
2. [各コンポーネントの詳細設定](#2-各コンポーネントの詳細設定)
3. [デプロイ戦略](#3-デプロイ戦略)
4. [監視・運用](#4-監視運用)
5. [コスト最適化](#5-コスト最適化)
6. [スケーラビリティ対応](#6-スケーラビリティ対応)

---

## 1. 推奨インフラ構成

### 1.1 全体アーキテクチャ図

```
[インターネット]
    ↓
[CDN (CloudFront/CloudFlare)]
    ↓
[Load Balancer (ALB/ELB)]
    ↓
    ├─→ [Web Server 1] ──→ [Application Server 1 (Gunicorn)]
    ├─→ [Web Server 2] ──→ [Application Server 2 (Gunicorn)]
    └─→ [Web Server 3] ──→ [Application Server 3 (Gunicorn)]
         ↓                        ↓
    [Nginx]                  [Django App]
         ↓                        ↓
         └────────────────────────┘
                    ↓
         ┌──────────┴──────────┐
         ↓                     ↓
[PostgreSQL Primary]    [Redis Cluster]
         ↓                     ↓
[PostgreSQL Standby]   [Celery Workers]
         ↓
    [S3/Object Storage]
```

### 1.2 小規模構成（月間10万PV程度）

#### 推奨スペック

| コンポーネント | スペック | 台数 | 備考 |
|--------------|---------|------|------|
| Webサーバー | 2 vCPU, 4GB RAM | 1台 | Nginx |
| Appサーバー | 2 vCPU, 4GB RAM | 1台 | Gunicorn (4 workers) |
| DBサーバー | 2 vCPU, 4GB RAM, 50GB SSD | 1台 | PostgreSQL 15 |
| キャッシュサーバー | 1 vCPU, 1GB RAM | 1台 | Redis |
| ストレージ | - | - | S3互換オブジェクトストレージ |

**推奨クラウドプロバイダー構成:**

- **AWS**: t3.medium (Web/App), db.t3.medium (DB), cache.t3.micro (Redis)
- **GCP**: e2-medium (Web/App), db-n1-standard-1 (DB), m1-standard-1 (Redis)
- **Azure**: Standard_B2s (Web/App), Basic (DB), Basic C0 (Redis)

**月額コスト概算**: $150-200

### 1.3 中規模構成（月間100万PV程度）

#### 推奨スペック

| コンポーネント | スペック | 台数 | 備考 |
|--------------|---------|------|------|
| Load Balancer | マネージドサービス | 1台 | ALB/NLB |
| Webサーバー | 2 vCPU, 4GB RAM | 2台 | Nginx |
| Appサーバー | 4 vCPU, 8GB RAM | 2台 | Gunicorn (8 workers) |
| DBサーバー (Primary) | 4 vCPU, 16GB RAM, 200GB SSD | 1台 | PostgreSQL 15 |
| DBサーバー (Standby) | 4 vCPU, 16GB RAM, 200GB SSD | 1台 | Read Replica |
| キャッシュサーバー | 2 vCPU, 4GB RAM | 2台 | Redis Cluster |
| 非同期タスクサーバー | 2 vCPU, 4GB RAM | 2台 | Celery Workers |
| ストレージ | - | - | S3互換オブジェクトストレージ |
| CDN | - | - | CloudFront/CloudFlare |

**推奨クラウドプロバイダー構成:**

- **AWS**: t3.large (Web/App), db.r5.large (DB), cache.r5.large (Redis)
- **GCP**: n1-standard-2 (Web/App), db-n1-highmem-2 (DB), m2-highmem-2 (Redis)
- **Azure**: Standard_D2s_v3 (Web/App), GP_Gen5_2 (DB), Standard C1 (Redis)

**月額コスト概算**: $800-1,200

### 1.4 大規模構成（月間1000万PV以上）

#### 推奨スペック

| コンポーネント | スペック | 台数 | 備考 |
|--------------|---------|------|------|
| Load Balancer | マネージドサービス | 複数AZ | ALB/NLB |
| Webサーバー | 4 vCPU, 8GB RAM | 4台以上 | Nginx (Auto Scaling) |
| Appサーバー | 8 vCPU, 16GB RAM | 4台以上 | Gunicorn (Auto Scaling) |
| DBサーバー (Primary) | 8 vCPU, 32GB RAM, 500GB SSD | 1台 | PostgreSQL 15 |
| DBサーバー (Standby) | 8 vCPU, 32GB RAM, 500GB SSD | 2台以上 | Read Replicas |
| キャッシュサーバー | 4 vCPU, 8GB RAM | 3台以上 | Redis Cluster (Sharding) |
| 非同期タスクサーバー | 4 vCPU, 8GB RAM | 4台以上 | Celery Workers (Auto Scaling) |
| ストレージ | - | - | S3互換オブジェクトストレージ |
| CDN | - | - | CloudFront/CloudFlare (Multi-Region) |
| 検索サーバー | 4 vCPU, 8GB RAM | 3台 | Elasticsearch (オプション) |

**推奨クラウドプロバイダー構成:**

- **AWS**: c5.2xlarge (Web/App), db.r5.2xlarge (DB), cache.r5.2xlarge (Redis)
- **GCP**: n1-highcpu-8 (Web/App), db-n1-highmem-8 (DB), m2-highmem-8 (Redis)
- **Azure**: Standard_D8s_v3 (Web/App), GP_Gen5_8 (DB), Premium P1 (Redis)

**月額コスト概算**: $3,500-5,000

---

## 2. 各コンポーネントの詳細設定

### 2.1 Webサーバー（Nginx）

#### 2.1.1 インストールと基本設定

```bash
# Ubuntu/Debianの場合
sudo apt update
sudo apt install nginx

# CentOS/RHELの場合
sudo yum install epel-release
sudo yum install nginx

# Nginxの起動と自動起動設定
sudo systemctl start nginx
sudo systemctl enable nginx
```

#### 2.1.2 Nginx設定ファイル

```nginx
# /etc/nginx/nginx.conf

user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log warn;

events {
    worker_connections 2048;
    use epoll;
    multi_accept on;
}

http {
    ##
    # 基本設定
    ##
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ##
    # ロギング設定
    ##
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';

    access_log /var/log/nginx/access.log main;

    ##
    # パフォーマンス設定
    ##
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 10M;

    ##
    # Gzip圧縮設定
    ##
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss 
               application/rss+xml font/truetype font/opentype 
               application/vnd.ms-fontobject image/svg+xml;
    gzip_disable "msie6";

    ##
    # セキュリティヘッダー
    ##
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;

    ##
    # SSL設定
    ##
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    ##
    # Rate Limiting（DDoS対策）
    ##
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=general:10m rate=100r/s;

    ##
    # アップストリーム設定（Gunicorn）
    ##
    upstream django_app {
        least_conn;  # 最小接続数による振り分け
        server 127.0.0.1:8000 max_fails=3 fail_timeout=30s;
        # 複数サーバーの場合
        # server app1.example.com:8000 max_fails=3 fail_timeout=30s weight=1;
        # server app2.example.com:8000 max_fails=3 fail_timeout=30s weight=1;
        keepalive 32;
    }

    ##
    # 仮想ホスト設定を読み込み
    ##
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

#### 2.1.3 Django用サイト設定

```nginx
# /etc/nginx/sites-available/django_app

# HTTPからHTTPSへのリダイレクト
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    # Let's Encrypt用
    location ^~ /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS設定
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;

    # SSL証明書設定
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    # HSTS (HTTP Strict Transport Security)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # セキュリティヘッダー
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Content-Security-Policy "default-src 'self' https:; script-src 'self' 'unsafe-inline' 'unsafe-eval' https:; style-src 'self' 'unsafe-inline' https:;" always;

    # ログ設定
    access_log /var/log/nginx/django_access.log main;
    error_log /var/log/nginx/django_error.log warn;

    # ルートディレクトリ
    root /var/www/django_app;

    # 静的ファイル（CSS、JS、画像など）
    location /static/ {
        alias /var/www/django_app/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # メディアファイル（ユーザーアップロード）
    location /media/ {
        alias /var/www/django_app/media/;
        expires 7d;
        add_header Cache-Control "public";
    }

    # favicon
    location = /favicon.ico {
        alias /var/www/django_app/staticfiles/favicon.ico;
        access_log off;
        log_not_found off;
    }

    # robots.txt
    location = /robots.txt {
        alias /var/www/django_app/staticfiles/robots.txt;
        access_log off;
        log_not_found off;
    }

    # 管理画面（IP制限）
    location /admin/ {
        # 特定のIPアドレスのみ許可（オプション）
        # allow 203.0.113.0/24;
        # deny all;

        limit_req zone=api burst=5 nodelay;
        proxy_pass http://django_app;
        include /etc/nginx/proxy_params;
    }

    # API（Rate Limiting）
    location /api/ {
        limit_req zone=api burst=10 nodelay;
        proxy_pass http://django_app;
        include /etc/nginx/proxy_params;
    }

    # その他のリクエスト
    location / {
        limit_req zone=general burst=20 nodelay;
        proxy_pass http://django_app;
        include /etc/nginx/proxy_params;
    }

    # ヘルスチェック用エンドポイント
    location /health/ {
        access_log off;
        proxy_pass http://django_app;
        include /etc/nginx/proxy_params;
    }
}
```

#### 2.1.4 プロキシパラメータ設定

```nginx
# /etc/nginx/proxy_params

proxy_set_header Host $http_host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $server_name;

proxy_redirect off;
proxy_buffering off;

proxy_connect_timeout 60s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;

proxy_http_version 1.1;
proxy_set_header Connection "";
```

#### 2.1.5 Nginx有効化

```bash
# シンボリックリンク作成
sudo ln -s /etc/nginx/sites-available/django_app /etc/nginx/sites-enabled/

# 設定テスト
sudo nginx -t

# Nginx再起動
sudo systemctl reload nginx
```

### 2.2 アプリケーションサーバー（Gunicorn）

#### 2.2.1 Gunicornのインストール

```bash
# 仮想環境内でインストール
pip install gunicorn gevent  # geventは非同期処理用（オプション）
```

#### 2.2.2 Gunicorn設定ファイル

```python
# /var/www/django_app/gunicorn_config.py

import multiprocessing
import os

# ワーカー設定
workers = multiprocessing.cpu_count() * 2 + 1  # CPUコア数 * 2 + 1
worker_class = 'sync'  # または 'gevent' for 非同期
worker_connections = 1000
max_requests = 1000  # メモリリーク対策
max_requests_jitter = 50
timeout = 30
keepalive = 5

# バインド設定
bind = '127.0.0.1:8000'
backlog = 2048

# ロギング
accesslog = '/var/log/gunicorn/access.log'
errorlog = '/var/log/gunicorn/error.log'
loglevel = 'info'
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s" %(D)s'

# プロセス命名
proc_name = 'django_app'

# デーモン化（systemd使用時は不要）
daemon = False
pidfile = '/var/run/gunicorn/gunicorn.pid'

# ユーザー・グループ
user = 'www-data'
group = 'www-data'

# ワーカーの再起動設定
graceful_timeout = 30
preload_app = True  # アプリケーションの事前ロード（メモリ節約）

# セキュリティ
limit_request_line = 4096
limit_request_fields = 100
limit_request_field_size = 8190

# ワーカープロセスの設定
def on_starting(server):
    """サーバー起動時の処理"""
    pass

def on_reload(server):
    """リロード時の処理"""
    pass

def when_ready(server):
    """サーバー準備完了時の処理"""
    pass

def pre_fork(server, worker):
    """ワーカーフォーク前の処理"""
    pass

def post_fork(server, worker):
    """ワーカーフォーク後の処理"""
    pass

def pre_exec(server):
    """新しいマスタープロセス起動前の処理"""
    pass

def worker_int(worker):
    """ワーカー割り込み時の処理"""
    pass

def worker_abort(worker):
    """ワーカー異常終了時の処理"""
    pass
```

#### 2.2.3 Systemdサービス設定

```ini
# /etc/systemd/system/gunicorn.service

[Unit]
Description=Gunicorn daemon for Django application
After=network.target postgresql.service redis.service

[Service]
Type=notify
User=www-data
Group=www-data
RuntimeDirectory=gunicorn
WorkingDirectory=/var/www/django_app
Environment="PATH=/var/www/django_app/venv/bin"
ExecStart=/var/www/django_app/venv/bin/gunicorn \
    --config /var/www/django_app/gunicorn_config.py \
    myproject.wsgi:application
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

#### 2.2.4 Gunicorn起動

```bash
# ログディレクトリ作成
sudo mkdir -p /var/log/gunicorn
sudo chown www-data:www-data /var/log/gunicorn

# サービス有効化と起動
sudo systemctl daemon-reload
sudo systemctl enable gunicorn
sudo systemctl start gunicorn

# ステータス確認
sudo systemctl status gunicorn

# ログ確認
sudo journalctl -u gunicorn -f
```

### 2.3 データベースサーバー（PostgreSQL）

#### 2.3.1 PostgreSQLのインストール

```bash
# Ubuntu/Debianの場合
sudo apt update
sudo apt install postgresql-15 postgresql-contrib-15

# CentOS/RHELの場合
sudo yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo yum install postgresql15-server postgresql15-contrib
sudo /usr/pgsql-15/bin/postgresql-15-setup initdb
```

#### 2.3.2 PostgreSQL基本設定

```ini
# /etc/postgresql/15/main/postgresql.conf

# 接続設定
listen_addresses = '*'  # または特定のIPアドレス
port = 5432
max_connections = 100

# メモリ設定（サーバーのRAMに応じて調整）
shared_buffers = 4GB  # RAMの25%程度
effective_cache_size = 12GB  # RAMの75%程度
maintenance_work_mem = 1GB
work_mem = 32MB

# WAL（Write-Ahead Logging）設定
wal_level = replica
wal_buffers = 16MB
checkpoint_completion_target = 0.9
max_wal_size = 2GB
min_wal_size = 1GB

# クエリプランナー設定
random_page_cost = 1.1  # SSD使用時
effective_io_concurrency = 200  # SSD使用時

# ロギング設定
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_min_duration_statement = 1000  # 1秒以上のクエリをログに記録
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_timezone = 'Asia/Tokyo'

# レプリケーション設定（スタンバイサーバー用）
hot_standby = on
max_wal_senders = 3
wal_keep_size = 64

# 自動VACUUM設定
autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = 1min
```

#### 2.3.3 PostgreSQL接続許可設定

```bash
# /etc/postgresql/15/main/pg_hba.conf

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# ローカル接続
local   all             postgres                                peer
local   all             all                                     peer

# IPv4接続
host    all             all             127.0.0.1/32            md5
host    all             all             10.0.0.0/8              md5  # プライベートネットワーク

# IPv6接続
host    all             all             ::1/128                 md5

# レプリケーション接続
host    replication     replicator      10.0.0.0/8              md5
```

#### 2.3.4 データベースとユーザーの作成

```bash
# PostgreSQLに接続
sudo -u postgres psql

# データベース作成
CREATE DATABASE django_db;

# ユーザー作成
CREATE USER django_user WITH PASSWORD 'secure_password_here';

# 権限付与
ALTER ROLE django_user SET client_encoding TO 'utf8';
ALTER ROLE django_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE django_user SET timezone TO 'Asia/Tokyo';
GRANT ALL PRIVILEGES ON DATABASE django_db TO django_user;

# データベース所有者を変更
ALTER DATABASE django_db OWNER TO django_user;

# 接続
\c django_db

# public スキーマの権限付与
GRANT ALL ON SCHEMA public TO django_user;

# 終了
\q
```

#### 2.3.5 PostgreSQL起動と自動起動設定

```bash
sudo systemctl enable postgresql
sudo systemctl start postgresql
sudo systemctl status postgresql
```

#### 2.3.6 PostgreSQLバックアップスクリプト

```bash
#!/bin/bash
# /usr/local/bin/postgres_backup.sh

# 設定
BACKUP_DIR="/backup/postgresql"
DB_NAME="django_db"
DB_USER="django_user"
DATE=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz"
RETENTION_DAYS=7

# バックアップディレクトリ作成
mkdir -p $BACKUP_DIR

# バックアップ実行
pg_dump -U $DB_USER -h localhost $DB_NAME | gzip > $BACKUP_FILE

# 古いバックアップを削除（7日以上前）
find $BACKUP_DIR -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete

# S3にアップロード（オプション）
# aws s3 cp $BACKUP_FILE s3://your-bucket/backups/postgresql/

echo "Backup completed: $BACKUP_FILE"
```

#### 2.3.7 Cronでバックアップ自動化

```bash
# crontabを編集
sudo crontab -e

# 毎日午前3時にバックアップ実行
0 3 * * * /usr/local/bin/postgres_backup.sh >> /var/log/postgres_backup.log 2>&1
```

#### 2.3.8 PostgreSQLパフォーマンスチューニング

```sql
-- インデックスの作成例
CREATE INDEX idx_user_email ON auth_user(email);
CREATE INDEX idx_created_at ON myapp_model(created_at);

-- 部分インデックス（条件付きインデックス）
CREATE INDEX idx_active_users ON auth_user(username) WHERE is_active = true;

-- VACUUM実行（定期的なメンテナンス）
VACUUM ANALYZE;

-- テーブルのサイズ確認
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- 遅いクエリの確認
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### 2.4 キャッシュサーバー（Redis）

#### 2.4.1 Redisのインストール

```bash
# Ubuntu/Debianの場合
sudo apt update
sudo apt install redis-server

# CentOS/RHELの場合
sudo yum install epel-release
sudo yum install redis
```

#### 2.4.2 Redis設定

```conf
# /etc/redis/redis.conf

# ネットワーク設定
bind 127.0.0.1
port 6379
protected-mode yes
tcp-backlog 511

# セキュリティ設定
requirepass your_secure_redis_password_here

# メモリ設定
maxmemory 2gb
maxmemory-policy allkeys-lru  # LRUでキャッシュを削除

# 永続化設定（キャッシュのみの場合は無効化）
save ""
# save 900 1
# save 300 10
# save 60 10000

# AOF（Append Only File）無効化（キャッシュのみの場合）
appendonly no

# ログ設定
loglevel notice
logfile /var/log/redis/redis-server.log

# パフォーマンス設定
timeout 300
tcp-keepalive 300
databases 16

# クライアント接続数
maxclients 10000

# スローログ設定
slowlog-log-slower-than 10000  # 10ms以上のコマンドをログに記録
slowlog-max-len 128
```

#### 2.4.3 Redis起動と自動起動設定

```bash
sudo systemctl enable redis-server
sudo systemctl start redis-server
sudo systemctl status redis-server
```

#### 2.4.4 Djangoでの設定

```python
# settings/production.py

# Redis設定
REDIS_HOST = os.environ.get('REDIS_HOST', '127.0.0.1')
REDIS_PORT = int(os.environ.get('REDIS_PORT', 6379))
REDIS_PASSWORD = os.environ.get('REDIS_PASSWORD', '')
REDIS_DB = int(os.environ.get('REDIS_DB', 0))

# キャッシュ設定
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': f'redis://:{REDIS_PASSWORD}@{REDIS_HOST}:{REDIS_PORT}/{REDIS_DB}',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'SOCKET_CONNECT_TIMEOUT': 5,
            'SOCKET_TIMEOUT': 5,
            'COMPRESSOR': 'django_redis.compressors.zlib.ZlibCompressor',
            'CONNECTION_POOL_KWARGS': {
                'max_connections': 50,
                'retry_on_timeout': True
            }
        },
        'KEY_PREFIX': 'django_app',
        'TIMEOUT': 300,  # 5分
    }
}

# セッションストレージをRedisに変更
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'
```

### 2.5 非同期タスク処理（Celery）

#### 2.5.1 Celeryのインストール

```bash
pip install celery[redis] flower  # flowerは監視ツール
```

#### 2.5.2 Celery設定

```python
# myproject/celery.py

import os
from celery import Celery

# Django設定モジュールを指定
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings.production')

app = Celery('myproject')

# Django設定から読み込む（CELERY_で始まる設定）
app.config_from_object('django.conf:settings', namespace='CELERY')

# 自動的にタスクを検出
app.autodiscover_tasks()

@app.task(bind=True, ignore_result=True)
def debug_task(self):
    print(f'Request: {self.request!r}')
```

```python
# myproject/__init__.py

from .celery import app as celery_app

__all__ = ('celery_app',)
```

```python
# settings/production.py

# Celery設定
CELERY_BROKER_URL = f'redis://:{REDIS_PASSWORD}@{REDIS_HOST}:{REDIS_PORT}/1'
CELERY_RESULT_BACKEND = f'redis://:{REDIS_PASSWORD}@{REDIS_HOST}:{REDIS_PORT}/2'

CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'Asia/Tokyo'

# タスク設定
CELERY_TASK_TRACK_STARTED = True
CELERY_TASK_TIME_LIMIT = 30 * 60  # 30分
CELERY_TASK_SOFT_TIME_LIMIT = 25 * 60  # 25分

# ワーカー設定
CELERY_WORKER_PREFETCH_MULTIPLIER = 4
CELERY_WORKER_MAX_TASKS_PER_CHILD = 1000

# 結果の保存期限
CELERY_RESULT_EXPIRES = 3600  # 1時間

# 定期タスク設定（Celery Beat）
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    'cleanup-old-sessions': {
        'task': 'myapp.tasks.cleanup_old_sessions',
        'schedule': crontab(hour=3, minute=0),  # 毎日午前3時
    },
    'send-daily-report': {
        'task': 'myapp.tasks.send_daily_report',
        'schedule': crontab(hour=9, minute=0),  # 毎日午前9時
    },
}
```

#### 2.5.3 Celery Systemdサービス設定

```ini
# /etc/systemd/system/celery.service

[Unit]
Description=Celery Service
After=network.target redis.service

[Service]
Type=forking
User=www-data
Group=www-data
EnvironmentFile=/var/www/django_app/.env
WorkingDirectory=/var/www/django_app
ExecStart=/var/www/django_app/venv/bin/celery -A myproject worker \
    --loglevel=info \
    --logfile=/var/log/celery/worker.log \
    --pidfile=/var/run/celery/worker.pid \
    --detach
ExecStop=/bin/kill -s TERM $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/celerybeat.service

[Unit]
Description=Celery Beat Service
After=network.target redis.service

[Service]
Type=simple
User=www-data
Group=www-data
EnvironmentFile=/var/www/django_app/.env
WorkingDirectory=/var/www/django_app
ExecStart=/var/www/django_app/venv/bin/celery -A myproject beat \
    --loglevel=info \
    --logfile=/var/log/celery/beat.log \
    --pidfile=/var/run/celery/beat.pid
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

#### 2.5.4 Celeryサービス起動

```bash
# ログディレクトリ作成
sudo mkdir -p /var/log/celery /var/run/celery
sudo chown www-data:www-data /var/log/celery /var/run/celery

# サービス有効化と起動
sudo systemctl daemon-reload
sudo systemctl enable celery celerybeat
sudo systemctl start celery celerybeat

# ステータス確認
sudo systemctl status celery celerybeat
```

### 2.6 ストレージ（S3互換オブジェクトストレージ）

#### 2.6.1 django-storagesのインストール

```bash
pip install django-storages[boto3]
```

#### 2.6.2 Django設定

```python
# settings/production.py

# Storages設定
INSTALLED_APPS += ['storages']

# AWS S3設定
AWS_ACCESS_KEY_ID = os.environ.get('AWS_ACCESS_KEY_ID')
AWS_SECRET_ACCESS_KEY = os.environ.get('AWS_SECRET_ACCESS_KEY')
AWS_STORAGE_BUCKET_NAME = os.environ.get('AWS_STORAGE_BUCKET_NAME')
AWS_S3_REGION_NAME = os.environ.get('AWS_S3_REGION_NAME', 'ap-northeast-1')
AWS_S3_CUSTOM_DOMAIN = f'{AWS_STORAGE_BUCKET_NAME}.s3.{AWS_S3_REGION_NAME}.amazonaws.com'

# S3設定
AWS_S3_OBJECT_PARAMETERS = {
    'CacheControl': 'max-age=86400',  # 1日
}
AWS_DEFAULT_ACL = 'public-read'
AWS_S3_FILE_OVERWRITE = False
AWS_QUERYSTRING_AUTH = False

# 静的ファイル設定
STATICFILES_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
STATIC_URL = f'https://{AWS_S3_CUSTOM_DOMAIN}/static/'

# メディアファイル設定
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
MEDIA_URL = f'https://{AWS_S3_CUSTOM_DOMAIN}/media/'

# カスタムストレージクラス（静的ファイルとメディアファイルを分ける場合）
# storage_backends.py
from storages.backends.s3boto3 import S3Boto3Storage

class StaticStorage(S3Boto3Storage):
    location = 'static'
    default_acl = 'public-read'

class MediaStorage(S3Boto3Storage):
    location = 'media'
    default_acl = 'private'
    file_overwrite = False

# settings.py
STATICFILES_STORAGE = 'myproject.storage_backends.StaticStorage'
DEFAULT_FILE_STORAGE = 'myproject.storage_backends.MediaStorage'
```

### 2.7 SSL/TLS証明書（Let's Encrypt）

#### 2.7.1 Certbotのインストール

```bash
# Ubuntu/Debian
sudo apt install certbot python3-certbot-nginx

# CentOS/RHEL
sudo yum install certbot python3-certbot-nginx
```

#### 2.7.2 証明書の取得

```bash
# Nginx用
sudo certbot --nginx -d example.com -d www.example.com

# または手動取得
sudo certbot certonly --webroot -w /var/www/certbot \
    -d example.com -d www.example.com \
    --email admin@example.com \
    --agree-tos \
    --no-eff-email
```

#### 2.7.3 自動更新設定

```bash
# Cron設定
sudo crontab -e

# 毎日午前2時に更新チェック
0 2 * * * certbot renew --quiet --post-hook "systemctl reload nginx"
```

---

## 3. デプロイ戦略

### 3.1 Djangoプロジェクト構成

```
/var/www/django_app/
├── .env                    # 環境変数
├── .env.example            # 環境変数のサンプル
├── .gitignore
├── manage.py
├── requirements/
│   ├── base.txt           # 共通の依存関係
│   ├── production.txt     # 本番環境用
│   └── development.txt    # 開発環境用
├── myproject/
│   ├── __init__.py
│   ├── asgi.py
│   ├── celery.py
│   ├── wsgi.py
│   ├── urls.py
│   └── settings/
│       ├── __init__.py
│       ├── base.py        # 共通設定
│       ├── development.py # 開発環境設定
│       └── production.py  # 本番環境設定
├── apps/
│   ├── app1/
│   └── app2/
├── static/                # 開発時の静的ファイル
├── staticfiles/           # collectstatic後の静的ファイル
├── media/                 # ローカルメディアファイル
├── templates/             # テンプレート
├── logs/                  # ログファイル
└── venv/                  # 仮想環境
```

### 3.2 デプロイスクリプト

```bash
#!/bin/bash
# /usr/local/bin/deploy.sh

set -e  # エラー時に停止

# 設定
APP_DIR="/var/www/django_app"
VENV_DIR="$APP_DIR/venv"
BRANCH="main"
USER="www-data"

echo "==== Starting deployment ===="

# リポジトリを更新
cd $APP_DIR
sudo -u $USER git fetch origin
sudo -u $USER git checkout $BRANCH
sudo -u $USER git pull origin $BRANCH

# 仮想環境のアクティベート
source $VENV_DIR/bin/activate

# 依存関係のインストール
pip install -r requirements/production.txt --no-cache-dir

# マイグレーション
python manage.py migrate --noinput

# 静的ファイルの収集
python manage.py collectstatic --noinput

# テストの実行（オプション）
# python manage.py test

# Gunicornの再起動
sudo systemctl restart gunicorn

# Celeryの再起動
sudo systemctl restart celery celerybeat

# Nginxの設定テストとリロード
sudo nginx -t && sudo systemctl reload nginx

echo "==== Deployment completed ===="
```

### 3.3 ゼロダウンタイムデプロイ

```bash
#!/bin/bash
# /usr/local/bin/deploy_zero_downtime.sh

set -e

APP_DIR="/var/www/django_app"
VENV_DIR="$APP_DIR/venv"
BRANCH="main"
USER="www-data"

echo "==== Starting zero-downtime deployment ===="

# 1. コードの更新
cd $APP_DIR
sudo -u $USER git fetch origin
sudo -u $USER git checkout $BRANCH
sudo -u $USER git pull origin $BRANCH

# 2. 依存関係のインストール
source $VENV_DIR/bin/activate
pip install -r requirements/production.txt --no-cache-dir

# 3. マイグレーション（ダウンタイムなし）
python manage.py migrate --noinput

# 4. 静的ファイルの収集
python manage.py collectstatic --noinput

# 5. ヘルスチェック
health_check() {
    curl -f http://localhost:8000/health/ > /dev/null 2>&1
    return $?
}

# 6. Gunicornのグレースフルリスタート
echo "Restarting Gunicorn..."
sudo systemctl reload gunicorn

# 7. 新しいプロセスの起動を待つ
sleep 5

# 8. ヘルスチェック
if health_check; then
    echo "Health check passed"
else
    echo "Health check failed! Rolling back..."
    sudo systemctl restart gunicorn
    exit 1
fi

# 9. Celeryの再起動
sudo systemctl restart celery celerybeat

echo "==== Zero-downtime deployment completed ===="
```

### 3.4 CI/CDパイプライン（GitHub Actions例）

```yaml
# .github/workflows/deploy.yml

name: Deploy to Production

on:
  push:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements/production.txt
      
      - name: Run tests
        env:
          DATABASE_URL: postgresql://test_user:test_password@localhost:5432/test_db
        run: |
          python manage.py test
      
      - name: Run linting
        run: |
          pip install flake8 black
          flake8 .
          black --check .
      
      - name: Security check
        run: |
          pip install safety bandit
          safety check
          bandit -r . -f json -o bandit-report.json

  deploy:
    needs: test
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            /usr/local/bin/deploy_zero_downtime.sh
      
      - name: Notify deployment
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment to production succeeded!"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 4. 監視・運用

### 4.1 ログ管理

#### 4.1.1 集中ログ管理（Logrotate）

```bash
# /etc/logrotate.d/django

/var/log/gunicorn/*.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    missingok
    sharedscripts
    postrotate
        systemctl reload gunicorn > /dev/null 2>&1 || true
    endscript
}

/var/log/celery/*.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    missingok
    sharedscripts
    postrotate
        systemctl restart celery > /dev/null 2>&1 || true
    endscript
}

/var/www/django_app/logs/*.log {
    daily
    rotate 30
    compress
    delaycompress
    notifempty
    missingok
}
```

### 4.2 監視ツール

#### 4.2.1 システム監視（Prometheus + Grafana）

```yaml
# docker-compose.yml（監視サーバー用）

version: '3'

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    restart: unless-stopped

  node_exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

```yaml
# prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node_exporter:9100']

  - job_name: 'django_app'
    static_configs:
      - targets: ['app1.example.com:8000', 'app2.example.com:8000']
```

#### 4.2.2 アプリケーション監視（Sentry）

```python
# settings/production.py

import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration
from sentry_sdk.integrations.celery import CeleryIntegration
from sentry_sdk.integrations.redis import RedisIntegration

sentry_sdk.init(
    dsn=os.environ.get('SENTRY_DSN'),
    integrations=[
        DjangoIntegration(),
        CeleryIntegration(),
        RedisIntegration(),
    ],
    traces_sample_rate=0.1,  # パフォーマンス監視のサンプリング率
    send_default_pii=False,  # 個人情報を送信しない
    environment='production',
    release=os.environ.get('GIT_COMMIT_SHA'),
)
```

### 4.3 ヘルスチェックエンドポイント

```python
# apps/core/views.py

from django.http import JsonResponse
from django.db import connection
from django.core.cache import cache
import redis

def health_check(request):
    """
    ヘルスチェックエンドポイント
    """
    status = {
        'status': 'healthy',
        'checks': {}
    }
    
    # データベース接続チェック
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        status['checks']['database'] = 'ok'
    except Exception as e:
        status['status'] = 'unhealthy'
        status['checks']['database'] = f'error: {str(e)}'
    
    # Redisキャッシュチェック
    try:
        cache.set('health_check', 'ok', 10)
        cache_value = cache.get('health_check')
        if cache_value == 'ok':
            status['checks']['cache'] = 'ok'
        else:
            status['status'] = 'unhealthy'
            status['checks']['cache'] = 'error: value mismatch'
    except Exception as e:
        status['status'] = 'unhealthy'
        status['checks']['cache'] = f'error: {str(e)}'
    
    # レスポンスコード
    status_code = 200 if status['status'] == 'healthy' else 503
    
    return JsonResponse(status, status=status_code)
```

---

## 5. コスト最適化

### 5.1 コスト削減のポイント

1. **Reserved Instances / Savings Plans の活用**
   - 1年または3年契約で最大72%割引
   - 安定したワークロードに適用

2. **Auto Scalingの活用**
   - トラフィックに応じてインスタンス数を自動調整
   - 夜間・休日はスケールダウン

3. **スポットインスタンスの活用**
   - バッチ処理やCelery Workerに活用
   - 最大90%割引

4. **CDNの活用**
   - 静的ファイルをCDNで配信
   - オリジンサーバーの負荷軽減

5. **データベース最適化**
   - 不要なインデックスの削除
   - クエリの最適化
   - 接続プーリングの活用

### 5.2 コスト監視

```bash
# AWS Cost Explorerを使用した定期レポート
aws ce get-cost-and-usage \
    --time-period Start=2025-12-01,End=2025-12-31 \
    --granularity DAILY \
    --metrics UnblendedCost \
    --group-by Type=DIMENSION,Key=SERVICE
```

---

## 6. スケーラビリティ対応

### 6.1 水平スケーリング

```python
# settings/production.py

# データベース接続プーリング
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST'),
        'PORT': os.environ.get('DB_PORT', '5432'),
        'CONN_MAX_AGE': 600,  # 接続プーリング
        'OPTIONS': {
            'connect_timeout': 10,
            'options': '-c statement_timeout=30000'  # 30秒
        }
    },
    'replica': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_REPLICA_HOST'),
        'PORT': os.environ.get('DB_PORT', '5432'),
        'CONN_MAX_AGE': 600,
        'OPTIONS': {
            'connect_timeout': 10,
        }
    }
}

# データベースルーティング
DATABASE_ROUTERS = ['myproject.routers.PrimaryReplicaRouter']
```

```python
# myproject/routers.py

class PrimaryReplicaRouter:
    """
    読み取りクエリをレプリカに、書き込みクエリをプライマリに振り分け
    """
    def db_for_read(self, model, **hints):
        return 'replica'
    
    def db_for_write(self, model, **hints):
        return 'default'
    
    def allow_relation(self, obj1, obj2, **hints):
        return True
    
    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == 'default'
```

### 6.2 Auto Scaling設定（AWS例）

```bash
# Launch Templateの作成
aws ec2 create-launch-template \
    --launch-template-name django-app-template \
    --version-description "Initial version" \
    --launch-template-data file://launch-template-data.json

# Auto Scaling Groupの作成
aws autoscaling create-auto-scaling-group \
    --auto-scaling-group-name django-app-asg \
    --launch-template LaunchTemplateName=django-app-template \
    --min-size 2 \
    --max-size 10 \
    --desired-capacity 2 \
    --target-group-arns arn:aws:elasticloadbalancing:region:account-id:targetgroup/django-tg \
    --health-check-type ELB \
    --health-check-grace-period 300

# スケーリングポリシーの設定
aws autoscaling put-scaling-policy \
    --auto-scaling-group-name django-app-asg \
    --policy-name scale-up \
    --scaling-adjustment 1 \
    --adjustment-type ChangeInCapacity \
    --cooldown 300
```

---

## まとめ

本ドキュメントでは、Djangoアプリケーションのベストプラクティスなインフラ構成を提示しました。

**重要ポイント:**

1. **セキュリティ**: HTTPS、ファイアウォール、適切な権限設定
2. **パフォーマンス**: キャッシング、接続プーリング、静的ファイルのCDN配信
3. **可用性**: ロードバランサー、複数AZ、データベースレプリケーション
4. **監視**: ログ、メトリクス、アラート
5. **自動化**: デプロイ自動化、バックアップ自動化、スケーリング自動化
6. **コスト最適化**: 適切なインスタンスサイズ、Reserved Instances、Auto Scaling

これらの要素を適切に組み合わせることで、安定した本番環境を構築できます。

# EC2 (Rocky Linux 9) PostgreSQL バージョン確認・アップグレード手順

## 環境情報

- OS: Rocky Linux 9 (RHEL系)
- PostgreSQL: 17.x (PGDG公式リポジトリ)
- データディレクトリ: `/var/lib/pgsql/17/data`
- 接続ユーザー: `admin`
- 接続方法: `-U admin -h 127.0.0.1`

## 1. 現在のバージョン確認

```bash
psql --version
psql -U admin -h 127.0.0.1 -d postgres -c "SELECT version();"
```

## 2. バックアップ

### 2-1. パスワード入力省略の準備

```bash
echo "127.0.0.1:5432:*:admin:<パスワード>" > ~/.pgpass
chmod 600 ~/.pgpass
```

### 2-2. データベースのバックアップ

```bash
# 全データベースの論理バックアップ
pg_dumpall -U admin -h 127.0.0.1 > /tmp/pg_backup_all.sql

# 個別データベースのカスタム形式バックアップ (圧縮・並列リストア可能)
pg_dump -U admin -h 127.0.0.1 -Fc -d <データベース名> -f /tmp/<データベース名>.dump
```

### 2-3. 設定ファイルのバックアップ

```bash
sudo cp /var/lib/pgsql/17/data/postgresql.conf /tmp/postgresql.conf.bak
sudo cp /var/lib/pgsql/17/data/pg_hba.conf /tmp/pg_hba.conf.bak
```

## 3. マイナーアップデート (例: 17.9 → 17.10)

pg_upgrade 不要。パッケージ更新と再起動のみ。

```bash
sudo dnf update -y postgresql17 postgresql17-server postgresql17-libs
sudo systemctl restart postgresql-17
psql --version
```

## 4. メジャーアップグレード (例: 17 → 18)

### 4-1. 新バージョンのインストール

```bash
sudo dnf install -y postgresql18-server postgresql18
```

### 4-2. 新バージョンのデータベース初期化

```bash
sudo /usr/pgsql-18/bin/postgresql-18-setup initdb
```

### 4-3. チェックサムの不一致対応

旧クラスタがチェックサム無効で新クラスタが有効の場合、合わせる必要がある。

```bash
# 新クラスタのチェックサムを無効化 (旧クラスタに合わせる場合)
sudo -u postgres /usr/pgsql-18/bin/pg_checksums --disable -D /var/lib/pgsql/18/data
```

### 4-4. pg_hba.conf を一時的に trust に変更

pg_upgrade は内部的にPostgreSQLを起動して接続するため、パスワード認証があると失敗する。

```bash
sudo -u postgres sed -i 's/scram-sha-256/trust/g' /var/lib/pgsql/17/data/pg_hba.conf
sudo -u postgres sed -i 's/md5/trust/g' /var/lib/pgsql/17/data/pg_hba.conf
sudo -u postgres sed -i 's/scram-sha-256/trust/g' /var/lib/pgsql/18/data/pg_hba.conf
sudo -u postgres sed -i 's/md5/trust/g' /var/lib/pgsql/18/data/pg_hba.conf
```

### 4-5. pg_upgrade によるアップグレード

```bash
# 旧バージョン停止
sudo systemctl stop postgresql-17

# /tmp に移動 (postgres ユーザーに書き込み権限が必要)
cd /tmp

# 事前チェック
sudo -u postgres /usr/pgsql-18/bin/pg_upgrade \
  --old-datadir /var/lib/pgsql/17/data \
  --new-datadir /var/lib/pgsql/18/data \
  --old-bindir /usr/pgsql-17/bin \
  --new-bindir /usr/pgsql-18/bin \
  --check

# 本実行
sudo -u postgres /usr/pgsql-18/bin/pg_upgrade \
  --old-datadir /var/lib/pgsql/17/data \
  --new-datadir /var/lib/pgsql/18/data \
  --old-bindir /usr/pgsql-17/bin \
  --new-bindir /usr/pgsql-18/bin
```

### 4-6. 設定の移行と起動

```bash
# バックアップした設定ファイルを新クラスタに復元
sudo cp /tmp/postgresql.conf.bak /var/lib/pgsql/18/data/postgresql.conf
sudo cp /tmp/pg_hba.conf.bak /var/lib/pgsql/18/data/pg_hba.conf

sudo systemctl enable postgresql-18
sudo systemctl start postgresql-18

psql -U admin -h 127.0.0.1 -d postgres -c "SELECT version();"
```

### 4-7. アップグレード後の処理

```bash
# 統計情報の再収集
/usr/pgsql-18/bin/vacuumdb -U admin -h 127.0.0.1 --all --analyze-in-stages --missing-stats-only

# 旧クラスタのデータ削除 (内容確認してから実行)
cat /tmp/delete_old_cluster.sh
sudo bash /tmp/delete_old_cluster.sh

# 旧バージョンの無効化・削除
sudo systemctl disable postgresql-17
sudo dnf remove -y postgresql17-server postgresql17
```

## 5. トラブル時のロールバック

```bash
sudo systemctl stop postgresql-18
sudo systemctl start postgresql-17
# 必要に応じてバックアップからリストア
psql -U admin -h 127.0.0.1 -d postgres < /tmp/pg_backup_all.sql
```

## 6. 後片付け

```bash
rm ~/.pgpass
```

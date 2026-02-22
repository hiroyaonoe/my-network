# ローカルPC用設定ファイル

このディレクトリには、管理用PCのローカル設定に追記する内容を保存しています。

## ファイル一覧

| ファイル | 追記先 | 用途 |
|---------|--------|------|
| `hosts` | `/etc/hosts` | ホスト名解決 |
| `ssh-config` | `~/.ssh/config` | SSH接続設定 |

## 適用方法

### /etc/hosts への追記

```bash
# バックアップ
sudo cp /etc/hosts /etc/hosts.bak

# 追記
sudo tee -a /etc/hosts < etc/hosts

# 確認
cat /etc/hosts
```

### ~/.ssh/config への追記

```bash
# ディレクトリ作成（存在しない場合）
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# バックアップ（既存ファイルがある場合）
cp ~/.ssh/config ~/.ssh/config.bak 2>/dev/null || true

# 追記
cat etc/ssh-config >> ~/.ssh/config

# パーミッション設定
chmod 600 ~/.ssh/config

# 確認
cat ~/.ssh/config
```

## 適用後の使用例

### ホスト名でSSH接続

```bash
# Catalystへ接続
ssh local-router
# または
ssh catalyst

# Proxmoxノードへ接続
ssh mandolin1
ssh mandolin2
ssh mandolin3
```

### ホスト名でping

```bash
ping local-router
ping mandolin1
```

### ホスト名でscp

```bash
# Proxmoxへファイルコピー
scp -r pm/mandolin/mandolin1/etc mandolin1:/

# Catalystから設定取得 (tftp等が必要)
# Catalystはscpをサポートしていないため、show running-configの出力を保存
ssh catalyst "show running-config" > catalyst-backup-$(date +%Y%m%d).conf
```

## 注意事項

- `/etc/hosts` の編集には管理者権限（sudo）が必要
- `~/.ssh/config` のパーミッションは 600 にする必要がある
- 既存の設定がある場合は、重複や競合に注意
- Catalyst接続用の古いSSH方式は、セキュリティ上のリスクがあるため、信頼できるネットワーク内でのみ使用すること

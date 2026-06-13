# ckpool プロキシセットアップ手順 (2026-06-13)

## 概要

旧サーバー (`133.167.82.173` / arkpool2) から新サーバーへ、3つの ckproxy サービスを移行・セットアップした。

| サービス | ポート | 接続先 | コイン |
|---|---|---|---|
| arkpool-proxy-dgb | 3335 | eu.molepool.com:7451 | DGB |
| arkpool-proxy-bc2 | 3336 | bc2.kryptex.network:7041 | BTC |
| arkpool-proxy-quai | 3337 | quai-sha256.kryptex.network:7044 | QUAI |

---

## 1. conf ファイル作成

`~/ckpool/` 以下に3つの conf ファイルを作成。

### ckproxy-dgb.conf (`~/ckpool/ckproxy-dgb.conf`)
```json
{
  "name": "ckproxy-dgb",
  "proxy": [{"url": "eu.molepool.com:7451", "auth": "DK4Cj6kTuGjtqrTnphwPQNiDyqWvDdgN69.arkpool", "pass": "x"}],
  "serverurl": ["0.0.0.0:3335"],
  "logdir": "/home/arkpool/ckpool/logs",
  "loglevel": 6,
  "startdiff": 16384,
  "mindiff": 16384,
  "stale_ms": 2000
}
```

### ckproxy-bc2.conf (`~/ckpool/ckproxy-bc2.conf`)
```json
{
  "name": "ckproxy-bc2",
  "proxy": [{"url": "bc2.kryptex.network:7041", "auth": "bc1qxhr5649audjv4f9lt7yc9up8e75ya6hnxrv85e.arkpool", "pass": "x"}],
  "serverurl": ["0.0.0.0:3336"],
  "logdir": "/home/arkpool/ckpool/logs",
  "loglevel": 6,
  "startdiff": 16384,
  "mindiff": 16384,
  "stale_ms": 2000
}
```

### ckproxy-quai.conf (`~/ckpool/ckproxy-quai.conf`)
```json
{
  "name": "ckproxy-quai",
  "proxy": [{"url": "quai-sha256.kryptex.network:7044", "auth": "0x00611e2cF57B087bAccf7A12498119D49f28FDd6.arkpool", "pass": "x"}],
  "serverurl": ["0.0.0.0:3337"],
  "logdir": "/home/arkpool/ckpool/logs",
  "loglevel": 6,
  "startdiff": 16384,
  "mindiff": 16384,
  "stale_ms": 2000,
  "version_mask": "00000000"
}
```

auth は旧サーバーの conf から取得:
```bash
ssh arkpool@133.167.82.173 'grep auth ~/arkpool2/ckproxy.conf'
ssh arkpool@133.167.82.173 'grep auth ~/arkpool2/ckproxy-bc2.conf'
ssh arkpool@133.167.82.173 'grep auth ~/arkpool2/ckproxy-quai.conf'
```

---

## 2. systemd サービスファイル作成

`/etc/systemd/system/` 以下に3ファイルを作成（sudo 必要）。

共通設定: `User=arkpool`, `Restart=always`, `RestartSec=10`, `RestartForceExitStatus=0`

### arkpool-proxy-dgb.service
```ini
[Unit]
Description=arkpool-proxy-dgb
After=network.target

[Service]
User=arkpool
ExecStart=/usr/local/bin/ckpool -u -c /home/arkpool/ckpool/ckproxy-dgb.conf -n arkpool-proxy-dgb -s /tmp/arkpool-proxy-dgb
Restart=always
RestartSec=10
RestartForceExitStatus=0

[Install]
WantedBy=multi-user.target
```

bc2・quai も同様（`-c`, `-n`, `-s` の引数を各サービス名に合わせて変更）。

```bash
sudo systemctl daemon-reload
sudo systemctl enable arkpool-proxy-dgb arkpool-proxy-bc2 arkpool-proxy-quai
sudo systemctl start  arkpool-proxy-dgb arkpool-proxy-bc2 arkpool-proxy-quai
```

---

## 3. トラブル: `/opt/ckdb/` エラー

### 症状
```
Failed to make directory /opt/ckdb/
```
サービスが起動直後に終了し、10秒ごとに再起動を繰り返す。

### 原因
`/usr/local/bin/ckpool` のインストール済みバイナリが古いビルドで、`socket_dir` のデフォルトが `/opt/ckdb/` にハードコードされていた。ソースツリー (`src/ckpool`) は修正済みで `/tmp/<name>` をデフォルトとしているが、インストール済みバイナリと別物だった。

```bash
md5sum /usr/local/bin/ckpool /home/arkpool/ckpool/src/ckpool
# → ハッシュが異なる
```

### 解決策
`cp` は "テキストファイルがビジー状態" エラーになるため（サービスが繰り返し起動中）、`mv` による atomic 置き換えを使用:

```bash
sudo systemctl stop arkpool-proxy-dgb arkpool-proxy-bc2 arkpool-proxy-quai
sudo cp /home/arkpool/ckpool/src/ckpool /usr/local/bin/ckpool.new
sudo mv /usr/local/bin/ckpool.new /usr/local/bin/ckpool
sudo systemctl start arkpool-proxy-dgb arkpool-proxy-bc2 arkpool-proxy-quai
```

---

## 4. ファイアウォール設定

```bash
sudo ufw allow 3335/tcp
sudo ufw allow 3336/tcp
sudo ufw allow 3337/tcp
sudo ufw status
```

---

## 5. ログ確認

```bash
# サービス状態
sudo systemctl status arkpool-proxy-dgb arkpool-proxy-bc2 arkpool-proxy-quai

# journald ログ
journalctl -u arkpool-proxy-dgb -n 30 --no-pager

# ファイルログ（認証・ユーザー・ワーカー）
tail -f ~/ckpool/logs/arkpool-proxy-dgb.log | grep -E "authoris|user|worker"
tail -f ~/ckpool/logs/arkpool-proxy-bc2.log | grep -E "authoris|user|worker"
tail -f ~/ckpool/logs/arkpool-proxy-quai.log | grep -E "authoris|user|worker"
```

---

## 動作確認済み状態

- 3サービスとも上流プールへの接続・認証成功
- ワーカー `DRe63e4QYorHFAjSurFK296siespy1CQa2.drjiro` が DGB プロキシ経由で稼働中
- ポート 3335/3336/3337 が ufw で開放済み

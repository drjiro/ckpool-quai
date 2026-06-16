# QUAI SHA256 Solo Mining — 修正記録と現在の仕様

最終更新: 2026-06-16

---

## 現在の構成

| 項目 | 内容 |
|---|---|
| モード | btcsolo（`-B` フラグ） |
| ローカルノード | `localhost:9200`（quai_getBlockTemplate / quai_submitShaBlock） |
| マイナー接続ポート | `0.0.0.0:3337` |
| サービス | `arkpool-proxy-quai.service` |
| バイナリ | `/usr/local/bin/ckpool` |
| 設定ファイル | `/home/arkpool/ckpool/quai-sha.conf` |

---

## 経緯

当初は `ckpool -u`（proxy モード）で Kryptex（`quai-sha256.kryptex.network:7044`）に上流転送する構成を試みた。
マイナーは接続・認証でき、ローカルでのハッシュ計算も正常だったが、Kryptexが全シェアを
`Invalid share` で拒否した。

原因として mining.submit の 6番目パラメータ（version field）の転送方式の問題
（BIP320のdeltaビットをそのまま送るか `base_version | delta` として送るかの違い）が疑われたが、
arkpool2との統合が難しいため、プロキシ方式を断念。

**結論：** ckpool は QUAI ローカルノード直接接続の solo pool 専用とした。

---

## コード修正一覧（src/stratifier.c）

### Fix 1: btcsolo モードで GBT target diff がクライアント diff として使われていた

**問題：** GBT ネットワーク target 由来の diff（約 76M〜90M）が `startdiff` を上書きし、
ASIC マイナーが有効なシェアを出せなくなっていた。

**修正（`~L3836`）：**

```c
/* Use startdiff for vardiff; GBT target diff (~76M) is too high for ASIC shares */
if (0) { /* disabled GBT-target-as-diff */
    // ← ネットワーク target をdiffに使う旧コードを無効化
} else {
    client->diff = client->old_diff = ckp->startdiff;
    /* client->target stays empty; share validation falls back to diff comparison */
}
```

**現状：** `if (0)` ブロックで無効化済み。`ckp->startdiff` がクライアントの初期diff。✓

---

### Fix 2: GBT 更新時に client->target がネットワーク target で上書きされていた

**問題：** ブロック更新のたびに `client->target` にネットワーク target（約 90M diff 相当）が
セットされ、ASIC の share が全て "high diff" として棄却されていた。

**修正：** `stratum_broadcast_updates()` 内の target 上書きコードを `if (0)` で無効化。

**現状：** `client->target` は空のまま。シェア採否は diff 比較のみで判定。✓

---

### Fix 3: シェア採否判定を diff 比較のみに変更

**問題：** `submission_diff()` 内の `meets_target` がネットワーク target との比較（`fulltest()`）を
使っていたため、ASIC シェア（diff 100〜2000 程度）は全て棄却されていた。

**修正（`~L6876`）：**

```c
/* Disabled: always use diff comparison; network target (~90M) must not be used */
(void)target; (void)target_swap; (void)target_to_use;
meets_target = (sdiff >= diff);
```

**現状：** diff 比較のみ。ネットワーク target は share 採否に使われない。✓

---

### Fix 4: ブロック提出は GBT target との厳密な比較（btcsolo モード）

**場所：** `check_solve()` 関数（`~L6250`）

ブロック提出時（`check_solve`）は btcsolo モードで `fulltest(hash, target_swap)` を使い、
ネットワーク target を下回るハッシュが出た場合のみ `quai_submitShaBlock` を呼ぶ。
シェア採否（Fix 3）とブロック提出判定は別ロジック。

```c
if (ckp->btcsolo) {
    hex2bin(target, wb->target, 32);
    bswap_256(target_swap, target);
    meets_target = fulltest(hash, target_swap);
    if (!meets_target) return; // ← ブロック候補でなければスルー
    LOGNOTICE("Share diff %.6f meets target %s, submitting to node", diff, wb->target);
}
```

**現状：** 正常動作。ネットワーク target を下回るハッシュが出たときのみ提出。✓

---

### Fix 5: OP_PUSH42 ガード（proxy モード向け、btcsolo には影響なし）

**問題（proxy モード時）：** Kryptex の coinb1 が `0x2a`（OP_PUSH42）で終わっていたため、
coinb2 の先頭 30 バイトゼロチェックがプロキシモードでも実行され、全シェアが棄却されていた。

**修正（`~L6427`、未コミット）：**

```c
if (!client->ckp->proxy && wb->coinb1bin[wb->coinb1len - 1] == 0x2a) {
```

**現状（btcsolo）：** `ckp->proxy = false` なので btcsolo では常にチェック実行。
go-quai の coinb2 は先頭 30 バイトがゼロなのでチェックは正常通過。✓

---

## coinbase 構造（go-quai GBT 提供）

go-quai の `quai_getBlockTemplate` が coinb1/coinb2 を直接提供する。
ckpool はこれを verbatim で使用し、pool 側での coinbase 組み立ては行わない。

```
coinb1: 92 bytes (末尾 0x2a = OP_PUSH42)
coinb2: 78 bytes (先頭 30 バイト = 0x00 × 30 ← extraData プレースホルダ)
extranonce1Length: 4 bytes
extranonce2Length: 8 bytes
合計 coinbase: 182 bytes
```

ペイアウトアドレスは go-quai ノード側の設定に依存（coinb2 に P2PKH scriptPubKey として埋め込み済み）。
ckpool 設定の `btcaddress` は btcsolo モードでは使用されない。

---

## 現在の設定（quai-sha.conf）

```json
{
  "btcd": [{"url": "localhost:9200", "auth": "", "pass": "", "notify": false}],
  "btcaddress": "0x00611e2cF57B087bAccf7A12498119D49f28FDd6",
  "logdir": "/home/arkpool/ckpool/logs",
  "logshares": true,
  "serverurl": ["0.0.0.0:3337"],
  "scrypt_algo": false,
  "mindiff": 32,
  "startdiff": 128,
  "blockpoll": 500,
  "update_interval": 30
}
```

| パラメータ | 値 | 理由 |
|---|---|---|
| `startdiff` | 128 | Avalon Nano 3S（2.93 GH/s）で約1〜2分/シェア |
| `mindiff` | 32 | vardiff の下限 |
| `blockpoll` | 500 | 500ms ごとに GBT を取得（ブロック変化への追従） |
| `scrypt_algo` | false | SHA256 モード |

---

## サービス（/etc/systemd/system/arkpool-proxy-quai.service）

```ini
[Unit]
Description=ckpool QUAI SHA256 Solo Pool (local node)
After=network.target

[Service]
Type=simple
User=arkpool
ExecStart=/usr/local/bin/ckpool \
  -B \
  -c /home/arkpool/ckpool/quai-sha.conf \
  -n arkpool-quai-pool \
  -s /tmp/arkpool-quai-pool
Restart=always
RestartSec=10
RestartForceExitStatus=0

[Install]
WantedBy=multi-user.target
```

---

## 稼働確認済み（2026-06-16）

```
Solo mode: using configured share difficulty - mindiff=32 startdiff=128
Connected to bitcoind: localhost:9200
quai_getBlockTemplate 正常レスポンス (height 955538, quaidifficulty ~1.4兆)
PoW verified: share difficulty = 1900.100670
Using Quai-supplied coinb1/coinb2 (lens 92/78)
```

ネットワーク難易度は約 1.44 兆（ASIC 2.93 GH/s でのブロック期待採掘時間は数日〜数ヶ月オーダー）。

---

## 既知の軽微な警告

- `Passed empty pointer to realloc_strcat`: btcd の auth/pass が空文字列であることによる無害な警告
- `connector.c` に `LOGWARNING` によるraw message ログが未コミットで入っており、ログが冗長になる場合がある

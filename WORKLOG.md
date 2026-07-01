# QUAI SHA256 Solo Mining - Work Log

## 2026-07-01: 非カストディアル（ノンカストディ）マルチワーカーマイニング実装

### 背景

- ckpool を QUAI SHA-256 ソロマイニングプール（port 3334）として稼働
- 従来は `quai-sha.conf` の `btcaddress` で設定した1アドレスのみがcoinbase受取先
- 複数ワーカーが接続しても全報酬がプールオペレーターのアドレスに集約される問題

### 発見した仕様

- `go-quai` の `quai_getBlockTemplate` RPC は `{"coinbase": "0x..."}` パラメータで任意アドレスの `coinb1`（SealHash入り）を返す
- `coinb1` がアドレスによって変わる（SealHashにPrimaryCoinbaseが埋め込まれる）
- `coinb2`（P2PKHスクリプト）は変わらない ← go-quaiの仕様制限
- ワーカーがStratum接続時のユーザー名を `0xWALLET_ADDRESS.workerName` 形式にすることで自動識別可能

### 実装内容

**`src/bitcoin.c` / `src/bitcoin.h`**
- `gen_gbtbase()` に `const char *coinbase_addr` 引数を追加
- アドレス指定時は `{"rules": ["sha"], "coinbase": "0x..."}` の形式でGBTリクエストを送信

**`src/generator.c` / `src/generator.h`**
- `generator_getbase_for_address(ckp, addr)` 関数を追加
- 既存の `generator_getbase()` は `coinbase_addr=NULL` として動作を維持

**`src/stratifier.c`**
- `struct userwb` に `coinb1bin`/`coinb1`/`coinb1len` フィールドを追加
- `struct user_instance` に `quai_address[128]`/`quai_coinb1`/`quai_coinb1bin`/`quai_coinb1len` を追加
- `extract_quai_address()`: ユーザー名から `0x...` アドレスを抽出するヘルパー
- `refresh_all_user_coinb1s()`: 接続中の全QUAIユーザーのアドレスでGBTを呼び出し、per-user coinb1を取得・キャッシュ
- `add_base()`: 新ブロックごとに `refresh_all_user_coinb1s()` → `generate_userwbs()` の順で実行
- `__generate_userwb()`: per-user coinb1をuserwbにコピー
- `__user_coinb1()`: `__user_coinb2()` と同パターンでper-user coinb1を返すインライン関数
- `__user_notify()`: `mining.notify` でper-user coinb1を送信
- `submission_diff()`: シェア検証・ブロック再構築時にper-user coinb1を使用

### マイナー設定方法

Stratum接続時のユーザー名を以下の形式にする：

```
0xYOUR_WALLET_ADDRESS.workerName
```

例：
```
0x002E2088873c7E5E7d02F479E4Af3C61092107Ac.rig01
```

これにより各マイナーの報酬が直接自身のウォレットに送付される。

### ブロックハッシュログ追加

- `block_solve()` に `hash` フィールドを追加
- ログ形式: `Solved and confirmed block HEIGHT by WORKERNAME hash BLOCKHASH`
- `importBlocks.ts` がハッシュを読み取り `quai_getBlockByHash` で報酬額も取得可能に

---

## 2026-06-30: 発見ブロックDB取り込み・WebUI修正

### 過去34ブロックのDB取り込み（`importBlocks.ts`）

- `ckpool-quai.log` の `Solved and confirmed block` エントリをgrepで抽出（OOM回避）
- `ckstats.Block` テーブルに34ブロックをインポート（2026-06-17〜07-01）
- cronで5分おきに実行（新規ブロックの自動登録・ブロックハッシュと報酬の遡及取得）

### WebUI（arkstats2）修正

- `PoolMinersSection.tsx`: `getBlocks()` でDBから発見ブロック一覧表示
- `reward=0` の場合は「—」表示（報酬取得不可ブロックの適切な表示）
- `ckdb-client.ts`: ckdbのBlockテーブルも読める実装を追加

### PostgreSQL BigIntオーバーフロー修正

- `poolHashrateFromLog()`: 最低3シェア必要、PostgreSQL bigint上限でキャップ
- スケールファクターを最大1000倍に制限

---

## 2026-06 以前: 基盤構築

### go-quai systemdサービス化

- `/etc/systemd/system/go-quai.service` 作成
- `--node.quai-coinbases 0x00611e2cF57B087bAccf7A12498119D49f28FDd6` で起動
- port 9200でRPC待ち受け

### QUAI SHA256ソロマイニング初期修正（src/stratifier.c）

1. btcsoloモードでGBT targetがdiffとして使われていた → `ckp->startdiff` を使用
2. GBT更新のたびにclient->targetが上書きされていた → 無効化
3. share検証でnetwork targetと比較していた → sdiff >= diff 比較のみに変更

### DGB Proxy BIP320修正

- `version_bits` フィールドのパース・適用処理追加

### 設定（quai-sha.conf）

- `startdiff`: 128（SHA256 ASIC向け初期難易度）
- `blockpoll`: 500ms
- `update_interval`: 30s
- `scrypt_algo`: false（SHA256）
- `port`: 3334

### 起動

```sh
sudo systemctl restart ckpool.service
```

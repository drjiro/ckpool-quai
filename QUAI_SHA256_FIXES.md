# QUAI SHA256 Solo Mining Fixes

## 問題
jdowning100/ckpool を QUAI ノード（localhost:9200）に直接接続して
SHA256ソロマイニングを行う際、share が Accepted にならなかった。

## 原因と修正箇所（src/stratifier.c）

### 1. btcsoloモードでGBT targetがdiffとして使われていた（約3835行）
- GBT network target diff（~90M）が startdiff を上書き
- `if (0)` で無効化し、`ckp->startdiff` を使用するよう変更

### 2. GBT更新のたびにclient->targetが上書きされていた（約7167行）
- `if (0)` で無効化

### 3. client->targetの初期値にnetwork targetが設定されていた（約3846行）
- 初期化コードから `strncpy(client->target, ...)` を削除

### 4. share検証でnetwork targetと比較していた（約6872行）
- `fulltest()` によるtarget比較を廃止
- `sdiff >= diff` のdiff比較のみに変更

## 設定（quai-sha.conf）
- `startdiff`: 32768（Nano 3S向け）
- `blockpoll`: 10000（10秒、coinbase更新頻度を抑える）
- `update_interval`: 30
- `scrypt_algo`: false（SHA256）

## 起動

```sh
./src/ckpool -B -c quai-sha.conf
```
